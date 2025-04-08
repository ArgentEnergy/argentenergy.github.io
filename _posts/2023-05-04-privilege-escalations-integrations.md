---
layout: post
title:  "Privilege Escalations through Integrations"
date:   2023-05-04
---

This assessment was interesting as multiple issues in the integration of Okta, Amazon Cognito, and Tableau resulted in privilege escalation vulnerabilities, allowing both unauthenticated and authenticated users to gain administrative access.

## Application Background
The application allows financial institutions to create dashboards using their financial data for analysis.

It acts as a wrapper for Tableau, with custom code integrating Amazon Cognito and Okta for authentication, Tableau for dashboard functionality, and a custom web page to list and share accessible dashboards.

## Amazon Cognito Misconfigurations
Amazon Cognito allows users to self-register, confirm their account, and retrieve tokens (access, ID, and refresh). In this application, however, self-registration was not intended, as the system was designed for administrators to provision accounts.

An unauthenticated user can exploit a misconfiguration by retrieving the Cognito client ID from the /static/js/main.js file. With Cognito self-registration enabled, users can bypass the Okta integration, create accounts, and log in. Correct profile and role values are necessary for successful access to the application, and these values were exposed in a public development environment.

Unauthenticated users can enumerate email addresses by accessing the public development environment, and authenticated users can easily format the necessary values. By self-registering, users can update their attributes and forge JWTs to impersonate other users.

<figure>
  <img src="/assets/img/2023/signup-account-takeover-1.png">
  <figcaption>Figure 1 – Confirmation Cognito Self-Registration is enabled for application client</figcaption>
</figure>  

<figure>
  <img src="/assets/img/2023/signup-account-takeover-2.png">
  <figcaption>Figure 2 – Confirmed the account using the code sent in the email</figcaption>
</figure>  

<figure>
  <img src="/assets/img/2023/signup-account-takeover-3.png">
  <figcaption>Figure 3 – Logged into application using Cognito bypassing the Okta integration</figcaption>
</figure>  

<figure>
  <img src="/assets/img/2023/signup-account-takeover-4.png">
  <figcaption>Figure 4 – Set user attributes to another user's email, role, and profile</figcaption>
</figure>  

<figure>
  <img src="/assets/img/2023/signup-account-takeover-5.png">
  <figcaption>Figure 5 – Logged in using another user's email and attacker's password</figcaption>
</figure>  

<figure>
  <img src="/assets/img/2023/signup-account-takeover-6.png">
  <figcaption>Figure 6 – Confirmation registered account can access application</figcaption>
</figure>  

## Tableau Trusted Auth Privilege Escalation
The application integrates Tableau using Tableau's trusted authentication. Custom code in the Amazon API Gateway only checked for a valid JWT but didn't verify the user's identity. The email address in the URL was used to confirm the identity and retrieve a valid Tableau ticket.

Unauthenticated users can self-register with Cognito and then use another user's email address (such as a Tableau server admin) to retrieve a valid Tableau ticket, leading to privilege escalation in Tableau.

<figure>
  <img src="/assets/img/2023/tableau-priv-escalation-1.png">
  <figcaption>Figure 7 – Retrieved another user's Tableau ticket</figcaption>
</figure> 

<figure>
  <img src="/assets/img/2023/tableau-priv-escalation-2.png">
  <figcaption>Figure 8 – Exchanged Tableau ticket for Tableau session</figcaption>
</figure> 

<figure>
  <img src="/assets/img/2023/tableau-priv-escalation-3.png">
  <figcaption>Figure 9 – Confirmation escalated Tableau privileges to Site Administrator Explorer</figcaption>
</figure> 

<figure>
  <img src="/assets/img/2023/tableau-priv-escalation-4.png">
  <figcaption>Figure 10 – List all site users to find Tableau Server Admin to escalate privileges again</figcaption>
</figure> 

## Privilege Escalation to Server Admin in Tableau Development Environment
