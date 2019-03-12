---
title: "Grant AWS Admin Permissions on a Specific Resource"
layout: single
excerpt: "We should always grant access based on a principle of least privilege. In AWS (Amazon Web Services) you can grant something:* on a very specific resource, that should be admin permissions right? Well in some cases it doesn't... "
sitemap: false
permalink: /granting-aws-admin-permissions-specific-resource
---

# TL;DR
In some cases, to grant admin access for a specific AWS resource you may also need to grant read only access to all resources.

# Why does this even matter
I am a DevOps Engineer, and in today's world means that I build and maintain cloud infrastructure. In most cases that cloud infrastucture resides in AWS (Amazon Web Services) and is maintained via some "Infrastructure as Code" tool like CloudFormation or Terraform. A lot of maintaining any infrastucture is managing permissions appropriately and a good rule of thumb is to always grant access following the ["Principle of Least Privilege"][least_priv]
# How AWS does permissions
# Why you want to grant permissions per resource
# What I do whenever I need to grant access to a specific resource in AWS
# Takeaway

[least_priv]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
