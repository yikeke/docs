---
title: Deploy TiDB to Kubernetes on Google Cloud
summary: Use Ansible to deploy a TiDB cluster.
category: operations
---

# Deploy TiDB, a distributed MySQL compatible database, to Kubernetes on Google Cloud

[TiDB](www.pingcap.com) is a scale-out distributed MySQL database.
Scale-out means that when you have high usage of CPU, RAM, or disk, you just add another node to your cluster.
This is much easier to administer at scale. But we need deployment to be simple from day one. We have an [ansible based deployment]() that works in almost any environment. However, if we commit to deploying to Kubernetes, we can provide an even better experience.


## TIDB architecture

Lets review what we will be deploying.

* TiKV
* PD
* TIDB SQL

The scale-out property of TIDB is provided by [TikV](), a distributed key-value store.
The TiKV cluster itself requires deploying a PD cluster. PD stands for placement driver, which controls where data is stored in TiKV.
MySQL compatibility is provided by a separatly deployed TIDB SQL component that works on top of TiKV.


## Kubernetes architecture

TiKV and PD maintain state, and are thus mapped to a [StatefulSet]().
TiDB SQL is stateless and is mapped to a [Deployment]().

These are wrapped together in a helm chart called tidb-cluster.
Additionally, we provide [tidb-operator](), a Kubernetes operator. This program monitors the status of TiDB in your Kubernetes cluster and provides a gateway to administrative duties.


## Deploying Kubernetes on Google

TiDB can be deployed onto any Kubernetes cluster: you can bring up a three node cluster however you see fit and skip this section. But here we will bring up a cluster on Google Cloud designed for TiDB.
Google provides a free small VM called Google Cloud Shell that you can run this tutorial from.
Just click the button:

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/pingcap/docs)
<!--
[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/pingcap/docs&tutorial=op-guide/google-kubernetes-tutorial.md)
-->

If you have any issues with this button, you can download this file as markdown and open it.

	git clone https://github.com/pingcap/docs
	cd docs
	teachme op-guide/google-kubernetes-tutorial.md


### Bring up the Kubernetes cluster 

At the end of this cluster creation step, we will have a Kubernetes cluster with kubectl authorized to connect to it.
You can [use terraform]() or other tools to achieve this.

For more detailed information, please review the [Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart) instructions for setting up a Kubernetes cluster.

First create a project that this demo will be ran in.

	gcloud projects create tidb-demo

Before you can create a cluster, you must [enable Kubernetes for your project in the console.](https://console.cloud.google.com/projectselector/kubernetes?_ga=2.78459869.-833158988.1529036412)

Now set your gcloud to use this project:

	gcloud config set project tidb-demo

Set it to use a [zone](https://cloud.google.com/compute/docs/regions-zones/)

	gcloud config set compute/zone us-west1-a

Now create the kubernetes cluster.

	gcloud container clusters create tidb --num-nodes 0

This could take more than a minute to complete. Now is a good time to do some stretches and refill your beverage.

When the cluster is completed, default gcloud to use it.

	gcloud config set container/cluster tidb

Kubernetes clusters are operated by Google at no charge.
However, you will be charged for the instances that you provision.
For the purposes of reducing cost, we will use the g1-small instance type (cuts compute cost in half).
In a real deployment we use an n1-standard instance (or larger), which is required to utilize a local SSD volume rather than slower networked storage.
Many cloud provided Kubernetes offerings do not offer local SSD support, and the feature is in beta on GCP.
TiDB requires a minimum of 3 deployed instance to ensure data is replicated to 3 instances.

	gcloud container node-pools create tidb-small --machine-type g1-small


## Running TiDB

Now we have a cluster! Verify that kubectl can connect to it and that it has three machines running.

	kubectl get nodes

We can install TiDB with helm charts. Maske sure [helm is installed](https://github.com/helm/helm#install) on your platform.

We can get the TiDB helm charts from the source repository.

	git clone https://github.com/pingcap/tidb-operator
	cd tidb-operator

Helm will need a couple of permissions to work properly.

	kubectl create serviceaccount tiller --namespace kube-system
	kubectl create -f charts/lib/tiller-clusterrolebinding.yaml
	helm init --service-account tiller --upgrade

Now we can run the TiDB operator and the TiDB cluster

	helm install charts/tidb-operator -n tidb-operator --namespace=tidb-operator

We can watch the operator come up with

	watch kubectl get pods --namespace tidb-operator -l cluster.pingcap.com/tidbCluster=demo-cluster -o wide

Now with a single command we can bring-up a full TiDB cluster.

	helm install charts/tidb-cluster -n tidb --namespace=tidb

Now we can watch our cluster come up

	watch kubectl get pods --namespace tidb -l cluster.pingcap.com/tidbCluster=demo-cluster -o wide

Now lets connect to our MySQL database. This will connect from within the Kubernetes cluster.

	kubectl run -n tidb mysql-client --rm -i --tty --image mysql -- mysql -P 4000 -u root -h $(kubectl get svc demo-cluster-tidb -n tidb --output json | jq -r '.spe
c.clusterIP')

Now you are up and running with a distribute MySQL database!