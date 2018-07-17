---
title: "Using OSBA With Some CosmosDB Samples - Part Two"
date: 2018-05-08T10:10:05-06:00
draft: false
---

The exciting weeks got even more exciting! Both Build and Red Hat Summit are going on. In case you missed it, today we announced [managed OpenShift on Azure](https://azure.microsoft.com/en-us/blog/openshift-on-azure-the-easiest-fully-managed-openshift-in-the-cloud/). Over the last few weeks, we spent a little time adding provisioning parameters to OSBA in an effort to make the experience of using the broker with OpenShift really nice. Seeing this announcement while writing up this second look at some examples of CosmosDB with OSBA was extra nice! I've been mostly using Helm with these examples, but you could adapt all of them to use OpenShift templates as well. I'll tackle that in another post. You can also use the Service Catalog cli tool [svcat](https://github.com/kubernetes-incubator/service-catalog/blob/master/docs/install.md#installing-the-service-catalog-cli) pretty easily with OpenShift and Service Catalog too!

[Part one]({{< ref "post/some-osba-examples.md" >}}) of this blog series built on work from [simonpbennett](https://twitter.com/simonpbennett) and we ended up tweeting a little yesterday after I posted the blog. He was curious in the trade-offs / differences between using CosmosDB via the MongoDB API and via Spring Data for CosmosDB:

{{< tweet 993586835132628992 >}}

The answer to Simon's question really relates to _how_ you want to use the service. CosmosDB lets you interact with the service with several APIs: MongoDB, SQL API (the Spring library we'll talk about in this blog), Graph API, Table API and [Cassandra](https://docs.microsoft.com/en-us/azure/cosmos-db/cassandra-introduction), which is currently in preview. The example used in the last blog post uses the MongoDB API. This is a great way to work with CosmosDB if you're already familiar with MongoDB. The [SQL API](https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-introduction) approach gives you a RESTful API for interacting CosmosDB and allows you to do ad-hoc SQL like queries. This means you can build up SQL-like queries using the [REST API](https://docs.microsoft.com/en-us/rest/api/cosmos-db/query-documents). One notable difference I found in the user experience switching from the MongoDB API to SQL API is that creation of the `database` was transparent with the MongoDB API but required an explicit creation operation when using SQL API. The predecessor to OSBA, Meta Azure Service Broker, automatically created the database for you when you created the Database Account. OSBA did not do this until v0.11.0.

Interacting with CosmosDB with SQL API is done via the REST API mentioned above, but there are a number of libraries to make this easier. In Java, one option you can use is the [Azure DocumentDB libary](https://github.com/Azure/azure-documentdb-java). You can create a  `DocumentClient` and use methods like [queryCollections](https://docs.microsoft.com/en-us/java/api/com.microsoft.azure.documentdb._document_client.querycollections#com_microsoft_azure_documentdb__document_client_queryCollections_String_SqlQuerySpec_FeedOptions_) to work with the data. The Spring Data for CosmosDB library does much of the above for you by creating a `DocumentClient` and then allowing you to use normal [Spring Data query methods](https://docs.spring.io/spring-data/commons/docs/current/reference/html/#repositories.query-methods.details). Now it's extremely easy to use Spring Data for CosmosDB with an instance provisioned by OSBA or fall back through to the `DocumentClient`. When you provision via OSBA, you can directly start using that from your code. To do so, you need the following info:

```
azure.documentdb.uri=
azure.documentdb.key=
azure.documentdb.database=
```

The binding from OSBA will give you all of this information. This means that you can pretty easily deploy applications using the library to Kubernetes (Minikube, AKS, etc) or OpenShift and provision and bind to an instance with OSBA, especially when using Spring Data for CosmosDB!

To demonstrate this on Kubernetes using Service Catalog and OSBA, I've extracted the [sample application](https://github.com/Microsoft/spring-data-cosmosdb/tree/master/samplecode) into a new sample project. You can find it [here](https://github.com/jeremyrickard/spring-data-cosmosdb-example). The sample code is really more for standalone execution, so I needed to make some changes to containerize it.

The first thing I did in this code base was to add some environment variables to the `application.properties` file that was included.

```
$ cat src/main/resources/application.properties
azure.documentdb.uri=${COSMOS_DB_URI}
azure.documentdb.key=${COSMOS_DB_KEY}
azure.documentdb.database=${COSMOS_DB_DATABASE}
```

With those set, I containerized the sample code using Bitnami's Java container image. This is obviously required for running on Kubernetes. 

```
$ more Dockerfile
FROM bitnami/java:1.8-prod
RUN mkdir /app
COPY ./build/libs/cosmosdb-demo-0.0.1-SNAPSHOT.jar /app/cosmosdb-demo.jar
WORKDIR /app
EXPOSE 8080
CMD ["java","-jar","cosmosdb-demo.jar"]
```
I pushed that up to my dockerhub: 

```
docker build -t jeremyrickard/cosmosdb-demo:blog-post .
docker push jeremyrickard/cosmosdb-demo:blog-post
```

With the container in hand, I was ready to build a Helm chart to deploy the container and handle the interaction with OSBA. This sample currently will create a Deployment, a Service, a ServiceInstance and a ServiceBinding. This is pretty similar to the example we looked at in the last blog post, but the ServiceInstance and way we're consuming the secret generated by the ServiceBinding is a little different. 

First, let's look at the ServiceInstance template:

```
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceInstance
metadata:
  name: {{ .Values.cosmosdb.name }}
  labels:
    app: {{ template "fullname" . }}
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
spec:
  clusterServiceClassExternalName: azure-cosmosdb-sql
  clusterServicePlanExternalName: sql-api
  parameters:
    location: {{ .Values.cosmosdb.location }}
    resourceGroup: {{ .Values.cosmosdb.resourceGroup | default .Release.Namespace }}
    ipFilters:
      allowedIPRanges:
        - 0.0.0.0/0
      allowAccessFromAzure: enabled
```

If you compare this to the ServiceInstance we created for the previous example, we are using a different `ClusterServiceClass` and `ClusterServicePlan`. These specifically instruct OSBA to create a CosmosDB instance using the SQL API instead of MongoDB.

Next, in our container spec, we are using the following environment variables from the secret:

```
 env:
          - name: COSMOS_DB_URI
            valueFrom:
              secretKeyRef:
                name: {{ .Values.cosmosdb.name }}
                key: uri
          - name: COSMOS_DB_KEY
            valueFrom:
              secretKeyRef:
                name: {{ .Values.cosmosdb.name }}
                key: primaryKey
          - name: COSMOS_DB_DATABASE
            valueFrom:
              secretKeyRef:
                name: {{ .Values.cosmosdb.name }}
                key: databaseName
```

In the previous example, we used the connectionString (which we renamed to uri) alone. Here, we need the databaseName that we also created. These environment variables map back to the `application.properties` file up above.

Now when we install this chart, we end up with a brand new CosmosDB database account that uses the SQL API, along with a Database created inside of it! Once the instance and binding have been created, the container will startup and load a sample record into our new CosmosDB instance. Make sure you've installed Service Catalog and OSBA on your cluster! Check out the [Minikube](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-minikube.md) or [AKS](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-aks.md) quick starts if you haven't! Make sure you install with the `EXPERIMENTAL` mode so you can use the CosmosDB services!

```
helm install contrib/k8s/charts/spring-data-cosmos-sample -n spring-cosmos-example
```

Once that finishes, you'll have a new ServiceInstance, a ServiceBinding, Secret and a Pod that uses the Secret!

```
$ k get serviceinstances
NAME               AGE
cosmosdb           18h
```

```
$ k get servicebindings
NAME               AGE
cosmosdb           18h
```

```
$ k get secrets
NAME                          TYPE                                  DATA      AGE
cosmosdb                      Opaque                                8         18h
```

```
$ k get pods
NAME                                                              READY     STATUS    RESTARTS   AGE
spring-cosmos-example-spring-data-cosmos-sample-db8c65ccb-h2xdk   1/1       Running   0          18h
```

The most important part of this is the connection from that Pod to our ServiceInstance, via the Binding and the Secret!

```
Environment:
      COSMOS_DB_URI:       <set to the key 'uri' in secret 'cosmosdb'>           Optional: false
      COSMOS_DB_KEY:       <set to the key 'primaryKey' in secret 'cosmosdb'>    Optional: false
      COSMOS_DB_DATABASE:  <set to the key 'databaseName' in secret 'cosmosdb'>  Optional: false
```

Again, these are the environment variables used in that property file up above.

![We have the data](/images/more-comsos/default-data.png)

The sample code will also expose this via [Spring Data REST](https://projects.spring.io/spring-data-rest/)!

![Do A GET](/images/more-comsos/do-a-get.png)

Obviously, that is not all that interesting, so you can POST some data back.

![Do A POST](/images/more-comsos/post-to-service.png)

Now we have a little more data, let's check out the Azure Portal again and do a SQL like query on it. `SELECT * from c where c.ID = "cafebabe"`

![SQL Like Query](/images/more-comsos/sql-like-query.png)

Hopefully this shows how easy it is to deploy an app to Kubernetes and use CosmosDB via OSBA! In both cases, the wiring of the application up to the service is really easy via OSBA and doesn't really require any changes to your application! Environment variables, secrets and Service Catalog make all of this possible. The other thing I think makes this workflow really nice is the fact that I've done almost nothing that doesn't use Kubernetes tooling. I used Helm here, but for all of these I could use kubectl directly!