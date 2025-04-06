---
layout: post
title:  "Blind Remote Code Execution through YAML Deserialization"
date:   2021-06-09 12:00:00
categories: node
---

While performing an application security assessment on a Ruby on Rails project, I discovered upload functionality that allowed users to upload text, CSV, and YAML files. The latter option interested me because reading online suggested YAML deserialization could be a potential vector.
