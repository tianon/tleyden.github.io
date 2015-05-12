---
layout: post
title: "Couchbase Server on Google Container Engine"
date: 2015-04-26 01:57
comments: true
categories: 
---

## Install docker container with cloud-sdk

```
$ docker pull google/cloud-sdk
$ docker run -ti google/cloud-sdk /bin/bash
# gcloud auth login
Go to the following link in your browser:
... etc
```

Now enable the alpha components:

```
# gcloud components update alpha
```

And set the zone:

```
# gcloud config set compute/zone us-central1-b
```

## Create a new project

Go to the [New Project](https://console.developers.google.com/project) page on Google Compute Engine and create a new project called `couchbase-container`.

```
# gcloud config set project couchbase-container
```

Verify this worked by trying to list the instances (should be empty)

```
# gcloud compute instances list
NAME ZONE MACHINE_TYPE INTERNAL_IP EXTERNAL_IP STATUS
```

## Create a cluster

```
# gcloud alpha container clusters create couchbase-server \
    --num-nodes 2 \
    --machine-type g1-small
```

Set your default cluster:

```
# gcloud config set container/cluster couchbase-server
```

## Create two pods

```
# wget https://gist.githubusercontent.com/tleyden/7cc7336f6f68f958e2b1/raw/a5b67223f8779004ae3e45dcc37f38ccf8a531de/couchbase-server.yaml
# gcloud alpha container kubectl create -f couchbase-server.yaml
# wget https://gist.githubusercontent.com/tleyden/ed1303f2f40a04c21811/raw/2d323556f8debb4e79d79fa67134e2eaa9866e32/couchbase-server-2.yaml
# gcloud alpha container kubectl create -f couchbase-server-2.yaml
```

## Expose port 8091 to public IP

** First couchbase server node **

```
# gcloud compute instances add-tags k8s-couchbase-server-node-1 --tags cb1
# gcloud compute firewall-rules create cbs-8091 --allow tcp:8091 --target-tags cb1
```

** Second couchbase server node **

```
# gcloud compute instances add-tags k8s-couchbase-server-node-2 --tags cb2
# gcloud compute firewall-rules create cbs2-8091 --allow tcp:8091 --target-tags cb2
```


## Find internal routable IP addresses of pods

```
# gcloud alpha container kubectl get pods
POD                                                   IP           CONTAINER(S)              IMAGE(S)                                                                            HOST                                        LABELS                                                              STATUS    CREATED
couchbase-server                                      10.248.1.3   couchbase-server          couchbase/server                                                                    k8s-couchbase-server-node-1/104.197.79.56   <none>                                                              Running   22 minutes
couchbase-server2                                     10.248.2.4   couchbase-server2         couchbase/server                                                                    k8s-couchbase-server-node-2/146.148.85.81   <none>                                                              Running   20 minutes
```

So 10.248.1.3 and 10.248.2.4 are the routable IP addresses of the two pods.

## Ssh into one of the GCE instances

First get the instance name via:

```
# gcloud compute instances list
```

Then ssh:

```
# gcloud compute ssh k8s-couchbase-server-node-1
```

## Setup Couchbase server cluster

```
root@k8s:~# container_1_private_ip=10.248.1.3; container_2_private_ip=10.248.2.4
root@k8s:~# docker run --entrypoint=/opt/couchbase/bin/couchbase-cli couchbase/server \
cluster-init -c $container_1_private_ip \
--cluster-init-username=Administrator \
--cluster-init-password=password \
--cluster-init-ramsize=600 \
-u admin -p password
```

** Create a bucket **

```
root@k8s:~# docker run --entrypoint=/opt/couchbase/bin/couchbase-cli couchbase/server \
bucket-create -c $container_1_private_ip:8091 \
--bucket=default \
--bucket-type=couchbase \
--bucket-port=11211 \
--bucket-ramsize=600 \
--bucket-replica=1 \
-u Administrator -p password
```

** Add second Couchbase server node + rebalance

```
root@k8s:~# docker run --entrypoint=/opt/couchbase/bin/couchbase-cli couchbase/server \
rebalance -c $container_1_private_ip \
-u Administrator -p password \
--server-add $container_2_private_ip \
--server-add-username Administrator \
--server-add-password password 
```

## Todo

* How can I use DNS hostnames instead of hardcoded private IPs for couchbase server nodes to see eachother?
    * Looks like these already get predefined
        * k8s-couchbase-server-node-1
	* k8s-couchbase-server-node-2
* Is it possible to expose Couchbase Server as a "service" to Sync Gateway?
* Replace individual pods with a Replication Controller


## References

* [Google cloud sdk](https://registry.hub.docker.com/u/google/cloud-sdk/)

* https://cloud.google.com/container-engine/docs/hello-wordpress

* https://cloud.google.com/container-engine/docs/guestbook
