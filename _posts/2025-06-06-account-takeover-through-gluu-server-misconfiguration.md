---
title:  "Account Takeover through Gluu Server Misconfiguration"
layout: post
date:   2025-06-06
---

In a previous assessment, I identified that the application used a Gluu server for identity and access management. This was confirmed by the discovery of the **/.well-known/scim-configuration** endpoint. Users managed within the Gluu server had access to multiple applications, including the one assessed.

This was my first encounter with Gluu, and I spent some time reviewing the documentation. Gluu offers an API for querying users via a filter query parameter at **/identity/restv1/scim/v2/Users**, as outlined in the [Gluu Server API Guide](https://gluu.org/docs/gluu-server/4.0/api-guide/scim-api/).

Due to a misconfiguration within the Gluu server, I was able to query user data without an active session or token.

## Path to Account Takeover
While reviewing the [Gluu documentation](https://gluu.org/docs/gluu-server/4.0/user-management/scim2/) on building filters, I referenced [RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644#section-3.4.2) for the supported operators.

I sent a request to **/identity/restv1/scim/v2/Users?filter=userName+eq+%22admin%22&count=100&startIndex=1** and, to my surprise, no authentication was needed. The response returned an admin account.

<figure>
  <img src="/assets/images/2025/preauth-list-admin-user.png">
  <figcaption>Figure 1 – Successfully queried an admin account without authentication</figcaption>
</figure>

When viewing the JSON response in the browser, I clicked the group URL (**/identity/restv1/scim/v2/Groups/60A...**), which exposed all users in the admin group. I then retrieved an admin user's ID and sent a PUT request to update the user.

<figure>
  <img src="/assets/images/2025/preauth-list-group-with-users.png">
  <figcaption>Figure 2 – Viewed group containing list of users</figcaption>
</figure>

<figure>
  <img src="/assets/images/2025/preauth-update-user.png">
  <figcaption>Figure 3 – Successfully updated the user's password</figcaption>
</figure>

To my surprise, the PUT request successfully updated the user’s password. I then logged into the admin account with the new password. The risk was critical, as no authentication was required for the API, and this vulnerability allowed anyone to view and potentially take over user accounts in a production environment, impacting access to various customer applications.

## Remediation
The recommended remediation was to configure Gluu to require authentication for accessing the SCIM APIs. These endpoints probably should only be accessible to highly privileged users. Additionally, it may be advisable to prevent direct password changes on user accounts, enforcing updates through the dedicated password reset functionality—if Gluu provides an option to disable direct password changes.
