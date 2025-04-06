---
layout: post
title:  "Blind Remote Code Execution through YAML Deserialization"
date:   2021-06-09 12:00:00
categories: node
---

During an application security assessment of a Ruby on Rails project, I identified upload functionality that allowed users to submit text, CSV, and YAML files. The YAML option stood out due to its potential as a deserialization vulnerability.  

After several uploads, I found that the process validated file contents before uploading them to Azure blob storage. YAML files starting with ```---!ruby/object:BadValue``` triggered a fatal status, while other invalid YAML files returned an error status. This fatal status was the only indicator that the upload process might be vulnerable.

<figure>
  <img src="/assets/2021/blind-rce-status-indicator.png">
  <figcaption>Figure 1 – Fatal Status on poc2.yml</figcaption>
</figure>
