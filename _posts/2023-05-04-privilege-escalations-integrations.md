---
title:  "Privilege Escalations through Integrations"
layout: post
date:   2023-05-04
---

This assessment was interesting as multiple issues in the integration of Okta, Amazon Cognito, and Tableau resulted in privilege escalation vulnerabilities, allowing both unauthenticated and authenticated users to gain administrative access.

## Application Background
The application allows financial institutions to create dashboards using their financial data for analysis.

It acts as a wrapper for Tableau, with custom code integrating Amazon Cognito and Okta for authentication, Tableau for dashboard functionality, and a custom web page to list and share accessible dashboards.

## Amazon Cognito Misconfigurations
Amazon Cognito allows users to self-register, confirm their account, and retrieve tokens (access, ID, and refresh). In this application, however, self-registration was not intended, as the system was designed for administrators to provision accounts.

An unauthenticated user can exploit a misconfiguration by retrieving the Cognito client ID from the **/static/js/main.js** file. With Cognito self-registration enabled, users can bypass the Okta integration, create accounts, and log in. Correct profile and role values are necessary for successful access to the application, and these values were exposed in a public development environment.

Unauthenticated users can enumerate email addresses by accessing the public development environment, and authenticated users can easily format the necessary values. By self-registering, users can update their attributes and forge JWTs to impersonate other users.

<figure>
  <img src="/assets/images/2023/signup-account-takeover-1.png">
  <figcaption>Figure 1 – Confirmation Cognito Self-Registration is enabled for application client</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2023/signup-account-takeover-2.png">
  <figcaption>Figure 2 – Confirmed the account using the code sent in the email</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2023/signup-account-takeover-3.png">
  <figcaption>Figure 3 – Logged into application using Cognito bypassing the Okta integration</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2023/signup-account-takeover-4.png">
  <figcaption>Figure 4 – Set user attributes to another user's email, role, and profile</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2023/signup-account-takeover-5.png">
  <figcaption>Figure 5 – Logged in using another user's email and attacker's password</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2023/signup-account-takeover-6.png">
  <figcaption>Figure 6 – Confirmation registered account can access application</figcaption>
</figure>  

## Tableau Trusted Auth Privilege Escalation
The application integrates Tableau using Tableau's trusted authentication. Custom code in the Amazon API Gateway only checked for a valid JWT but didn't verify the user's identity. The email address in the URL was used to confirm the identity and retrieve a valid Tableau ticket.

Unauthenticated users can self-register with Cognito and then use another user's email address (such as a Tableau server admin) to retrieve a valid Tableau ticket, leading to privilege escalation in Tableau.

<figure>
  <img src="/assets/images/2023/tableau-priv-escalation-1.png">
  <figcaption>Figure 7 – Retrieved another user's Tableau ticket</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/tableau-priv-escalation-2.png">
  <figcaption>Figure 8 – Exchanged Tableau ticket for Tableau session</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/tableau-priv-escalation-3.png">
  <figcaption>Figure 9 – Confirmation escalated Tableau privileges to Site Administrator Explorer</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/tableau-priv-escalation-4.png">
  <figcaption>Figure 10 – List all site users to find Tableau Server Admin to escalate privileges again</figcaption>
</figure> 

## Privilege Escalation to Server Admin in Tableau Development Environment
The **/static/js/main.js** file exposed Cognito client IDs and Tableau URLs for all environments (dev, QA, UAT, and production). The dev environment did not require a JWT to retrieve a Tableau ticket. An unauthenticated user could use the default Tableau admin username to obtain the ticket and gain Tableau server admin privileges. Additionally, the dev environment was publicly accessible.

As noted earlier, an attacker could retrieve profile and role values to escalate privileges in other environments.

<figure>
  <img src="/assets/images/2023/dev-env-access-1.png">
  <figcaption>Figure 11 – Dev environment details in JS file</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/dev-env-access-2.png">
  <figcaption>Figure 12 – Retrieved default Tableau admin Tableau ticket</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/dev-env-access-3.png">
  <figcaption>Figure 13 – Exchanged ticket for Tableau session</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/dev-env-access-4.png">
  <figcaption>Figure 14 – Confirmation an unauthenticated user can become a Tableau server admin</figcaption>
</figure> 

## Privilege Escalation to Server Admin in Tableau using Okta
While investigating the Okta integration, I attempted to access the Okta console at https://idp-redacted.com/login/default to identify any misconfigurations. Not only was I able to access the console, but I also discovered custom fields in Okta.

Through the Okta APIs, I found a custom field called tableauusername. I updated my user's tableauusername to "admin" to attempt to gain access as the default Tableau admin. After logging out of Okta and logging into the application, I had full access to all Tableau dashboards in the system.

<figure>
  <img src="/assets/images/2023/okta-tableau-priv-escal-1.png">
  <figcaption>Figure 15 – Access to Okta console</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/okta-tableau-priv-escal-2.png">
  <figcaption>Figure 16 – Found custom fields under profile in the Okta API</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/okta-tableau-priv-escal-3.png">
  <figcaption>Figure 17 – Updated the custom tableauusername to admin</figcaption>
</figure> 

<figure>
  <img src="/assets/images/2023/okta-tableau-priv-escal-4.png">
  <figcaption>Figure 18 – Got Tableau server admin viewing all dashboards</figcaption>
</figure> 

## Remediation
For Amazon Cognito, it is recommended to disable self-registration and update user attributes to prevent authenticated users from forging JWTs as another user.

In the custom code for the Amazon API Gateway, the JWT should be used to confirm the user's identity for Tableau trusted authentication, instead of relying on the email query parameter. This fix depends on disabling Cognito's update user attributes call to prevent JWT forging.

For the development environment, it is advised to disable public access at the network level, remove unnecessary environment details from the JavaScript file, and use a JWT for Tableau trusted authentication, as in other environments.

Regarding the Okta integration, the Okta console should not be publicly accessible, and validation should be implemented to prevent users from updating specific custom fields.
