---
title: "A hybrid approach for (im)mutable application deployment"
layout: post
---

# A hybrid approach for (im)mutable application deployment

If one of your new car's airbags has deployed or is faulty,
do you replace just the airbag, or the entire car? Chances are,
you would say "airbag", unless you lived in a world with
prohibitively restrictive automotive regulations, or where
the only available replacements included volatile materials like
[ammonium nitrate](http://www.nytimes.com/2016/08/27/business/takata-airbag-recall-crisis.html).

If your production web application has a bug that is wreaking
unexpected havoc, do you deploy a patched application artifact
to the running virtual server, or spin up a new server with
the latest artifact? This question, posed to today's IT
community, is likely to draw a
[more](https://highops.com/insights/immutable-infrastructure-hangout-recordings/)
[mixed](https://blog.chef.io/2014/06/23/immutable-infrastructure-practical-or-not/)
[response](https://blog.codeship.com/immutable-infrastructure/).

That tension between the decision to repair or replace, is the
subject of this post. The first section teases at the factors
which, when given different weights, increase the appeal of one 
or the other alternative. The second section attempts to make
a case for a hybrid approach to application deployment which
supports both the replacement and the repair of (virtual) servers.
The last section describes a deploy system that illustrates
the hybrid approach.

## Considering the factors

While by no means exhausting all of the factors one could
consider when evaluating a choice to repair or replace a
server, the elements of time, cost and safety are, I think,
a sensible starting list.

### Cost, time, and safety

*Cost.*
Prior to cloud computing, replacing a server was by and large 
synonymous with replacing a physical machine. In that era,
assuming application compabitility with the current machine,
requisitioning a new machine to run the latest application would
be not only unjustifiable, but probably unheard of. In the
present era, however, servers are often virtual and can be
exchanged for a new one at no monetary cost.

*Time.*
What about time? Surely, you might say, it must take more
time to spin up a new server, physical or virtual, than it does to
simply replace an application running on it. Metrics may vary
across cloud providers, but my experience with AWS and, to a
lesser extent, with Google Cloud, give me no reason to disagree.
When the layer of virtualization moves a level higher, however,
to the world of containers (LXC, Docker, Rkt, etc.), the difference
in time between starting a new container with the latest application
code and updating the application code inside an existing container
may be small enough to ignore. On the other hand, if the
changes that need to be made to a server are time-consuming,
and multiple servers performing the same role need to be
updated, it may less time-consuming to create an image containing
the new application and the server, and deploy that to
however many new servers are needed to replace the existing one(s).

*Safety.*
Safety largely depends on what precautions are taken prior to
deploying the application, on the one hand, or, on the other,
a new server. One would like for the patched application to be
subjected to a barrage of integration tests, and, furthermore,
for the test environment to bear the greatest possible resemblance
to he production environment. If we replace the buggy application
with a patched version on an existing server, it's difficult to
guarantee that the existing server is otherwise identical to
the environment where patched version was tested. Perhaps,
in the course of wreaking havoc, the buggy application badly
polluted the server in ways that are difficult or impossible to
reproduce in the test environment. If, on the other hand, we create
a new server, deploy the patched application to it, subject it to our
barrage of integration tests, and then make the new server live,
we ensure that environments in which the new application are tested
and deployed are not just equivalent, but identical.

From the above discussion, repairing an existing server holds more
appeal in situations where the additional time to wait for a new
server (or container) to start is unacceptable. When the safety
guarantee of a new server cannot be sacrificed, replacing the
server (or container) is a more compelling option. Cost, in the
era of virtual appliances, doesn't matter.

### Immutability and its riches

The "freshness" guarantee of a new server, and the potential
time-savings of pre-baking an image to be deployed to multiple
servers performing the same role, are, I believe, the central
pillars upon which patterns like
["phoenix server"](http://martinfowler.com/bliki/PhoenixServer.html)
and
["immutable infrastructure"](http://chadfowler.com/2013/06/23/immutable-deployments.html)
are built.

While pre-baking an image containing a new server is not the
only way to achieve immutability, as a technique it does provide
benefits which a configuration management system like Chef
does not. Florian Motlik of Codeship
[points out](https://blog.codeship.com/immutable-infrastructure/)
that by deploying an image, a transition to a new server can
done in a potentially reduced number of steps, and that rollbacks
are simplified, for example. These properties alone may make
replacing a server by starting a new one from an image the
preferred deployment strategy. Similarly, Hashicorp
[advocates](https://www.hashicorp.com/blog/atlas-mindset.html)
using [packer](https://www.packer.io/), a tool for creating
images for AWS, Docker, etc. as part of a larger strategy for
having repeatable, testable, and auditable infrastructure.

# Links

https://www.oreilly.com/ideas/an-introduction-to-immutable-infrastructure
http://chadfowler.com/2013/06/23/immutable-deployments.html
http://martinfowler.com/bliki/ImmutableServer.html
http://martinfowler.com/bliki/PhoenixServer.html
http://martinfowler.com/bliki/SnowflakeServer.html
https://blog.chef.io/2014/06/23/immutable-infrastructure-practical-or-not/
