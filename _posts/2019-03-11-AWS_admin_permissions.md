---
title: "Grant AWS Admin Permissions on a Specific Resource"
layout: single
excerpt: "We should always grant access based on a principle of least privilege. In AWS (Amazon Web Services) you can grant something:* on a very specific resource, that should be admin permissions right? Well in some cases it doesn't... "
sitemap: false
permalink: /granting-aws-admin-permissions-specific-resource
---

# TL;DR
In some cases, to grant admin access for a specific AWS resource you may also need to grant read only access to all resources.

# Managing Permissions is an Extremely Important Part of the Job
I am a DevOps Engineer, and in today's world means a part of my job is building and maintaining cloud infrastructure. In most cases that cloud infrastucture resides in AWS (Amazon Web Services) and is maintained via some "Infrastructure as Code" tool like CloudFormation or Terraform. A lot of maintaining any software infrastucture is implementing and managing permissions to said infrastructure. I mainly work in AWS so let's start with how permissions are done in AWS.

# My Takeaways on AWS Permissions
Feel free to [read their documentation][iam_policies], but here are my key takeaways from doing permissions in AWS for a few years.

In general you can grant access to your AWS infrastucture in two ways:

1. Resource Level Policies
2. IAM Policies

Each policy has a bunch of `statements` that can grant different `actions` to different `resources`. With that being said, each actor in your infrastructure should have it's own IAM role and you should always try to write that role's policies with the ["Principle of Least Privilege"][least_priv] in mind.

_One strong opinion I have about resource level policies (ie S3 bucket policies, KMS key policies) is that if you can defer access management to IAM policies by writing a single static resource level policy (typically you grant root of your account(s) `service:*` permissions on the resource, do that), you should always do that. Countless hours have been spent trying to hunt down permissions errors only to find out that there was a step that was missed because there was a resource level policy in place._

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

Let us highlight the important levers that you can pull when limiting permissions.

* __Action__ - In general `Actions` can be grouped into read or write/admin actions.
  * Read actions - Are generally encompassed in `get*`, `describe*`, and `list*`. (If you want to grant read-only access definitely search the [AWS pre-canned policies][policies_dashboard] for "read" policy examples)
  * Write/Admin actions - If you're giving someone write permissions those tend to be `put*`, `delete*` or just plain old `*`
* __Resource__ - Either specific ARNs (Amazon Resource Name) that can include wildcards or just a `*` can even be placed here.
* __Condition__ - You can key off of different parameters of the AWS API call, I try to stay away from this as it can get hairy fast.

I recently wanted to give access to a secret key to every server in my VPC (Virtual Private Cloud) because one application needed it and it would not hurt if everyone could access it because of some other permission we already had... WRONG. The correct approach would be the provision a new secret key that my application and only my application can use, ala least privilege access.

# So what broke that inspired this post
* Accessing the Elastic Search service via the AWS management console was broken because of missing listDomains on resource *

# What I do whenever I need to grant access to a specific resource in AWS
* Check for a canned policy that exists for "Read Only" permissions on the resource you are granting access to
* Create an IAM policy with a minimum of 2 Statements:
** Grant on `*` resource, the actions from the read only policy
** Grant on your specific resource, admin actions (`service:*`)

# Takeaway
* If you are granting permissions on ANYTHING... ALWAYS ask "is it necessary?"
  * Watch out for the times that you say, "But it really does not hurt if I do this..."
  * When you really need to be asking yourself, "Do I really need to do this?"
* If you are granting AWS permissions on a specific resource... ALWAYS check if you need som get*, list*, describe* to actually access those resources programatically or via the admin panel.

[least_priv]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[iam_policies]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
[resource_vs_iam_policies]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html
[policies_dashboard]: https://console.aws.amazon.com/iam/home?region=us-east-1#/policies
