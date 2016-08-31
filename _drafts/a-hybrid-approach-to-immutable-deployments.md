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

Instead, I want to make a case for why it makes sense to have both,
and to sketch out a way to do so with minimal complexity.

## Immutable deploys can be time-consuming

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
    from the AMI, run a set of tests against it,
    and, finally, make it live

Your system may include many more or fewer steps than these.
However many steps there are, and however automated they are, chances are
that they will still slow things down quite a bit.
In my experience, spinning up and shutting
down AWS instances both take a minute or more; creating AMIs
from them is even slower; copying those AMIs between regions is slowest.
Perhaps Google Cloud is much faster, but I'm going to assume here that
it is slow enough to be painful in situations when code needs to get out
quickly.

When a money-eating bug is making your customers sad, you may not want
or be able to wait the extra minutes to fix it in an immutable fashion.

## What about a hybrid approach?

One kind of hybrid approach favors the immutable procedure, such as
the one outlined above, while supporting faster procedures when needed,
such as:

 * Creating a new instance from the previous image, placing a code
   change on the new instance, and making the new instance live
 * Placing a code change on a live, running instance

This kind of hybrid system confers all of the benefits of the immutable
approach, while supporting two kinds of mutable code deploys when time
is scarce.

### The challenge

The challenge of designing a system that can support both
immutable and mutable procedures is to avoid creating a disjoint set
of tools, scripts, and services to support each procedure.

Failing to do so would result not only in increased time to
implement and maintain each procedure, but would also require
operators to possess additional knowledge and skills.

In short, part of the success of any hybrid system will depend
on its ability to re-use as many of the tools used by any given
procedure in every other procedure.

## Building a hybrid

With the motivation and challenge stated, the stage is now set
to begin sketching out an implementation that supports multiple
deploy procedures, while maximimizing the overlap in the tools
used by each.

### The three procedures

The three procedures outlined below are not presented as
best practice, not as exhausting all of the requirements of
any given organization.

However, as abstractions, they all perform at least one key
step in a slightly different way: getting new code into
production.

#### Fully immutable

 1. Bake a runnable image, such as a Docker image or an
    AWS AMI, containing new application code and any
    other components it depends on
 1. Instantiate a server or set of servers, be they
    Docker containers, AWS EC2 instances, or equivalent,
    with the image
 1. Perform any post-deploy tasks, such as mounting
    persistent storage, running smoke-tests, etc.

#### Partially immutable

 1. Use the image created in the first step of the fully
    immutable procedure to instantiate a new server
    or group of servers
 1. Replace the existing application code on the instantiated
    server(s) with new application code
 1. Perform any post-deploy tasks, such as mounting
    persistent storage, running smoke-tests, etc.

### Mutable

 1. Replace the existing application code on the existing
    server or set of servers with new application code
 1. Perform any post-deploy tasks, such as mounting
    persistent storage, running smoke-tests, etc.
