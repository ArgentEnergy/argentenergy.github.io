---
layout: post
title:  "Privilege Escalation to Moderator using Email Aliases"
date:   2022-08-04
---

During a recent assessment, I identified a vulnerability in a Firebase application that enabled me to create a secondary password for another user, granting access to their privileged account.

## Application Background
The application allows users to watch event sessions, engage in discussions, and interact with others. Moderators have the ability to ban users or delete messages.

