---
title: "Consul, Packer, Terraform: a tale of a 3-headed deploy system"
layout: post
---

This post is a part-history and part-guide to the creation of
a flexible setup for deploying infrastructure and application code.
In an initial draft, I tried to make this system appear an inevitable
outcome of inarguably superior design goals. In reality, I arrived
where I am because I drank some heady immutable infrastructure potion,
tried to follow the righteous 12-factor path, but was forced by
limited time and money (I work at a modestly funded startup) to take
occasional refuge in a duct-taped-together shelter. What emerged in the
end was a deploy system that supports a few variants to getting
application code into production. In historical order, they are: 

 1. Trigger an upgrade of an application by changing a Consul key
 2. Bundle application code in an immutable image with Packer,
    and use Terraform to provision a new cloud-based server from that
    image
 3. Use the Terraform Consul provider to combine an out-of-date
    image with the latest application artifact


While my initial goal was to reach (2), I found that, once I got there,
(1) was a much more practical way to get code into production.
Later, I found that (3) was also useful, and could be had at no extra cost
once (1) and (2) were in place.

(1) was inspired by Josh Symmonds' blog post on deploying code with
Consul, and (2) is based on methodology embodied in Hashicorp's Atlas
product. If you're looking to implement just one of those two variants,
then look no further than the respective hyperlinks provided above.
If you're curious about (3), or how to seamlessly combine all variants
into a practical, flexible system, then read on, dear reader.

While AWS, Ansible, and Maven also play a part in this story, their roles
are somewhat more interchangeable (say, with Google Cloud, Salt and Gradle)
than the titular tools. I prefer to refer to specific tools rather than
more abstract terms like "cloud", "image" and "build tool" because
doing so makes a friendlier read.

## Part I - In which I recall unpleasant memories

It was November 2015, and I was working feverish days and nights
on a workplace meditation product, hoping to get it into the hands
of our first customers before the holidays. Things were working locally.
For simplicity's sake, I'll pretend these things were just a single Java
application. In any case, as the time for putting it into production grew
near, I began to recall some dim memories of previous deploy systems
I have known.
