---
title: "Using Open Service Broker for Azure with PCF Dev"
date: 2018-02-07T08:23:38-07:00
draft: false
---

**Update**: I originally incorrectly stated that the Azure Spring Boot library didn't work with the VCAP_SERVICES populated by CF when using Open Service Broker for Azure. It does indeed work for a select number of services, such as DocumentDB. I've updated the blog post to reflect this.  

I've had the good fortune of working with a great team on [Open Service Broker for Azure](https://github.com/Azure/open-service-broker-azure) for the last few months. What's Open Service Broker for Azure (OSBA!) you ask? It's an [Open Service Broker API](https://www.openservicebrokerapi.org/) compliant service broker that enables you to easily provision and bind to services like [MySQL](https://azure.microsoft.com/en-us/services/mysql/), [SQLDB](https://azure.microsoft.com/en-us/services/sql-database/) and [CosmosDB](https://azure.microsoft.com/en-us/services/cosmos-db/) in Azure.  I was familiar with service brokers from working with [Cloud Foundry](https://github.com/cloudfoundry) in the past, so I was really excited to join the Azure team and get a chance to work on bringing the great exeperience developers get using service brokers with CF to Kubernetes with [Service Catalog](https://github.com/kubernetes-incubator/service-catalog/blob/master/docs/design.md)! While I'm most excited about the things we can bring to the Kubernetes community, I'm also excited that we can continue to provide value to the CF community and provide easy access to the tons of services available in Azure. 

If you look in our git repo, you'll see we're obviously very focused on supporting Kubernetes and Service Catalog with OSBA. To that end, we've produced a couple of nice quickstart guides for [Minikube](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-minikube.md) and now [Azure Container Service (AKS)](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-aks.md) that will help you get going pretty quickly. We've also got some cool examples in our [charts](https://github.com/Azure/helm-charts) repo that illustrate how to modify some popular Helm charts like WordPress and Drupal to work with Azure managed services using OSBA and Service Catalog. I think the MiniKube quickstart is especially great because you can start working with Azure services almost immediately using Service Catalog if you have an Azure account. I wondered how easy it would be to do that with PCF Dev. 


To start out, I decided I would work with CosmosDB. CosmosDB lets you use a MongoDB client and treat it like a normal MongoDB instance. Not _all_ MongoDB features are supported in the [CosmosDB MongoDB API support](https://docs.microsoft.com/en-us/azure/cosmos-db/mongodb-feature-support), but it should be sufficient for this. Next, I grabbed the great [Spring Music](https://github.com/cloudfoundry-samples/spring-music) sample application from [Scott Frederick](https://github.com/scottfrederick) and Cloud Foundry. It already works well with MongoDB, but there ~~isn't currently a [Spring Cloud Connector](https://cloud.spring.io/spring-cloud-connectors/) that works with the Azure services via OSBA.~~ the Azure Spring Boot library doesn't handle VCAP_SERVICES for all the services that OSBA can provision. I modified it to work with OSBA by adding some Spring Configuration objects that will create a `MongoDbFactory` using the data provided in the VCAP_SERVICES block once you bind to our CosmosDB service. Other than that, it's pretty much the stock Spring Music project. Now, let's find out how OSBA works with PCF Dev! 

First, you'll need a couple of things to get going:

* A [Microsoft Azure account](https://azure.microsoft.com/en-us/free/).
* The [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* A [git client](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [PCF Dev](https://docs.pivotal.io/pcf-dev/)
* [curl](https://curl.haxx.se/)
* [Java](http://www.oracle.com/technetwork/java/javase/downloads/index.html)

Once those are taken care of, let's setup our environment to use the Azure account. 

Start by running `az login` and follow the instructions in the command output to authorize `az` to use your account

Next, list your Azure subscriptions:

```console
az account list -o table
```

Once you've decided which to use, copy your subscription ID and save it in an environment variable:

```console
export AZURE_SUBSCRIPTION_ID="<SubscriptionId>"
```


Now, you need to create a service principle for OSBA to use and export the results for later use

```console
az ad sp create-for-rbac --name osba-quickstart -o table
```

This will give you a bunch of out put for the newly created service principle. You'll need to have these later, so make them into environment variables. OSBA will use them to create resoruces in your Azure account. You'll want to match the values in the table output to the export statements below.

```console
export AZURE_TENANT_ID=<Tenant>
export AZURE_CLIENT_ID=<AppId>
export AZURE_CLIENT_SECRET=<Password>
```

Finally, create a resource group to contain the resources you'll create with OSBA. If you don't provide one, OSBA will make one up for you, but this is much easier to manage later on. 

```console
az group create --name cf-osba --location eastus
```

At this point, everything related to our Azure account is done. Next, let's setup up PCF Dev. If you haven't already installed it, [do that](https://pivotal.io/platform/pcf-tutorials/getting-started-with-pivotal-cloud-foundry-dev/install-pcf-dev) now. You should be able to start it with the default options. 

```console
cf dev start
```

Note: This may take some time to start. If you want to provide any arguments you'll, need to make sure that you start the built in redis broker (`-s redis`). We'll be making use of that for OSBA. 

Next, login to your instance as admin. When you start PCF Dev up, it should inform you of the credentials to use and the login endpoint. I chose to use the pcfdev-org for this. 


```console
cf login -a https://api.local.pcfdev.io --skip-ssl-validation
```

Next, we will need to provision a Redis instance. OSBA uses Redis for both storage and for managing asynchronous operations. PCF Dev includes a service broker that will provision a local Redis instance. You can see the available brokers with the `cf service-brokers` command. Once we register OSBA later, it will also show up here!

```console
$ cf service-brokers
Getting service brokers as admin...

name           url
local-volume   http://localbroker.local.pcfdev.io
p-mysql        http://mysql-broker.local.pcfdev.io
p-rabbitmq     http://rabbitmq-broker.local.pcfdev.io
p-redis        http://redis-broker.local.pcfdev.io
```

For local use, the shared Redis will work great. For produciton use, you'd obviously want to use something else (our official instructions say to use an [Azure Redis Cache](https://azure.microsoft.com/en-us/services/cache/)).  To provision a redis instance, use the cf cli. You'll need the plan name as well, so the `cf marketplace` command will show you what's available.

```console
$ cf marketplace
Getting services from marketplace in org pcfdev-org / space pcfdev-space as admin...
OK

service        plans             description
local-volume   free-local-disk   Local service docs: https://github.com/cloudfoundry-incubator/local-volume-release/
p-mysql        512mb, 1gb        MySQL databases on demand
p-rabbitmq     standard          RabbitMQ is a robust and scalable high-performance multi-protocol messaging broker.
p-redis        shared-vm         Redis service to provide a key-value store
```

Let's use the cf cli to provision it now.

```console
cf create-service p-redis shared-vm redis
```

This will creaet a new service instance named Redis:

```console
$ cf service redis
Showing info of service redis in org pcfdev-org / space pcfdev-space as admin...

name:            redis
service:         p-redis
bound apps:
tags:
plan:            shared-vm
description:     Redis service to provide a key-value store
documentation:
dashboard:

Showing status of last operation from service redis...

status:    create succeeded
message:
started:   2018-02-07T16:43:36Z
updated:   2018-02-07T16:43:36Z
```

Now, we're ready to install OSBA! There are a couple of options for running OSBA, but we'll run it in PCF Dev for this walk through. We'll use the CF go build pack to do that!  Note: While we do publish OSBA in Docker form and that's what we use with Kubernetes, our current image isn't compatible with Cloud Foundry's 
Docker runtime. Instead, we'll use the go build pack and build from the OSBA repo. 

First clone the repo: 

```console
git clone https://github.com/Azure/open-service-broker-azure.git
```

OSBA is under pretty active development, so let's also checkout a specific tag to ensure we have a known version that works with this blog post.

```console
git checkout v0.8.0-alpha
```


In the repo, we provide a CF manifest file under the `contrib` directory. When OSBA runs, it expects to get configuration via environment variables. You can review these in the manifest:

```yaml
---
applications:
- name: osba
  buildpack: https://github.com/cloudfoundry/go-buildpack/releases/download/v1.8.13/go-buildpack-v1.8.13.zip
  command: broker
  env:
    AZURE_SUBSCRIPTION_ID: <YOUR SUBSCRIPTION ID>
    AZURE_TENANT_ID: <YOUR TENANT ID>
    AZURE_CLIENT_ID: <APPID FROM SERVICE PRINCIPAL>
    AZURE_CLIENT_SECRET: <PASSWORD FROM SERVICE PRINCIPAL>
    AZURE_DEFAULT_RESOURCE_GROUP: <DEFAULT AZURE RESOURCE GROUP FOR SERVICES>
    AZURE_DEFAULT_LOCATION: <DEFAULT AZURE REGION FOR SERVICES>
    LOG_LEVEL: DEBUG
    REDIS_HOST: <HOSTNAME FROM AZURE REDIS CACHE>
    REDIS_PASSWORD: <PRIMARYKEY FROM AZURE REDIS CACHE>
    REDIS_PORT: 6380
    REDIS_ENABLE_TLS: true
    AES256_KEY: AES256Key-32Characters1234567890
    BASIC_AUTH_USERNAME: username
    BASIC_AUTH_PASSWORD: password
    GOPACKAGENAME: github.com/Azure/open-service-broker-azure
    GO_INSTALL_PACKAGE_SPEC: github.com/Azure/open-service-broker-azure/cmd/broker
```

Some of these you already have thanks to the account setup we did above. The Redis information you'll need to get by binding OSBA to the Redis service we created above. Once we have those, we'll use `cf set-env` to change the environment. You can modify this file and replace the values, but we'll use the `cf set-env` command to do that in a bit. Note that if you were deploying this for production use, you'd want to change things like the AES256_KEY and the basic auth credentials. 

First, let's push OSBA. 

```console
cd open-service-broker-azure
cf push -f contrib/cf/manifest.yml
```

This will fail because we have not provided the configuration yet, but that's ok. We'll restart it in a bit. 

Next, bind OSBA to Redis using `cf bind-service`. OSBA isn't going to directly be able to use the VCAP_SERVICES environment that this will generate, so we're just going to obtain the Redis credentials from it. 

```console
cf bind-service osba redis
```

This will bind OSBA to Redis, but as mentioned OSBA doesn't handle VCAP_SERVICES directly. So use the `cf env osba` command to view it's new environment.

```console
$ cf env osba
Getting env variables for app osba in org pcfdev-org / space pcfdev-space as admin...
OK

System-Provided:
{
 "VCAP_SERVICES": {
  "p-redis": [
   {
    "credentials": {
     "host": "redis-broker.local.pcfdev.io",
     "password": "739f2a20-ca7b-4319-a5ea-53b9fdaa4c43",
     "port": 36964
    },
    "label": "p-redis",
    "name": "redis",
    "plan": "shared-vm",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "pivotal",
     "redis"
    ],
    "volume_mounts": []
   }
  ]
 }
}
```

The host, password and port are what we'll use to configure OSBA now. We'll set a number of environment variables now using the `cf set-env` command.

```console
cf set-env osba AZURE_SUBSCRIPTION_ID $AZURE_SUBSCRIPTION_ID
cf set-env osba AZURE_TENANT_ID $AZURE_TENANT_ID
cf set-env osba AZURE_CLIENT_ID $AZURE_CLIENT_ID
cf set-env osba AZURE_CLIENT_SECRET $AZURE_CLIENT_SECRET
cf set-env osba AZURE_DEFAULT_RESOURCE_GROUP cf-osba
cf set-env osba AZURE_DEFAULT_LOCATION eastus
cf set-env osba REDIS_ENABLE_TLS false
cf set-env osba REDIS_HOST redis-broker.local.pcfdev.io
cf set-env osba REDIS_PORT 36964
cf set-env osba REDIS_PASSWORD 739f2a20-ca7b-4319-a5ea-53b9fdaa4c43
```

The first set of environment variables are the things we obtained from setting up our Azure account. Next we disable TLS for the redis instance since the PCF Dev provisioned Redis doesn't support it. Finally we provide connection info for Redis that was obtained from the `cf env` command above. Now we can restart the app. 

```console
cf restart osba
```

This should eventually give you output that indicates the broker is now running!

```console
Restarting app osba in org pcfdev-org / space pcfdev-space as admin...

Stopping app...

Waiting for app to start...

name:              osba
requested state:   started
instances:         1/1
usage:             256M x 1 instances
routes:            osba.local.pcfdev.io
last uploaded:     Thu 07 Feb 10:57:51 MST 2018
stack:             cflinuxfs2
buildpack:         https://github.com/cloudfoundry/go-buildpack/releases/download/v1.8.13/go-buildpack-v1.8.13.zip
start command:     broker

     state     since                  cpu    memory          disk            details
#0   running   2018-02-07T18:05:34Z   0.0%   11.5M of 256M   72.1M of 512M
```

We can verify the broker is up and running with `cf logs`

```console
cf logs --recent osba
```

You should see messages toward the end indicating that the broker is health. We can check that the broker is up and running with Curl before moving on to register it with the local PCF Dev instance.

```console
curl -u username:password -H "X-Broker-API-Version: 2.13" http://osba.local.pcfdev.io/v2/catalog
```

This should give you a long JSON response with the OSBA catalog of services. Now we can registger it with PCF Dev as a new service broker.

```console
cf create-service-broker osba username password http://osba.local.pcfdev.io
```

This uses the default username and password provided to the broker by the manifest.  You can verify it's there now by checking `cf service-brokers` again.

```console
$cf service-brokers
Getting service brokers as admin...

name           url
local-volume   http://localbroker.local.pcfdev.io
osba           http://osba.local.pcfdev.io
p-mysql        http://mysql-broker.local.pcfdev.io
p-rabbitmq     http://rabbitmq-broker.local.pcfdev.io
p-redis        http://redis-broker.local.pcfdev.io
```

Great! OSBA is registerd now. But the Azure services won't be available in the Marketplace yet since by default new brokers are private. To make the services available, you need to use the `cf enable-service-access` command. There are a lot of great services available via OSBA and you can view them with the `cf service-access -b osba` command.

```console
$ cf service-access -b osba
Getting service access for broker osba as admin...
broker: osba
   service                    plan                              access   orgs
   azure-postgresqldb         basic50                           none
   azure-postgresqldb         basic100                          none
   azure-rediscache           basic                             none
   azure-rediscache           standard                          none
   azure-rediscache           premium                           none
   azure-mysqldb              basic50                           none
   azure-mysqldb              basic100                          none
   azure-mysqldb              standard100                       none
   azure-mysqldb              standard200                       none
   azure-mysqldb              standard400                       none
   azure-mysqldb              standard800                       none
   azure-servicebus           basic                             none
   azure-servicebus           standard                          none
   azure-servicebus           premium                           none
   azure-eventhubs            basic                             none
   azure-eventhubs            standard                          none
   azure-keyvault             standard                          none
   azure-keyvault             premium                           none
   azure-sqldb                basic                             none
   azure-sqldb                standard-s0                       none
   azure-sqldb                standard-s1                       none
   azure-sqldb                standard-s2                       none
   azure-sqldb                standard-s3                       none
   azure-sqldb                premium-p1                        none
   azure-sqldb                premium-p2                        none
   azure-sqldb                premium-p4                        none
   azure-sqldb                premium-p6                        none
   azure-sqldb                premium-p11                       none
   azure-sqldb                data-warehouse-100                none
   azure-sqldb                data-warehouse-1200               none
   azure-sqldb-vm-only        sqldb-vm-only                     none
   azure-sqldb-db-only        basic                             none
   azure-sqldb-db-only        standard-s0                       none
   azure-sqldb-db-only        standard-s1                       none
   azure-sqldb-db-only        standard-s2                       none
   azure-sqldb-db-only        standard-s3                       none
   azure-sqldb-db-only        premium-p1                        none
   azure-sqldb-db-only        premium-p2                        none
   azure-sqldb-db-only        premium-p4                        none
   azure-sqldb-db-only        premium-p6                        none
   azure-sqldb-db-only        premium-p11                       none
   azure-sqldb-db-only        data-warehouse-100                none
   azure-sqldb-db-only        data-warehouse-1200               none
   azure-cosmos-document-db   document-db                       none
   azure-cosmos-mongo-db      mongo-db                          none
   azure-storage              general-purpose-storage-account   none
   azure-storage              blob-storage-account              none
   azure-storage              blob-container                    none
   azuresearch                free                              none
   azuresearch                basic                             none
   azuresearch                standard-s1                       none
   azure-aci                  aci                               none
 ```


 We'll be deploying an Azure CosmosDB Mongo instance and using the MongoDB API with it. To do that, enable the service and plan with the following command.

```console
cf enable-service-access azure-cosmos-mongo-db
```

Now we're ready to provision an instance. You don't need to provide any parameters to the CosmosDB provision request since we've already set the resource group and location, so you can simply run this command.

```console
cf create-service azure-cosmos-mongo-db mongo-db cosmosdb
```

We've named the service instance `cosmosdb` so we can easily refer to it later after we bind our sample applciation to it. The provision will take few minutes and will be done asynchronously. You can view the status with the `cf services` command. 

```console
$ cf services
Getting services in org pcfdev-org / space pcfdev-space as admin...
OK

name       service                 plan        bound apps   last operation
cosmosdb   azure-cosmos-mongo-db   mongo-db                 create in progress
redis      p-redis                 shared-vm   osba         create succeeded
```

When the provision is finished, you're ready to move on to deploying our sample application. You can get it by cloning my spring-music repo.

```console
git clone https://github.com/jeremyrickard/spring-music.git
```

Once you've cloned the repo it's as simple as building the application and doing a `cf push`. As I previously mentioned, the existing Cloud Connectors don't work with CosmosDB. I'm planning on ~~developing an Azure Cloud Connectors library~~ submitting a PR to add support for more Azure services to go along with the cool stuff going on in the Microsoft [azure-spring-boot](https://github.com/Microsoft/azure-spring-boot) project. To make things easier in the context of this walkthrough, I've already modified the manifest to bind to our cosmosdb service and select the cosmosdb profile. 

```console
./gradlew clean assemble
cf push
```

Once this finishes, you should have a copy of Srping Music running with CosmosDB backing the service. In the output of the `cf push`, you should see a url exposed to access the app. Mine was `spring-music-interadditive-penguin.local.pcfdev.io`. Use that URL in your browser of choice, and you should be greated with the Spring Music app!

![Not My Music Choices](/images/cf-osba/spring-music.png)

The Azure Portal will also have your newly created CosmosDB instance. You can use it to dig around in the new database as well.


![Portal](/images/cf-osba/portal.png)


When you're done, you'll probably want to clean this up. Use `cf` to remove the application and deprovision CosmosDB.

```console
cf delete spring-music
cf delete-service cosmosdb
```

The deprovision operation will also take a bit of time to complete. If the deprovision fails for whatever reason, you can use the Azure portal or CLI to remove the resources and the resoruce group. 
