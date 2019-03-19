---
title: "Grant AWS Permissions on a Specific Resource"
layout: single
excerpt: "Always grant access based on a principle of least privilege. In AWS (Amazon Web Services) you can grant something:* on a very specific resource, that should be admin permissions right? Well in some cases it is not ... "
sitemap: false
permalink: /grant-aws-permissions-specific-resource
---


# TL;DR
In some cases, to grant access for a specific AWS (Amazon Web Services) resource you may also need to grant read only access to more resources. ie you may need `<service_name>:list*`, `<service_name>:get*` or `<service_name>:describe*` on the resource `*` in addition to resource specific permissions on your specific resource.

# Table of Contents
<!-- MarkdownTOC autolink="true" -->

- [Overview](#overview)
    - [Most Recent Example of permissions misconfiguration \(es:* on specific resource\)](#most-recent-example-of-permissions-misconfiguration-es-on-specific-resource)
- [My key takeaways on AWS IAM permissions](#my-key-takeaways-on-aws-iam-permissions)
    - [1. I prefer IAM policies over resource policies](#1-i-prefer-iam-policies-over-resource-policies)
    - [2. Each actor in your infrastructure should have its own IAM role.](#2-each-actor-in-your-infrastructure-should-have-its-own-iam-role)
    - [3. Write tight IAM policies](#3-write-tight-iam-policies)
- [How to do Least Privilege in AWS IAM policies](#how-to-do-least-privilege-in-aws-iam-policies)
    - [What can go wrong when limiting permissions to a specific resource](#what-can-go-wrong-when-limiting-permissions-to-a-specific-resource)
    - [What you can do to make sure you do not play permissions whack-a-mole](#what-you-can-do-to-make-sure-you-do-not-play-permissions-whack-a-mole)
- [Takeaways](#takeaways)

<!-- /MarkdownTOC -->
# Overview
Who am I? Why am I even writing this? What broke that inspired this post?

I am a DevOps Engineer, and in today's world means a part of my job is building and maintaining cloud infrastructure. A lot of maintaining any software infrastucture is implementing and managing permissions to said infrastructure. I have played too many games of permissions whack-a-mole when trying to grant permissions in AWS and wanted to write about it.

## Most Recent Example of permissions misconfiguration (es:* on specific resource)
I recently granted someone "admin" permission to access to a specific domain in AWS' (Amazon Web Services) Elastic Search service with the following policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ElasticsearchAdmin",
            "Effect": "Allow",
            "Action": [
                "es:*"
            ],
            "Resource": [
                "arn:aws:es:us-east-1:1234567890:domain/my-cool-es-domain/*"
            ]
        }
    ]
}
```

But when he tried to access the Elastic Search Domain from the AWS admin console, he got the following error.

```
ListDomainNames: {"Message":"User: arn:aws:sts::1234567890:assumed-role/the_admin_role/the_user_assuming_that_role is not authorized to perform: es:ListDomainNames on resource: arn:aws:es:us-east-1:1234567890:domain/*"}
```

What did I do wrong? I did not grant exactly what the error is asking for `listDomainNames` on `arn:aws:es:us-east-1:1234567890:domain/*`. But how could I have known that to access the AWS admin console for Elastic Search the user would need these permissions? Enter the world of AWS permissions.

Here is my corrected policy for reference:

```
{
    "Version": "2012-10-17",
    "Statement": [
        ...
        {
            "Sid": "ElasticSearchRead",
            "Effect": "Allow",
            "Action": [
                "es:Describe*",
                "es:List*",
                "es:Get*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ElasticsearchAdmin",
            "Effect": "Allow",
            "Action": [
                "es:*"
            ],
            "Resource": [
                "arn:aws:es:us-east-1:1234567890:domain/cool-es-domain",
                "arn:aws:es:us-east-1:1234567890:domain/cool-es-domain/*"
            ]
        }
    ]
}
```

# My key takeaways on AWS IAM permissions
Feel free to [read their documentation][iam_policies], but here are my key takeaways from doing permissions in AWS for a few years.

## 1. I prefer IAM policies over resource policies
In general you can grant access to your AWS infrastucture in two ways:

1. Resource Level Policies
2. IAM Policies

First of all policies are simply a bunch of `statements` that can grant different `actions` to different `resources`. I prefer to grant access via IAM policies over resource level policies (attached to the resource, for example an s3 bucket, or a cloudsearch domain). because they often get forgotten since most administration of permissions gets done in the IAM portion of AWS, where you manage your users.

I have spent a bunch of hours trying to hunt down permissions errors only to find out that there was a resource policy in place. For example say you have an s3 bucket named `brand-new-bucket` and an IAM role authorized to do all s3 actions on said bucket with the following policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::brand-new-bucket/*"
      ]
    }
  ]
}
```

But when you try to download any object from it you get a 403 Access Denied. Since your role permissions are correct, the next place I would check is your bucket to see if it had a bucket policy that is conflicting with your IAM policy. Here is an example of a bucket policy that would conflict with the above IAM policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Principal": "*"
      "Effect": "Deny",
      "Action": [
        "s3:Get*"
      ],
      "Resource": [
        "arn:aws:s3:::brand-new-bucket/*"
      ]
    }
  ]
}
```

_note: Resource Level Policies do have their uses though. One strong opinion I have about resource level policies (ie S3 bucket policies, KMS key policies) is that if you can defer access management to IAM policies by writing a single static resource level policy (typically you grant root of your account(s) `<service_name>:*` permissions on the resource), you should always do that._

## 2. Each actor in your infrastructure should have its own IAM role.
As in each different service or type of server should have its own IAM role and policies. This allows us to really lock down who has access to what resoruces... And hey... AWS does not charge extra for extra IAM resources so... why not?

## 3. Write tight IAM policies
You should always try to write that role's policies with the ["Principle of Least Privilege"][least_priv] in mind. This can be tricky as can be seen in my example above an error appeared when we tried to access the AWS admin panel. So let us move on to talking about that.

# How to do Least Privilege in AWS IAM policies
AWS IAM policies are fairly straight forward and can limit access in different ways. For the purposes of this blog post I will be focusing on iAM policy levers that I like to pull but don't forget that resource level policies can also limit by a `Principal` element.

Let's take a look at a snippet form a sample policy from the aws documentation.

```
{
  "Version": "2012-10-17",
  "Statement": [
    ...
    {
      "Sid": "ThirdStatement",
      "Effect": "Allow",
      "Action": [
        "s3:List*",
        "s3:Get*"
      ],
      "Resource": [
        "arn:aws:s3:::confidential-data",
        "arn:aws:s3:::confidential-data/*"
      ],
      "Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}
    }
  ]
}
```

Let us highlight the important levers that you can pull when limiting permissions and explain some of the different values you can have for them.

* __Action__ - In general `Actions` can be grouped into read or write/admin actions.
  * Read actions - Are generally encompassed in `get*`, `describe*`, and `list*`. (If you want to grant read-only access definitely search the [AWS pre-canned policies][policies_dashboard] for "read" policy examples)
  * Write/Admin actions - If you're giving someone write permissions those tend to be `put*`, `delete*` or just plain old `*`
* __Resource__ - Either specific ARNs (Amazon Resource Name) that can include wildcards or just a `*` can even be placed here.
* __Condition__ - You can key off of different parameters of the AWS API call, I try to stay away from this as it can get hairy fast.

In general you only need a subset of actions (never do `*:*` or `<service_name>:*`) unless you are granting admin permissions. If you are doing any `*` bomb permissions you should almost always be locking down which resource those permissions apply to

## What can go wrong when limiting permissions to a specific resource
In my example above a wild error appeared when we tried to access the AWS admin panel because when viewing the list of domains you need to allow `es:ListDomainNames` (or easier  `es:List*`) on Resource `*` which I did not grant. This happens way more often than you think.

Another Admin panel example would be if i gave someone `s3:*` permissions on a `bucket` and `bucket/*` Resources. They would get an error trying to view the s3 dashboard because they need `s3:ListBuckets` on `*` Resources.

How about a command line example. You grant `s3:*` on a `bucket` and `bucket/*` and someone tries to do an [s3 sync][s3_sync], they'll get an error because they can't `ListBucketContents`

## What you can do to make sure you do not play permissions whack-a-mole
If you are granting AWS permissions on a specific resource... ALWAYS check if you need some `get*`, `list*`, `describe*` permission on all resources (not the one you're trying to grant access to) to actually access those resources programatically or via the admin panel.

_One thing I often do is I check for a canned policy that exists for "Read Only" permissions on the resource you are granting access to. ie in my case there was an `AmazonESReadOnlyAccess` policy that showed me what additional access a user would need in addition to Resource Specific Actions._

Often something I do when creating IAM policies is to always have a minimum of 2 Statements:
* Grant on `*` resource, the actions needed for ReadOnly access (ie `s3:get*`).
* Grant on your `specific resource arn`, admin actions (ie `s3:*`)

# Takeaways
* If you are granting permissions on ANYTHING... ALWAYS ask "is it necessary?"
  * Watch out for the times that you say, "But it really does not hurt if I do this..."
  * When you really need to be asking yourself, "Do I really need to do this?"
* If you are granting AWS permissions on a specific resource... ALWAYS check if you need some `get*`, `list*`, `describe*` permission on all resources (not the one you're trying to grant access to) to actually access those resources programatically or via the admin panel.

[least_priv]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[iam_policies]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
[resource_vs_iam_policies]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html
[policies_dashboard]: https://console.aws.amazon.com/iam/home?region=us-east-1#/policies
[s3_sync]: https://docs.aws.amazon.com/cli/latest/reference/s3/sync.html
