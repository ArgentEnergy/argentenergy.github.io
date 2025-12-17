---
title:  "Azure SSRF with Workflow Designer Feature"
layout: post
date:   2025-08-20
---

During a recent security assessment, I discovered a Server-Side Request Forgery (SSRF) vulnerability that granted access to the Azure Instance Metadata Service (IMDS). Using this access, I successfully retrieved an Azure access token with Contributor role permissions.

## Discovering the Vulnerability
The application included a workflow designer for building web forms, which featured a template system. Within this system, there was an "External API" option that allowed users to configure API calls by specifying a URL, HTTP method, request headers, and response mappings to populate form fields with data from the API response.

<figure>
  <img src="/assets/images/2025/ssrf-1.png">
  <figcaption>Figure 1 – External API pop up</figcaption>
</figure>

To use the External API functionality, stages must be created within the workflow designer, and a trigger must be configured on a stage component to invoke the API. During testing, I set the trigger to activate when the user entered the word "Stratum" into a text field and clicked the submit button while previewing the workflow.

<figure>
  <img src="/assets/images/2025/ssrf-2.png">
  <figcaption>Figure 2 – Mapping the API trigger to a text field in the workflow web form</figcaption>
</figure>

<figure>
  <img src="/assets/images/2025/ssrf-3.png">
  <figcaption>Figure 3 – Configured the API trigger to run when the text field contains Stratum</figcaption>
</figure>

After clicking the submit button in the workflow web form, the user is redirected to a results page displaying the request and response details. However, if the response body is large, part of it may be truncated. Knowing the response would be rendered, I configured the External API to call **http://169.254.169.254/metadata/instance?api-version=2020-06-01** with the header **Metadata: true**. Upon previewing the workflow, entering "Stratum" in the text field, and submitting the form, I successfully received and viewed the Azure instance metadata in the response.

<figure>
  <img src="/assets/images/2025/ssrf-4.png">
  <figcaption>Figure 4 – Previewed workflow with the word Stratum in the text field to trigger the External API</figcaption>
</figure>

<figure>
  <img src="/assets/images/2025/ssrf-5.png">
  <figcaption>Figure 5 – Retrieved the Azure instance metadata from the workflow form submission</figcaption>
</figure>

## Path to Exploitation
While attempting to retrieve the Azure access token, I encountered errors indicating a client ID was required. Using the client ID obtained from the instance metadata, I was able to retrieve a partial access token—but it wasn’t sufficient to demonstrate meaningful impact.

<figure>
  <img src="/assets/images/2025/ssrf-6.png">
  <figcaption>Figure 6 – Retrieved a partial Azure access token</figcaption>
</figure>

To address this, I returned to the workflow template and updated the External API configuration to use a JPath expression on the response body. This allowed the response body to be parsed, extracting the full access token and placing it into a text field within the web form.

To view the token, the user must preview the workflow and follow the steps to trigger the API. After that, they exit the workflow, return to the main page, locate the generated workflow request, and navigate to its request page. There, the full Azure access token appears in the designated text field.

<figure>
  <img src="/assets/images/2025/ssrf-7.png">
  <figcaption>Figure 7 – Full Azure access token in the text field</figcaption>
</figure>

After retrieving the full access token, I wanted to determine what permissions were assigned to it. This was my first successful Azure SSRF, as access is typically blocked due to the required Metadata request header.

<figure>
  <img src="/assets/images/2025/ssrf-8.png">
  <figcaption>Figure 8 – Listed role assignments for the token</figcaption>
</figure>

<figure>
  <img src="/assets/images/2025/ssrf-9.png">
  <figcaption>Figure 9 – Token has the Azure Contributor role</figcaption>
</figure>

After submitting the report, the client informed us that the issue had been resolved. The application was configured to allow self-registration, meaning anyone could create an account and access the vulnerable functionality. This made the SSRF a critical finding because of the self-registration.
