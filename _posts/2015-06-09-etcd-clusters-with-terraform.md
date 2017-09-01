---
title: Etcd clusters with Terraform
layout: post
---

# Etcd clusters with Terraform

[Etcd](https://github.com/coreos/etcd) is a "highly-available key-value store for shared
configuration and service discovery." Some of its uses includes taking "advantage of the
consistency guarantees to implement database master elections or do distributed locking
across a cluster of workers." Kubernetes, for instance, is built on top of it.

[Terraform](https://github.com/hashicorp/terraform) is a "tool for building, changing, and
combining infrastructure safely and efficiently." Compared to its ilk (Chef, Salt, etc.)
it does the best job, in my opinion, of hewing to the "immutable infrastructure" and 
"infrastructure as code" concepts, and does so in a powerful an elegant way.

If we wanted to manage an Etcd cluster with Terraform, it would be nice if we could
easily

 1. Have the peers of the cluster discover each other
 2. Increase or decrease the size of the cluster

Terraform makes it fairly easy to spin up a new Etcd cluster, and configure the Etcd
instances to discover each other. It is slightly more challenging to configure
Terraform to gracefully scale the cluster size up or down. In this post I will
share a straightforward way of accomplishing the first goal, and a (somewhat hacky)
way to do the second.

(Terraform is a brilliant tool even in its infancy. It is under rapid development, so
the techniques described here are likely to become obsolete in the near future.)

## Requirements

I will assume that you already have Terraform installed. You will need to have an account
with a cloud provider supported by both CoreOS and Terraform, such as AWS or Google. In
this post, I use Google.

## Initial Configuration

To start off simple, here is a Terraform configuration for starting a three-node Etcd
cluster running on CoreOS, Google Compute instances. The example Terraform configurations
in this post are JSON-formatted, but you could just as easily use Terraform format.

```json
{
    "provider": {
        "google": {
            "account_file": "${var.GOOGLE_ACCOUNT_FILE}",
            "project": "${var.GOOGLE_PROJECT}",
            "region": "us-central1"
        }
    },

    "resource": {
        "google_compute_instance": {
            "etcd": {
                "count": "${var.ETCD_COUNT}",
                "disk": {
                    "image": "coreos-beta-681-0-0-v20150527"
                },
                "machine_type": "n1-standard-1",
                "metadata": {
                    "user-data": "${template_file.etcd_cloud_config.rendered}"
                },
                "name": "etcd-${index.count+1}",
                "network_interface": {
                    "access_config": {},
                    "network": "default"
                },
                "zone": "us-central1-a"
            }
        },
        "template_file": {
            "etcd_cloud_config": {
                "filename": "etcd_cloud_config.yaml.tpl"
            }
        }
    },

    "variable": {
        "ETCD_COUNT": {
            "default": 3
        },
        "GOOGLE_ACCOUNT_FILE": {},
        "GOOGLE_PROJECT": {}
    }
}
```

When Terraform processes this configuration, it reads in `etcd_cloud_config.yaml.tpl`,
and passes it in as metadata to new Google Compute instances. CoreOS is smart enough
to read this metadata and configure itself using any information stored in the `user-data`
key.

For our purposes, the following configuration is sufficient for `etcd_cloud_config.yaml.tpl`.

```yaml
#cloud-config

coreos:
  etcd2:
    # $public_ipv4 and $private_ipv4 are populated by the cloud provider
    advertise-client-urls: http://$public_ipv4:2379
    initial-advertise-peer-urls: http://$private_ipv4:2380
    listen-client-urls: http://0.0.0.0:2379
    listen-peer-urls: http://$private_ipv4:2380
  units:
    - name: etcd2.service
      command: start
```

The last thing we need before spinning up this starter cluster is a `terraform.tfvars.json`
file with our private variables.

```json
{
    "GOOGLE_ACCOUNT_FILE": </path/to/your/account/file.json>,
    "GOOGLE_PROJECT": <your_google_project_id>
}
```

Now we can start the cluster with:

    $ terraform apply

## Peer Discovery

In order for the cluster we spun up to be useful, its members need to become aware of each
other, and elect a master. Etcd provides more than one way to do this, but the slickest way,
in my opinion, is via a [discovery URL](https://coreos.com/docs/cluster-management/setup/cluster-discovery/).

Etcd, when configured to use a discovery URL, will discover other peers that are also
connecting to the same URL. Once enough peers have joined, master election takes place.

The Etcd documentation recommends making an HTTP request to their free, hosted discovery
service in order to generate a new discovery URL. To generate a discovery URL for a three-node
Etcd cluster, we would do

    $ curl -w 'https://discovery.etcd.io/new?size=3' \
    https://discovery.etcd.io/6a28e078895c5ec737174db2419bb2f3

Terraform does not currently have great options for storing a value computed via the shell at
runtime, but we can do so with the combination of a `template_file` provider and a `local-exec`
provisioner:

```json
{
    "provider": {
        "template_file": {
            "depends_on": [
                "template_file.etcd_discovery_url"
            ],
            "etcd_cloud_config": {
                "filename": "etcd_cloud_config.yaml.tpl",
                "vars": {
                    "etcd_discovery_url": "${file(var.ETCD_DISCOVERY_URL)}"
                }
            },
            "etcd_discovery_url": {
                "filename": "/dev/null",
                "provisioner": { 
                    "local-exec": {
                        "command": "curl https://discovery.etcd.io/new?size=${var.ETCD_COUNT} > ${var.ETCD_DISCOVERY_URL}"
                    }
                }
            }
        }
    },

    "var": {
        "ETCD_DISCOVERY_URL": {
            "default": "etcd_discovery_url.txt"
        }
    }
}
```

With these additions, Terraform will save a new discovery URL to `etcd_discovery_url.txt`,
and interpolate the contents of the file into `etcd_cloud_config.yaml.tpl`. In order for
the URL to appear in the rendered version, we need to add a `${etcd_discovery_url}`
variable to our `etcd_cloud_config.yaml.tpl`:

```yaml
#cloud-config

coreos:
  etcd2:
    # Discovery is populated by Terraform
    discovery: ${etcd_discovery_url}
    # $public_ipv4 and $private_ipv4 are populated by the cloud provider
    advertise-client-urls: http://$public_ipv4:2379
    initial-advertise-peer-urls: http://$private_ipv4:2380
    listen-client-urls: http://0.0.0.0:2379
    listen-peer-urls: http://$private_ipv4:2380
  units:
    - name: etcd2.service
      command: start
```

To see this in action, first destroy the cluster:

    $ terraform destroy

Start it back up again:

    $ terraform apply

And inspect our work:

    $ terraform show

You should see the same discovery URL rendered in the cloud config supplied to all three
Etcd instances. To verify that the peers discovered each other and elected a master, you
can log in to the instances and run `sudo journalctl -u etcd2`.

_Why /dev/null?_

You might be wondering why we do not use a real file for the `filename` in the
`etcd_discovery_url` template file resource, and avoid making use of the handy
Terraform mechanism `template_file.<name>.rendered` for reading interpolated
template content.

The reason for this diversion is because Terraform renders the template file *before*
the `local-exec` provisioner runs. Any change made to `filename` by a `local-exec`
provisioner would not be available via `.rendered` until the next Terraform run.

By using `${file(...)}`, we are able to pass provisioned template files to other
resources.

## Scaling

Scaling the cluster up and down is a little bit trickier. If we were managing a fleet
of disconnected workers, we could simply increase or decrease `ETCD_COUNT`. However,
since Etcd peers talk to each other, it is necessary for newly arrived members to
be discovered and accepted into the existing cluster.

If we increase `ETCD_COUNT` from three to five, we get the following error on the
newly add members:

    etcd-4 $ sudo journalctl -u etcd2
    etcd: discovery cluster full, falling back to proxy

In order to add a (non-proxy) peer to an existing cluster with Terraform, we could run
a script to add the new member via the HTTP-based 
[members API](https://github.com/coreos/etcd/blob/master/Documentation/other_apis.md#add-a-member),
or with [`etcdctl`](https://github.com/coreos/etcd/blob/master/Documentation/runtime-configuration.md).

Either approach requires using the following logic for any given `terraform apply`:

 1. Determine if there is an existing cluster
 1. If not, get a discovery URL from https://discovery.etcd.io,
    and launch new instances with that discovery URL
 1. If so, launch new instances without a discovery URL, and add
    them to the existing cluster

While this approach is the most correct, it is difficult to accomplish in Terraform
because Terraform does not currently provide much (or anything at all) in the way of
branching logic.

### Immutable Clusters

Another approach is to simply create a brand new cluster anytime the cluster size is
increased or decreased. I call this the "immutable clusters" approach, because the size
of any given cluster is immutable. In order to change the size of a cluster, an entire
new cluster must be brought up, and the old one destroyed.

Obviously, this approach is only useful if we do not need to preserve data stored on
the discarded cluster, or have a great way to transfer data from it to the new cluster.
In any case, here is how to create a brand new Etcd cluster any time there is a change
in the number of peers.

#### Generate a New Discovery URL

We will need to generate a new discovery URL every time we change `ETCD_COUNT`. The simple
way to do this is to add a var to the `etcd_discovery_url` `template_file` resource:

```json
{
    "etcd_discovery_url": {
        "filename": "/dev/null",
        "provisioner": { 
            "local-exec": {
                "command": "curl https://discovery.etcd.io/new?size=${var.ETCD_COUNT} > ${var.ETCD_DISCOVERY_URL}"
            }
        },
        "vars": {
            "size": "${var.ETCD_COUNT}"
        }
    }
}
```

This will force the resource to be recreated (and there re-run the provisioner) any time the
size of the cluster changes. (I do not know why Terraform does not detect changes inside the
`command` of the `local-exec` provisioner.)

#### Force Re-creation of Existing Peers

Forcing the re-creation of the `etcd_discovery_url` will trigger an update to the metadata
of existing Etcd peers, but the peers themselves will not be re-created nor rebooted, and so
they will not use the new discovery URL. To force the re-creation of a peer, we can change
its name:

```json
{
    "google_compute_instance": {
        "etcd": {
            "count": "${var.ETCD_COUNT}",
            "disk": {
                "image": "coreos-beta-681-0-0-v20150527"
            },
            "machine_type": "n1-standard-1",
            "metadata": {
                "user-data": "${template_file.etcd_cloud_config.rendered}"
            },
            "name": "etcd-${index.count+1}-${var.ETCD_COUNT}",
            "network_interface": {
                "access_config": {},
                "network": "default"
            },
            "zone": "us-central1-a"
        }
    }
}
```

By giving each Etcd peer a name of the form `etcd-n-N`, we ensure that any change in cluster
size changes any existing instance name, forcing its re-creation and usage of the new
discovery URL.

## Example Code

[https://github.com/maxenglander/etcd-terraform-example](https://github.com/maxenglander/etcd-terraform-example)
