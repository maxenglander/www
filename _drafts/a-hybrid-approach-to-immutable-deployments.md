---
title: A hybrid approach for (im)mutable application deployment
layout: post
---

# A hybrid approach for (im)mutable application deployment

This post is addressed to the hopefully non-fictional audience that,
like me, has swallowed the "immutable infrastructure" Kool-Aid, but
wants to also support "mutable" deploys when time is limited,
*without having to maintain two distinct deployment systems*.

I won't discuss what
["immutable infrastructure"](https://www.oreilly.com/ideas/an-introduction-to-immutable-infrastructure)
is, how it differs from its "mutable" predecessor, nor advocate for one
approach over the other.

Instead, I want to present a way to support both approaches with
minimal complexity.

## Immutable is slower

For all of the advantages of an immutable approach to deployment,
there is a particular drawback which, for me, is sufficient motivation
for adopting a hybrid system: time. Going immutable means expanding
a traditional deployment procedure beyond placing a code change on a
running instance to include, for example:

 1. Starting an AWS EC2 t2.micro instance running Ubuntu
 1. SSHing into the instance
 1. Downloading a bunch of packages with `apt-get`
 1. Installing your latest application artifact
 1. Shutting down the EC2 instance
 1. Creating and storing an AMI from the stopped instance
 1. Coping the AMI to 2 other AWS regions
 1. Notifing Jenkins to create a new EC2 instance
    from the AMI, run a set of ests against it,
    and, finally, make it live

While, naturally, all of these steps can and would be automated, they
still slow things down. In my experience, spinning up and shutting
down AWS instances both take a minute or more; creating AMIs
from them is even slower; copying those AMIs between regions is slowest.

When a money-eating bug is making your customers sad, you may not want
or be able to wait the extra minutes to fix it in an immutable fashion.

## Designing a hybrid

A hybrid approach would favor the immutable procedure, such as the one 
outlined above, while supporting faster procedures when needed,
such as:

 * Creating a new instance from the previous image, placing a code
   change on the new instance, and making the new instance live
 * Placing a code change on a live, running instance

The challenge we face in designing a hybrid system is to avoid creating
a disjoint set of tools, scripts, and services to support each procedure
beyond the first. Failing to do so would result not only in increased
time to implementation and test each procedure, but would also require
operators to possess additional knowledge and skills. Procedures with
disjoint implementations might mean that, for any change to one
procedure, a maintainer has to port that change to the other two.
