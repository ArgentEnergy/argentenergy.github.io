---
title:  "Apache Lucene Injection on Auth0 Integration"
layout: post
date:   2023-12-18
---

Apache Lucene Injection was identified in a recent assessment, and I found that it isn't widely discussed online. This may be because Lucene Injection is more limited than SQL Injection. Unlike SQL Injection, Lucene Injection can only query the data (index) that the application is designed to query. However, it can still be exploited if the index (similar to an SQL table) contains sensitive data and users are only meant to access a specific subset or certain fields.

## Application Background
The application is a multi-tenant system that stores user information for all tenants in Auth0. [Auth0](https://auth0.com/docs/manage-users/user-search) uses Lucene to query user data, and the application web server calls Auth0 to retrieve users assigned to the current tenant for sharing cybersecurity incident cases. This API is intended to query only users within the assigned tenant, but improper handling of user input allowed for Lucene Injection.

## Exploiting Lucene Injection
The application exposed error information that revealed the Lucene query, making it easier to exploit. The screenshot below shows the query, which is intended to return users for a single tenant.

<figure>
  <img src="/assets/images/2023/apache-lucene-auth0-query-injection-1.png">
  <figcaption>Figure 1 – Lucene query disclosed</figcaption>
</figure>  

The following screenshots show how the Lucene query can be modified to return users from other tenants, as well as how a boolean-based condition can be used to discover additional fields in the Auth0 user index. If a field doesn't exist, an empty JSON array is returned.

<figure>
  <img src="/assets/images/2023/apache-lucene-auth0-query-injection-2.png">
  <figcaption>Figure 2 – Lucene Injection to query all tenant users</figcaption>
</figure>  

<figure>
  <img src="/assets/images/2023/column-exists.png">
  <figcaption>Figure 3 – Lucene Injection to discover other fields on the user index</figcaption>
</figure>  

The following payloads were used to query all tenant users and access other fields in Auth0.

```
// Break out of query and query all users using the name field
1)+OR+(name:*)+OR+(name:*

// Perform a boolean-based condition checking if the user_metadata.country field exists
1)+OR+(name:*)+AND+(_exists_:user_metadata.country

// Discover values searching one character at a time
1)+OR+(name:*)+AND+(user_metadata.country:C*
1)+OR+(name:*)+AND+(user_metadata.country:Ca*
...
```

The last two payloads could be used to query custom fields under app_metadata or user_metadata, as well as other [Auth0 fields](https://auth0.com/docs/manage-users/user-accounts/user-profiles/user-profile-structure) such as phone numbers and IP addresses.

## Remediation
Online sources mention that Lucene supports prepared statements, similar to SQL. Since this was an integration with Auth0, it's likely that the client encoded special characters before passing user input to Auth0 to prevent the injection.
