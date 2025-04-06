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
