---
title: Deployments (and Rollbacks) with AWS and Python
layout: post
---

<a href="top"></a>

# Deployments (and Rollbacks) with AWS and Python

Creating a deploy process for AWS in Python over the last couple of weeks
presented some interesting frustrations. What follows are my observations on
some of the fundamental differences between local builds vs. deploys, how
those differences impact the design of their processes.

## Context

The build process that I developed looked roughly like this:

1. Take an image of the live, production instance
1. Launch a new instance from that image
1. Deploy new code
1. Dump and restore the live database to new database
1. Run migrations on the new database
1. Run smoketests
1. Update DNS record with IP address of new instance

<a href="tools"></a>

With these tools:

* [boto](https://github.com/paramiko/paramiko)
* [paramiko](https://github.com/paramiko/paramiko)
* [pyinvoke](https://github.com/pyinvoke/invoke)

This build process obviously has flaws, but it was a good beginning for the
budding startup that I was contracted to do the work for. The rest of this 
post is written with the above process in mind, particularly the part about 
launching (and shutting down) new EC2 instances.

## Naming Resources and Tracking State

One of the special characteristics of cloud deployments, as opposed to
a local build procedure, is that in a local build procedure you generally
control the naming of artifacts (such as files and folders). In a PaaS such
as AWS, you often do not. For example, AWS reserves the responsibility of 
assigning it a unique IDs to EC2 instances.

This difference entails consequences for how cleanups and rollbacks are 
handled during a failed build process.

If, during a local build procedure, a stage fails (maybe `libcurl` could not
be found), the process for cleaning up is usually simple and invariable:
`rm -r build/.*` or something similar. It is not usually necessary to track
the names of files and folders produced during the build, because, as a rule,
they are pre-determined by the build process. Traditional build tools are
well-geared to handle these sorts of rules.

Contrast this situation to an aborted cloud deploy. If, after launching 
a dozen new AWS instances, a subsequent stage fails, we are now in a position
where we have to rollback (e.g. shutdown) those dozen instances. Since
we did not generate the unique identifiers for those instances, they must be
tracked somewhere.

How to keep track of the unique identifiers?

### Resource tagging

AWS allows tagging of EC2 resources. Given this, a deploy process
could launch new EC2 instances, always tagging them with the tag and value.
Rolling back after a failed deploy, in this case, would be a matter of
finding the instance(s) with this tag and value, and shutting it/them down.

This allows the deploy process to forget about tracking resource IDs, and
behave more like a local build process, by predetermining an attribute
of the new resources.

A couple of concerns:

1. AWS does not offer a way to enforce uniqueness of a key across resources
1. Launching-and-tagging is a two-step over-the-network process

In other words, I am concerned about maintaining the correct tags and values
on instances.

### Local filesystem tracking

Would look something like this

1. Launch a bunch of stand by instances -> write their IDs to `./build/ids` 
1. Run smoketests, the fail -> abort and rollback
1. Shutdown all instances in `./build/ids`

### Track in central storage

Like filesystem tracking, except instead of storing deploy history locally,
write to a remote storage system such as S3.

## Dirty Filesystems vs. Dirty Clouds

Another characteristic difference between local builds and deploys is that
local builds produce limited artifacts, whereas cloud deploys can,
depending on the design of the deploy process, create new ones endlessly.

Compiling a package locally produces artifacts in, typically, the package
folder, `/tmp`, and whatever paths installation artifacts occupy. Re-running
a `./configure; make intall` produces new artifacts, in a sense, but over-
writes the previous ones, thus capping the total artifact count.

Because of the way AWS EC2 instances are created, on the other hand, a deploy
process could easily proliferate instances unless care is taken to ensure
that resources created during previous, aborted deploys are shutdown.

(Why produce new artifacts instead of just deploying to the same set of
instances over and over?
[Here](https://blog.codeship.com/immutable-deployments/) is a great look
at that question.)

On a related note, we *care* more about resources that are produced during
an aborted deploy than we do about a dirty filesystem produced by a failed
local build, because EC2 instances, generally speaking, cost more money! 

### Avoiding resource proliferation

A simple way to avoid resource proliferation is to forbid a deploy process
to proceed when there is a previous, aborted deploy that was not fully
cleaned up. This is very different from the way local build processes work,
where `./configure; make install` can be run with impunity at no cost.

Preventing a deploy from running when there is "dirt" could look
something like this:

1. Check for presence of deploy log
1. Abort deploy if present
1. Create deploy log
1. Write instance IDs to deploy log
1. Shutdown IDs in deploy log upon abort
1. When deploy finishes, archive/rotate deploy log
1. Delete deploy log

### Cleaning up resources in a timely manner

In some cases, it may be OK to leave the cloud "dirty" for a while
after a failed deploy. Resources can be cleaned up by the following deploy
attempt. However, if the cost of leaving the cloud dirty for a long time
is a concern, cleaning up during the deploy itself is an option, and
looks something like this:

{% highlight python %}
@task
def deploy_standby(instance_id):
    image_id = create_image(instance_id)
    try:
        new_instance = run_instance(image_id)
        ip_address = new_instance.ip_address
        try:
            scp(code, ip_address)
            ssh(ip_address, 'httpd restart')
            tests_pass = smoketest(ip_address)
            if not tests_pass:
                raise AbortDeploy()
        except AbortDeploy as e:
            terminate_instance(new_instance.id)
            raise e
    except AbortDeploy as e:
        deregister_image(image_id)
        raise e
{% endhighlight %}

## Conclusion

Some cloud deploys can benefit from following rules that differ
from those followed by traditional, locally based build processes.

1. Deploy processes keep track of newly created resource IDs since
they are defined by the PaaS and are needed during cleanup/rollback
1. Deploy processes do not run if there is a previous, uncleaned
failed deploy
1. Failed deploys clean up after themselves

In subsequence posts, I will look at implementing some of these
rules with the [tools](#tools) I listed at the top.
