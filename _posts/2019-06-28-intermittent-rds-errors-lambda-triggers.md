---
layout: post
title:  "Intermittent RDS Errors When Triggering Lambda Functions"
date:   2019-06-28 9:00:00 +0000
categories: [aws]
---

In recent years we've started moving our infrastructure from dedicated hosts such 
as OVH to cloud providers like AWS. Now this post is not about the pros and cons
of cloud providers, though there are many, but rather a specific issue we faced
when using AWS [RDS](https://aws.amazon.com/rds/) in conjunction with AWS 
[Lambda](https://aws.amazon.com/lambda/).

We store our user details in an RDS instance which is then consumed by 
an internal API. We also have an Elasticsearch cluster which stores certain 
modified user information for analysis and other purposes. MySQL triggers are
used to invoke a Lambda function that upserts a modified version of the user 
information to our Elasticsearch cluster.

At first this system seemed to be working perfectly. Indeed we had many other
RDS instances triggering Lambda functions hundreds of times a day successfully.
However, it then became clear that the MySQL triggers were occasionally failing,
resulting in outdated data in our Elasticsearch. The issue appeared to be random.
We could not consistently replicate it despite trying various approaches. We removed
the triggers to negate the impact it was having on our production infrastructure and
created a clone of the RDS instance for testing. This is the error we were faced with
when we managed to break the trigger:

> Statement could not be executed (42000 - 1064 - Unknown trigger has an error in its 
body: 'Access denied; you need (at least one of) the Invoke Lambda privilege(s) for 
this operation')

Access denied, how could that only be occurring part of the time? I checked that
our cluster had the correct IAM role as well as the correct VPC security groups.
They were the same as our working instances. Googling the issue did not unearth
anyone with similar symptoms to us, only the same error message when they had
misconfigured their IAM roles. We were stumped.

Whilst reading the documentation on MySQL triggers on RDS, we noticed that
we were using MySQL version 5.6 where RDS now supported 5.7 too. The documentation
did not mention that we may be facing our issue due to the version difference, but
we figured it was worth performing the upgrade regardless. Secretly, we hoped it
would cause our problem to disappear too. The upgrade was performed and... our 
triggers were still breaking.

At this point we decided to call in backup and contact AWS support. Realistically
we should have done this much earlier. We opened a ticket, explaining our issue
and the error messages we were getting as well as the things we had tried. 

Now some background on our infrastructure on AWS. Our infrastructure is inside a 
[VPC](https://aws.amazon.com/vpc/) so that we can define security groups and
access control lists to our resources. We have several subnets in this VPC, three of
which have access to a NAT Gateway. The NAT Gateway allows our private instances to
access the internet without exposing their private IP. The reason we have three subnets
is that the subnets are availability zone specific and there are three availability zones
in the London region.

While waiting for a reply from AWS, I compared the configuration of a working RDS
instance to our user information instance for what felt like the hundredth time.
Suddenly I spotted it, our user information instance was only in subnet `a` and `b`.
Was it possible that the subnet was causing these occasional errors? Unfortunately
you cannot modify the subnets of an existing RDS instance so I created a new clone
of the instance with all 3 subnets configured.

Once our new testing instance was ready, I created our triggers and started to test
them. They worked every time. It then became obvious to me that the reason the issue
appeared random was that our requests were being load balanced across the 3 subnets.
Any requests to subnet `c` would fail, resulting in a 66% success rate for the trigger
invocation.

We recreated our RDS instance with the correct subnets and pointed our applications at 
the new host, fixing the issue for good.

Looking back, the issue and the solution seem obvious. At the time it left us puzzled,
especially since we could not find any similar cases online.