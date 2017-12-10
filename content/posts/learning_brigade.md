---
title: "Learning Brigade"
date: 2017-12-09T18:47:03-08:00
draft: true
---

One of the things I'm currently looking into in my new job is how we might use [Brigade](https://github.com/Azure/brigade) to handle our CI needs instead of CircleCI. I won't copy the Brigade README here, but at a high level Brigade is an event based scripting system that users Kubernetes primitives like pods and secrets. It provides out of the box support for GitHub events like pull requests and pushes and allows you to build workflows (i.e. CI pipelines) using fairly simple JavaScript. Within a Brigade event handler, you can define one or more Jobs and execute them in parallel or sequence. Jobs are Docker containers with an associated set of __tasks__. Brigade makes it fairly easy to execute your build/test processes against your repo because if you configure your Brigade project to include a source code repository, Brigade will automatically mount it into the Job container for you. You just need to write one ore more jobs and Brigade will execute them for you.

The Brigade team has also just released a tool called [Kashti](https://github.com/Azure/kashti) that provides a UI/dashbaord for your Brigade projects. Both Brigade and Kashti are _fairly_ new and you might encounter issues when using them so feel free to submit issues on GitHub if you run into any!

## Getting Started

Brigade is a tool for Kubernetes, so you'll obviously need a Kubernetes cluster to get started. In order to take advantage of GitHub webhooks, you'll need to expose your Brigade gateway to the internet. While it is possible to do that with Minikube, I'll be using an [AKS](https://azure.microsoft.com/en-us/services/container-service/) cluster for this. I already created the cluster and I am using it test out Brigade for our projects. If you have an Azure subscription, it's really easy to create an AKS cluster following this [walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough). You'll also want kubectl installed and have it configured to talk to your cluster.  

The easiest way to install both Brigade and Kashti is to use [Helm](https://github.com/kubernetes/helm). The Helm GitHub repo has binaries for most operating systems and you can install it by simply unpacking the binary and putting into your path. If you are using a Mac, it's also really easy to install with [brew](https://brew.sh/). With brew, you just need to:

```
$ brew install kubernetes-helm
```
Once installed, run `helm init` to deploy the helm components into your cluster.

Note: I'm not using RBAC here. If you're using RBAC, you'll need to create a service account for the Helm component `tiller` to use. If you'd like to do RBAC with Helm, refer to [the using Helm](https://github.com/kubernetes/helm/blob/master/docs/using_helm.md) document. If you are using RBAC, you will also need to create some RBAC configuration for the service account used by Brigade. I'll address that later. 

In order to make the most out of using Brigade, you should also install Go so you can build the *_brig_* command line tool. Building *_brig_* is pretty simple once you have Go installed, clone the Brigade repo and setup your GOPATH like this and use the provided Makefile.

```console
$ cd ~/projects
$ mkdir -p brigade/src/github.com/Azure
$ cd brigade/src/github.com/Azure
$ git clone https://github.com/Azure/brigade.git
$ export GOPATH=~/projects/brigade/
$ export GOBIN=$GOPATH/bin
$ export PATH=$GOBIN:$PATH
$ make bootstrap brig
```

If this was successful, you'll get a *bin* diretory with the brig tool inside of it. I moved this to a directory that lives in my path so I can easily use it later. 

You'll also want to clone the Kashti repo as the Helm charts are currently located within the GitHub repo. 


## Installing Brigade and Kashti

I'll use Helm to install both Brigade and Kashti. The Brigade install will create a couple of Kubernetes components:

The Brigade chart will create the following kuberentes *services*:

* Brigade API Service
* Brigade Controller Service
* Brigade Gateway Service

By default, the Gateway and the Controller will be LoadBalancer services and exposed to the internet if you're running on a kubernetes cluster that has a cloud provider integration turned on. 


In addition to these services, the Brigade chart will also create the following kubernetes *deployments*:

You can view these with helm or kubectl. Once you start triggering workflow, you'll also notice that worker pods are created along with pods for each Job you define.



