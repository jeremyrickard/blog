---
title: "Using OSBA With Some CosmosDB Samples - Part One"
date: 2018-05-07T10:53:15-06:00
draft: false
---

It's been a pretty exciting few weeks! I just returned from Kubecon EU (watch my [talk](https://www.youtube.com/watch?v=SuicCBXJRPg)!!) and we've been hard at work on Open Service Broker for Azure and Service Catalog! We recently released [v0.11.0](https://github.com/Azure/open-service-broker-azure/releases/tag/v0.11.0) of OSBA that brings along some cool new things, like alignment with the new pricing tiers for MySQL and PostgreSQL and a couple of great enhancements to CosmosDB!

I think the one that I'm most excited about in the CosmosDB service is that you can now automatically create both a database AND a database account directly from Kubernetes of Cloud Foundry. This has worked with CosmosDB: MongoDB API for some time, but if you were using the SQL API we always stopped a little short of making the service usable with minimal effort! That has changed in our latest release. While we were hard at work on this, the [Spring Data for Azure Cosmos DB](https://github.com/Microsoft/spring-data-cosmosdb) library has seen some updates and a new sample app has been included. There was also a great blog post on the [Microsoft + Open Source blog](https://open.microsoft.com/2018/03/28/deploy-java-application-azure-kubernetes-service-cosmos-db/) from our friends at Bitnami about deploying a Java application on AKS with CosmosDB: MongoDB API. As we don't have a good CosmosDB example in our [Azure/helm-charts](https://github.com/Azure/helm-charts) repo, I thought this would be a great time so show how easy it is to use OSBA with both of these examples! I'll do this in a series of blog posts in order to make these a little more easy to digest. 

Let's start with the [example](https://github.com/nomisbeme/customerapp) from [Simon Bennett](https://twitter.com/simonpbennett) at Bitnami!

This example is great and walks you through an AKS cluster, installing helm and doing ingress and using SSL with [kube-lego](https://github.com/jetstack/kube-lego). If you follow along, you get a pretty great experience using Kubernetes tooling once you have the cluster up and running, until you need to create the database. Using the `az` cli or the portal is sort of unavoidable for creation of the AKS cluster, but we can replace the manual steps of the creation of the Azure CosmosDB database and the creation of the secret with a pretty small addition to Simon's example! For this, we'll use my fork of his example app, which you can find [here](https://github.com/jeremyrickard/customerapp).

First, you'll want to install Service Catalog and OSBA. You can do that by following our [quickstart](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-aks.md), but the TL;DR is:

First, setup some environment variables with your Azure account info:

```
az account list -o table
export AZURE_SUBSCRIPTION_ID="<SubscriptionId>"
az ad sp create-for-rbac --name osba-quickstart -o table
export AZURE_TENANT_ID=<Tenant>
export AZURE_CLIENT_ID=<AppId>
export AZURE_CLIENT_SECRET=<Password>
```

Next, install Service Catalog. Simon already has you install Helm, so....

```
helm repo add svc-cat https://svc-catalog-charts.storage.googleapis.com
helm install svc-cat/catalog --name catalog --namespace catalog \
   --set rbacEnable=false \
   --set apiserver.storage.etcd.persistence.enabled=true
```
This might take a minute or two, but once it's done you just need to install OSBA:

```
helm repo add azure https://kubernetescharts.blob.core.windows.net/azure
helm install azure/open-service-broker-azure --name osba --namespace osba \
  --set azure.subscriptionId=$AZURE_SUBSCRIPTION_ID \
  --set azure.tenantId=$AZURE_TENANT_ID \
  --set azure.clientId=$AZURE_CLIENT_ID \
  --set azure.clientSecret=$AZURE_CLIENT_SECRET \
  --set modules.minStability=EXPERIMENTAL
```

Check the status of everything and pause here until everything is up and running.

```
$ kubectl get pods --namespace catalog
NAME                                                     READY     STATUS    RESTARTS   AGE
po/catalog-catalog-apiserver-5999465555-9hgwm            2/2       Running   4          9d
po/catalog-catalog-controller-manager-554c758786-f8qvc   1/1       Running   11         9d

$ kubectl get pods --namespace osba
NAME                                           READY     STATUS    RESTARTS   AGE
po/osba-azure-service-broker-8495bff484-7ggj6   1/1       Running   0          9d
po/osba-redis-5b44fc9779-hgnck                  1/1       Running   0          9d
```

Once everything is happy, you're good to go!

We don't actually need to make *ANY* code changes to make this example work, we only need to enhance the Helm chart to also create a service instance and create a service binding for that instance. I also added an attribute to the values.yaml to support those new templates.

We'll take advantage of the new secret transforms capability from Service Catalog in order to shape the binding secret into the form the helm chart was already expecting! Simon's docker container is already published, so you can use it as is or build it yourself. I'll build it myself, just to be safe.

```
 mvn clean package
docker build -t jeremyrickard/customerapp:0.1.2 .
docker push jeremyrickard/customerapp:0.1.2
helm install helm/customerapp/ -n capp --set image.repository=jeremyrickard/customerapp --set image.tag=0.1.2
```

When you run this, you'll see a couple of things that are a little different than if you just followed Simon's example as is.

The first difference is in the helm install output. We've created two additional resources here, the _ServiceInstance_ and the _ServiceBinding_. The _ServiceInstance_ is that will instruct Service Catalog to actually create a new CosmosDB instance in Azure! The _ServiceBinding_ will create a secret with all the connection info needed.

```
==> v1beta1/ServiceBinding
NAME                               AGE
capp-customerapp-cosmosdb-binding  1s

==> v1beta1/ServiceInstance
mongodb  1s
```

The other thing you'll notice is that the pods for this deployment go into a failure state:

```
$ k get pods
NAME                                                        READY     STATUS                       RESTARTS   AGE
capp-customerapp-56dd4f7d44-hbzm6                           0/1       CreateContainerConfigError   0          3m
capp-customerapp-56dd4f7d44-vlj8p                           0/1       CreateContainerConfigError   0          3m
capp-customerapp-56dd4f7d44-z88w6                           0/1       CreateContainerConfigError   0          3m
```

This is because we first need to create the CosmosDB instance AND the binding  before the pods will start up. The pods use a secret that will be generated as part of that process, so they won't start until those are both complete. After a bit of time, they should start up and be happy. You'll notice that a secret was created for you with all the connection information as a result of the binding. You no longer need to manually create that as in Simon's example.

The secret has some extra info in it, because OSBA returns additional information to let you use the connection info in a flexible way. We applied a transform to that data though, so we should have the `MONGODB_URI` key that Simon's example was expecting:

```
$ k get secret customerdatabasesecret -o yaml
apiVersion: v1
data:
  MONGODB_URI: <redacted>
  host: <redacted>
  password: <redacted>
  port: MTAyNTU=
  uri: <redacted>
  username: <redacted>
kind: Secret
metadata:
  creationTimestamp: 2018-05-07T18:31:47Z
  name: customerdatabasesecret
  namespace: tweets
  ownerReferences:
  - apiVersion: servicecatalog.k8s.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: ServiceBinding
    name: capp-customerapp-cosmosdb-binding
    uid: 1262c90b-5224-11e8-8368-0a580af4019a
  resourceVersion: "1970008"
  selfLink: /api/v1/namespaces/tweets/secrets/customerdatabasesecret
```

As you can see from the secret, it was created by Service Catalog! You can now launch the app and it just works!

![Sample App](/images/cosmos-sample-apps/sample-app.png)
![Portal View](/images/cosmos-sample-apps/portal-view.png)

The main difference between this example and the Spring Data for Azure CosmosDB example is the choice of API. This one uses MongoDB and in this case, all that is REALLY needed is the database account. Using a MongoDB client with the database account would automagically create both the collection and database inside the account. That's a little different than the behavior when using the SQL API with CosmosDB. Until v0.11.0, OSBA wouldn't do this for you automatically. In part two of this series, I'll show you how that works.
