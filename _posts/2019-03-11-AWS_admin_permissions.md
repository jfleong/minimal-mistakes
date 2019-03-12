---
title: "Grant AWS Admin Permissions on a Specific Resource"
layout: single
excerpt: "We should always grant access based on a principle of least privilege. In AWS (Amazon Web Services) you can grant something:* on a very specific resource, that should be admin permissions right? Well in some cases it doesn't... "
sitemap: false
permalink: /granting-aws-admin-permissions-specific-resource
---

# TL;DR
In some cases, to grant admin access for a specific AWS resource you may also need to grant read only access to all resources.

# Background
Who am I? Why am I even writing this? What broke that inspired this post.

I am a DevOps Engineer, and in today's world means a part of my job is building and maintaining cloud infrastructure. A lot of maintaining any software infrastucture is implementing and managing permissions to said infrastructure. I recently granted someone "admin" permission to access to a specific domain in AWS' (Amazon Web Services) Elastic Search service with the following policy:

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

But when he tried to access the Elastic Search Domain from the AWS admin console it he got the following error.

```
ListDomainNames: {"Message":"User: arn:aws:sts::1234567890:assumed-role/the_admin_role/the_user_assuming_that_role is not authorized to perform: es:ListDomainNames on resource: arn:aws:es:us-east-1:1234567890:domain/*"}
```

What did I do wrong? TL;DR - I didn't grant exactly what the error is asking for `listDomainNames` on `arn:aws:es:us-east-1:1234567890:domain/*`. But how could I have known that to access the AWS admin console for Elastic Search the user would need these permissions? Enter the world of AWS permissions.

_This was a AWS admin console example but something similar can be seen if someone is trying to do a `aws s3 sync` from command line and has `get*` permissions but not `listBucket` permissions._

# Managing Permissions in AWS
Feel free to [read their documentation][iam_policies], but here are my key takeaways from doing permissions in AWS for a few years.

## My Takeaways
In general you can grant access to your AWS infrastucture in two ways:

1. Resource Level Policies
2. IAM Policies

Each policy has a bunch of `statements` that can grant different `actions` to different `resources`. With that being said, each actor in your infrastructure should have it's own IAM role and you should always try to write that role's policies with the ["Principle of Least Privilege"][least_priv] in mind.

_One strong opinion I have about resource level policies (ie S3 bucket policies, KMS key policies) is that if you can defer access management to IAM policies by writing a single static resource level policy (typically you grant root of your account(s) `service:*` permissions on the resource, do that), you should always do that. Countless hours have been spent trying to hunt down permissions errors only to find out that there was a step that was missed because there was a resource level policy in place._

## How to do Least Privilege in AWS IAM policies
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

# How to Make Sure You Really Granted Access to a Resource.
* Check for a canned policy that exists for "Read Only" permissions on the resource you are granting access to. ie in my case there was an `AmazonESReadOnlyAccess` policy that showed me what additional access a user would need in addition to Resource Specific Actions.
* Create an IAM policy with a minimum of 2 Statements:
** Grant on `*` resource, the actions from the read only policy
** Grant on your specific resource, admin actions (`service:*`)

# Takeaway
* If you are granting permissions on ANYTHING... ALWAYS ask "is it necessary?"
  * Watch out for the times that you say, "But it really does not hurt if I do this..."
  * When you really need to be asking yourself, "Do I really need to do this?"
* If you are granting AWS permissions on a specific resource... ALWAYS check if you need some get*, list*, describe* permission on all resources (not the one you're trying to grant access to) to actually access those resources programatically or via the admin panel.

[least_priv]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[iam_policies]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
[resource_vs_iam_policies]: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html
[policies_dashboard]: https://console.aws.amazon.com/iam/home?region=us-east-1#/policies

I recently wanted to give access to a secret key to every server in my VPC (Virtual Private Cloud) because one application needed it and it would not hurt if everyone could access it because of some other permission we already had... WRONG. The correct approach would be the provision a new secret key that my application and only my application can use, ala least privilege access.
