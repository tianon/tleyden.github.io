---
layout: post
title: "Running a Sync Gateway Cluster Under CoreOS on AWS"
date: 2014-12-15 19:22
comments: true
categories: couchbase-mobile sync-gateway coreos aws
---

Follow the steps below to create a Sync Gateway + Couchbase Server cluster running under AWS with the following architecture:

![architecture diagram](http://tleyden-misc.s3.amazonaws.com/blog_images/sync-gw-coreos-onion.png)

Disclaimer: this approach to running Couchbase Server and Sync Gateway is entirely **experimental** and is not recommended for running a production system yet.  There are [known issues](https://github.com/couchbaselabs/couchbase-server-docker/issues/2), and I'm working with the Couchbase support team to get these resolved.

### Screencast available

There is a [screencast video](https://www.youtube.com/watch?v=7-7jsLzHsWU) (12 mins) which walks through this entire setup process, or you can follow the instructions below.  

## Kick off Couchbase Server + Sync Gateway cluster

### Launch EC2 instances

Go to the [Cloudformation Wizard](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#cstack=sn%7ECouchbase-CoreOS%7Cturl%7Ehttp://tleyden-misc.s3.amazonaws.com/couchbase-coreos/sync_gateway.template) 

Recommended values:

* **ClusterSize**: 3 nodes (default)
* **Discovery URL**:  as it says, you need to grab a new token from https://discovery.etcd.io/new and paste it in the box.
* **KeyPair**: the name of the AWS keypair you want to use.  If you haven't already, you'll want to upload your local ssh key into AWS and create a named keypair.

### ssh into a CoreOS instance

Go to the AWS console under EC2 instances and find the public ip of one of your newly launched CoreOS instances.  

Choose any one of them (it doesn't matter which), and ssh into it as the **core** user with the cert provided in the previous step:

```
$ ssh -i aws.cer -A core@ec2-54-83-80-161.compute-1.amazonaws.com
```

## Sanity check

Let's make sure the CoreOS cluster is healthy first:

```
$ fleetctl list-machines
```

This should return a list of machines in the cluster, like this:

```
MACHINE	        IP              METADATA
03b08680...     10.33.185.16    -
209a8a2e...     10.164.175.9    -
25dd84b7...     10.13.180.194   -
```

### Kick off cluster

From the CoreOS machine you ssh'd into in the previous step:

```
$ wget https://raw.githubusercontent.com/tleyden/sync-gateway-coreos/master/scripts/sync-gw-cluster-init.sh
$ chmod +x sync-gw-cluster-init.sh
$ SG_CONFIG_URL=https://gist.githubusercontent.com/tleyden/4ae1fe9b2b18783708cd/raw/fe4fb7f8637c1bf813c70e957bac35fa5ad28d01/sync_gw_config.json
$ ./sync-gw-cluster-init.sh -n 1 -c master -b "todos" -z 512 -g $SG_CONFIG_URL -v 3.0.1 -m 3 -u user:passw0rd
```

You'll want to use your own config URL for the SG_CONFIG_URL value.  For example, a file hosted in github or on your own webserver.  

Parameters to sync-gw-cluster-init.sh:

* **-n** -- the number of sync gateway nodes to start.
* **-c** -- the commit or branch name of sync gateway to use.  (if master has issues, the latest tested commit is currently: 422ba63a1afedd459bffec3031a8680f2daffd5e)
* **-b** -- the couchbase bucket to be created before starting sync gateway
* **-z** -- the size of the bucket in MB to use for the bucket created via the **-b** param.
* **-g** -- the url of the sync gateway config.  You will want to customize this.
* **-v** -- the version of couchbase server to launch (3.0.1 | 2.2.0)
* **-m** -- the number of couchbase server nodes to launch.
* **-u** -- the username and password in a single string separated by a colon.  You will want to customize this to use something sensical.

### View cluster

After the above script finishes, run `fleetctl list-units` to list the services in your cluster, and you should see:

```
UNIT						MACHINE				ACTIVE	SUB
couchbase_bootstrap_node.service                281cd575.../10.150.73.56        active	running
couchbase_bootstrap_node_announce.service       281cd575.../10.150.73.56        active	running
couchbase_node.1.service                        36ab135c.../10.79.132.157       active	running
couchbase_node.2.service                        f815a846.../10.51.179.214       active	running
sync_gw_announce@1.service                      36ab135c.../10.79.132.157       active	running
sync_gw_node@1.service                          36ab135c.../10.79.132.157       active	running
```

They should all be in the `active` state.  If any are in the `activating` state -- which is normal because it might take some time to download the docker image -- then you should wait until they are all active before continuing.

## Verify internal

**Find internal ip**

```
$ fleetctl list-units
sync_gw_node.1.service				209a8a2e.../10.164.175.9	active	running
```

**Curl**

On the CoreOS instance you are already ssh'd into, Use the ip found above and run a curl request against the server root:

```
$ curl 10.164.175.9:4984
{"couchdb":"Welcome","vendor":{"name":"Couchbase Sync Gateway","version":1},"version":"Couchbase Sync Gateway/master(6356065)"}
```

## Verify external

**Find external ip**

Using the internal ip found above, go to the EC2 Instances section of the AWS console, and hunt around until you find the instance with that internal ip, and then get the public ip for that instance, eg: `ec2-54-211-206-18.compute-1.amazonaws.com`

**Curl**

From your laptop, use the ip found above and run a curl request against the server root:

```
$ curl ec2-54-211-206-18.compute-1.amazonaws.com:4984
{"couchdb":"Welcome","vendor":{"name":"Couchbase Sync Gateway","version":1},"version":"Couchbase Sync Gateway/master(6356065)"}
```

Congratulations!  You now have a Couchbase Server + Sync Gateway cluster running.

## Known issues

* [Does not handle node restarts (in particular bootstrap node)](https://github.com/couchbaselabs/couchbase-server-docker/issues/2)

## Appendix A: Kicking off more Sync Gateway nodes.

To launch two more Sync Gateway nodes, run the following command:

```
$ fleetctl start sync_gw_node@{2..3}.service && fleetctl start sync_gw_announce@{2..3}.service
```

## Appendix B: Setting up Elastic Load Balancer.

Setup an Elastic Load Balancer with the following settings:

![elb screenshot](http://tleyden-misc.s3.amazonaws.com/blog_images/sync_gateway_coreos_elb.png)

Note that it forwards to **port 4984**.

Once the Load Balancer has been created, go to its configuration to get its DNS name:

![elb screenshot](http://tleyden-misc.s3.amazonaws.com/blog_images/sync_gateway_coreos_elb2.png)

Now you should be able to run curl against that:

```
$ curl http://coreos-322270867.us-east-1.elb.amazonaws.com/
{"couchdb":"Welcome","vendor":{"name":"Couchbase Sync Gateway","version":1},"version":"Couchbase Sync Gateway/master(b47aee8)"}
```

What next?  You could try running [GrocerySync-Android](https://github.com/couchbaselabs/GrocerySync-Android) or [GrocerySync-iOS](https://github.com/couchbaselabs/Grocery-Sync-iOS) and pointing the Sync Gateway URL to your own Sync Gateway instance.

## Appendix C: Shutting down the cluster.

Warning: if you try to shutdown the individual ec2 instances, the **CloudFormation template will cause AWS to restart the instances**!

To shutdown the cluster, you must use the CloudFormation console.  

![screenshot](http://tleyden-misc.s3.amazonaws.com/blog_images/shutdown_cluster.png)

## References

* [youtube screencast](https://www.youtube.com/watch?v=7-7jsLzHsWU) (12 mins) 
* [sync gateway Docker + CoreOS fleet files](https://github.com/tleyden/sync-gateway-coreos)
* [couchbase-server-coreos](https://github.com/tleyden/couchbase-server-coreos)
* [Sync Gateway docs regarding reverse proxies](http://developer.couchbase.com/mobile/develop/guides/sync-gateway/nginx/index.html)
* [Couchbase Mobile Google Group discussion on ELB](https://groups.google.com/forum/?utm_medium=email&utm_source=footer#!msg/mobile-couchbase/pXKQIAiCaW8/s9W_gSfRL50J)
* [sync gateway](https://github.com/couchbase/sync_gateway)
