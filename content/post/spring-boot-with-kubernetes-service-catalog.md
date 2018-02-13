---
title: "Using Spring Boot With Kubernetes Service Catalog"
date: 2018-02-11T20:29:20-07:00
draft: false 
---


In my last couple blog posts, I've talked about [Service Catalog](https://github.com/kubernetes-incubator/service-catalog), the [Open Service Broker API Specification](https://github.com/openservicebrokerapi/servicebroker) and [Open Service Broker for Azure](https://github.com/Azure/open-service-broker-azure), a.k.a. OSBA. You might ask yourself why those matter. I think [Aaron](https://twitter.com/arschles) did a great job of explaining [why](https://medium.com/@arschles/service-brokers-are-your-apps-best-friend-5de9e100404d) you should care about service brokers, so I won't really get into that. But my firm belief is that you should care about brokers and I think you will. I also think that there need to be some better abstractions to make you _really_ want brokers. I think this twitter exchange between Gabe and Kelsey is sort of along those lines (open it up so you can read all the replies).

{{< tweet 849370912591933440 >}}

There will eventually be some higher level abstractions for what an application looks like, and part of that is going to be how you provision and connect to other services, especially things like data stores. Stateful data is going to make you want a service broker to handle that for you on all your clusters. In my last role, our application was largely based on [Xenon](https://github.com/vmware/xenon) and as a side effect of that, our implementation resulted in stateful containers. In my opinion, this lead to a lot of complexity in terms of operation of our services that we didn't really need. Using a Stateful set worked out pretty well, but our bad architectural decision got in the way. I found this quote from Brendan Burns (by way of James Watters) to summarize what I think we experienced pretty well.

{{< tweet 962746726779011072 >}}


In the end, we decided to shift architecturally and ended up gravitating toward PostgreSQL. We also ended up (at least for the interim) consuming a managed version via AWS. But we ended up falling into a variant of what Aaron calls the "post-it note" workflow. We built automation to use Terraform to provision the DB, then evolved the automation to populate Vault with the required info to connect. We essentially _built_ a broker and parts of service catalog. Part of that was the relative immaturity of Service Catalog at the time, but AWS also didn’t have a really robust broker at the time. They do have [one now](https://github.com/awslabs/aws-servicebroker-documentation/wiki) based on RedHat’s [Ansible Broker](https://github.com/openshift/ansible-service-broker). I think developers will expect all the major cloud providers to provide a broker. As people move to managed offerings like AKS and GKE, they will want easier access to managed services. This won't be everyone and there is nothing wrong with running those stateful services on cluster, but managed services exist for a reason. Eventually it will become table stakes for any of the managed Kubernetes providers. 

In Cloud Foundry, this easy avenue to service consumption via brokers is a fundamental part of the platform. _Using_ services via CF is such a fundamental thing that [Spring](https://spring.io/) has extensive support to make this _just work_. The developer experience is great. To be fair, Kubernetes is _not_ a PaaS like CF so the comparison isn't entirely fair. But there are a lot of things we can learn from CF and build in the Kubernetes space. Let's take a look at how we might build a [Spring Boot](https://projects.spring.io/spring-boot/) application and deploy it to Kubernetes and use Service Catalog in a similar way. If you want to follow along with this, you'll need a Kubernetes cluster with Service Catalog and a service broker (if you want to deploy my sample app, you'll need OSBA as well) installed. You can grab my sample application from [github](https://github.com/jeremyrickard/svc-cat-documentdb). You can use our [Minikube](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-minikube.md) or [AKS](https://github.com/Azure/open-service-broker-azure/blob/master/docs/quickstart-aks.md) quickstarts to get a cluster with Service Catalog and OSBA up and running.  

First some context. When you build an application with Spring Boot, you get a lot for free. You can combine various [spring-boot-starters](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-project/spring-boot-starters) and get a lot of functionality out of the box. When you deploy to CF and declare a binding to a service, in keeping with the [12 Factor App](https://12factor.net/) methodology you'll end up with an environment variable added to your application called VCAP_SERVICES. This will contain everything you need to connect to the service in question. Using Spring Boot and CF means you can take advantage of [Spring Cloud Connectors](https://cloud.spring.io/spring-cloud-connectors/) to automatically [parse](http://engineering.pivotal.io/post/spring-boot-injecting-credentials/) these things. As an application developer, it more or less _just works_. 

When using Service Catalog with Spring, the experience isn’t quite as straightforward. There are more steps and the out of the box experience via kubectl is a little lacking. When you create a binding with Service Catalog, you'll get a secret that contains the same information as what you'd end up with in VCAP_SERVICES. It's then up to you as the application developer to wire this up. Provisioning a service and making a binding can  make your app deployment a bit more complicated. I think CF makes all this a little easier, so let’s compare what we have today with Kubernetes and Service Catalog. For the purposes of this comparison, I've developed a _very_ basic Spring Boot application using the [azure-spring-boot](https://github.com/Microsoft/azure-spring-boot) starter and DocumentDB, provisioned with Service Catalog and OSBA. 

If you take a look at the [how to](https://docs.microsoft.com/en-us/java/azure/spring-framework/configure-spring-boot-starter-java-app-with-cosmos-db) for using the Azure spring boot starter, you'll see that it requires a couple of properties:

```
azure.documentdb.uri=your-documentdb-uri
azure.documentdb.key=your-documentdb-key
azure.documentdb.database=your-documentdb-databasename
```

The how-to documentation is showing you exactly the post-it note workflow! Luckily with Spring, you can provide those things as environment variables as well. The equivalent environment variables would be:

```
AZURE_DOCUMENTDB_URI=your-documentdb-uri
AZURE_DOCUMENTDB_KEY=your-documentdb-key
AZURE_DOCUMENTDB_DATABASE=your-documentdb-databasename
```

With this in mind, we can get pretty close to the CF experience, but first we need to provision the service and create a binding for our application to use.

Starting with making the service instance, we really have _two_ options here. The first is to define the YAML for what that service instance might look like and create it with `kubectl` or the API. I'll get to the other way in a moment. The YAML would look something like this.

```yaml
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceInstance
metadata:
  name: <INSTANCE_NAME>
spec:
  clusterServiceClassExternalName: <CLASS>
  clusterServicePlanExternalName: <PLAN>
  parameters:
    <KEY :  VALUE>
```
After you created that, you'd also need to create one for the binding. Your second option is a domain specific CLI called [svcat](https://github.com/kubernetes-incubator/service-catalog/tree/master/cmd/svcat). svcat brings a lot of that user experience from CF to service catalog. To get the plans and classes, you can simply run `svcat get plans`. Here is some sample output from svcat with Service Catalog configured with OSBA. 

```console
$ svcat get plans
               NAME                          CLASS                      DESCRIPTION                             UUID
+---------------------------------+--------------------------+--------------------------------+--------------------------------------+
premium                           azure-rediscache           Premium Tier, 6GB Cache          b1057a8f-9a01-423a-bc35-e168d5c04cf0
  basic                             azure-rediscache           Basic Tier, 250MB Cache          362b3d1b-5b57-4289-80ad-4a15a760c29c
  standard                          azure-rediscache           Standard Tier, 1GB Cache         4af8bbd1-962d-4e26-84f1-f72d1d959d87
  premium-p4                        azure-sqldb-db-only        PremiumP4 Tier, 500 DTUs,        feb25d68-2b52-41b5-a249-28a747bc2c2e
                                                               500GB, 35 days point-in-time
                                                               restore
  premium-p6                        azure-sqldb-db-only        PremiumP6 Tier, 1000 DTUs,       19487202-dc8a-4930-bbad-7bbf1486dbca
                                                               500GB, 35 days point-in-time
                                                               restore
  standard-s0                       azure-sqldb-db-only        Standard Tier, 10 DTUs, 250GB,   9d36b6b3-b5f3-4907-a713-5cc13b785409
                                                               35 days point-in-time restore
```

svcat also brings along other helpful commands, like `provision` and `bind`, which would replace creating the YAML above with more a more friendly command line experience. I'll be using svcat for the rest of this comparison instead of `kubectl`. These would be similar to corresponding `cf` commands for creating a service (what they would call a service instance) and binding a service to your app. 

To create my DocumentDB instance with `svcat`, I ran the following command.

```console
$ svcat provision osba-documentdb --class  azure-cosmos-document-db --plan document-db -p location=eastus
Name:        osba-documentdb
  Namespace:  default
  Status:
  Class:       azure-cosmos-document-db
  Plan:        document-db
```

I can then view the status with `svcat` like so.


```console
$ svcat get instances
            NAME              NAMESPACE              CLASS                PLAN          STATUS
+--------------------------+--------------+--------------------------+-------------+--------------+
  osba-documentdb   osba-example   azure-cosmos-document-db   document-db   Provisioning
```


Almost all brokers will do an asynchronous provision operation, and OSBA is no different. You see the status here provided as `Provisioning`. After a few minutes, it should be finished if all went well.

```console 
$ svcat get instances
            NAME              NAMESPACE              CLASS                PLAN       STATUS
+--------------------------+--------------+--------------------------+-------------+--------+
  osba-documentdb   default   azure-cosmos-document-db   document-db   Ready
```

At this point, we can create a binding to it. Service Catalog makes it a little easier than Cloud Foundry to bind your app to a service. In CF, you wouldn't be able to create the binding until you push your application. You could do this as part of the manifest when you are pushing the service or with the `cf` CLI after the application is pushed. If you read my post about using OSBA with pcf-dev, you can see an example of that. With Service Catalog, you can create the binding before you create the application. 

You could do this with `kubectl` and a YAML file, but I'll do it with `svcat`. 

```console
svcat bind osba-documentdb --secret-name docuemntdb-binding
```

Once you run this command, you can query it with `svcat` to see the status. You'll also be able to see the secrets created in Kubernetes once the binding is finished. The secret that gets created is what we'll want to use for our application!

```console
$ svcat get bindings
            NAME              NAMESPACE             INSTANCE           STATUS
+--------------------------+--------------+--------------------------+--------+
  osba-documentdb-instance   default   osba-documentdb-instance   Ready


$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
default-token-vkwss   kubernetes.io/service-account-token   3         7m
docuemntdb-binding    Opaque                                3         49s
```

That `documentdb-binding` secret will contain everything we need for our application. If you examine it, you'll see it contains the following important pieces of information.

* primaryConnectionString
* primaryKey
* uri

If we refer back to the required properties (which we can set via environment variables) for the Azure documentdb starter, we get a mapping like:

* AZURE_DOCUMENTDB_URI => uri
* AZURE_DOCUMENTDB_KEY => primaryKey

When you provide these values, the Azure Spring Boot starter and the Azure Spring Data DocumentDB library will create a `DocumentClient` for you. This is what you'd use to interact with the specified database. Currently, OSBA does _not_ create a database when you provision a DocumentDB service, only the DatabaseAccount, so we do not have the third required piece of information, `AZURE_DOCUMENTDB_DATABASE` (we have an [open issue](https://github.com/Azure/open-service-broker-azure/issues/259) to enhance that!!). The DocumentDB Spring Data library will automatically create collections in the database if they do not exist, but not the database itself. Using the Java SDK for DocumentDB, however, you can create a database with the uri and key above, so for this sample application I created an event listener and check for the database. If the database doesn't exist, it will do that. 

```Java
@Component
public class DatabaseChecker {


    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Autowired
    DocumentDbFactory factory;

    @Value("${azure.documentdb.database}")
    private String databaseName;
    
    @EventListener(ApplicationReadyEvent.class)
    public void checkDatabase() {

        DocumentClient client = factory.getDocumentClient();
       
        String queryString = String.format("SELECT * FROM root r WHERE r.id='%s'", databaseName);
        List<Database> databaseList = client.queryDatabases(queryString, null).getQueryIterable().toList();
        if (databaseList.size() < 1) {
            try { 
                Database database = new Database();
                database.setId(databaseName);   
                client.createDatabase(database, null).getResource();
            } catch (DocumentClientException dce) {
                logger.error("Unable to create databae", dce);
            }
        }      
    }
}
```

The `checkDatabase` method is invoked when Spring fires the `ApplicationReadyEvent`. The class gets a `DocumentDBFactory` autowired into it. This is created with the two pieces of our secret above. Next, it queries to see if the database exists. It gets the database name from the @Value annotation, which is populated by our environment variable. Once this is done, the database _should_ exist. 

At this point, all we need to do is wire that secret to our Spring Boot application via environment variables. This is done with normal Kubernetes mechanisms. 

```YAML
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: documentdb-example-deployment
  labels:
    app: documentdb-example
spec:
  replicas: 1
  selector:
    matchLabels:
      app: documentdb-example
  template:
    metadata:
      labels:
        app: documentdb-example
    spec:
      containers:
      - name: documentdb-example
        image: jeremyrickard/documentdb-example:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: AZURE_DOCUMENTDB_URI
          valueFrom:
            secretKeyRef:
              name: docuemntdb-binding
              key: uri
        - name: AZURE_DOCUMENTDB_KEY
          valueFrom:
            secretKeyRef:
              name: docuemntdb-binding
              key: primaryKey
        - name: AZURE_DOCUMENTDB_DATABASE
          value: osba-example
```

When I created the binding above, I specified a secret name of `documentdb-binding`. Using that secert name, I was able to wire up the secret to my application. I also specified the database name, since that is not created for us ahead of time. My event listener takes care of creating that database though. In contrast, if I deployed this with CF, I could define the binding in my manifest and then rely upon the VCAP_SERVICES environment variable being populated. Spring would also try to wire that up for me for many services (including DocumentDB here via the Azure library). For things that don't populate,  you still will get the VCAP_SERVICES environment variable and can pretty easily use it with the VCAP processor Spring provides. This is exactly what I did to create a MongoDB client using CosmosDB in my earlier blog post. 


Once you have the manifest created and you've created the instance and binding, you can deploy the application! I've included the manifest above in the my sample application repo and I've published an image to Dockerhub with the sample app at `jeremyrickard/documentdb-example:latest`. 

If you've been following along with the commands above and have a service instance and a binding created, you can run this in your Service Catalog and OSBA enabled cluster like so:

```console
kubectl create -f kubernets-manifest.yaml -n osba-example
```

If you'd like to build it yourself, that's pretty easy too:

```console
./gradlew clean build
```

Once that is done, you'll need to build a Docker image with the included Dockerfile, push it to a repository your cluster can use and update the manifest to use that instead. 

When this is up and running, you can see the DocumentClient being initialized with the secret we provided. It will look something like:

```
018-02-12 03:18:46.474  INFO 1 --- [           main] c.m.azure.documentdb.DocumentClient      : Initializing DocumentClient with serviceEndpoint [https://{some-identifier}.documents.azure.com:443/]
```

If you inspect the secret and decode the uri field, you should see a match between what's in the log file and what is in your secret. 

In the end, it’s fairly easy to build a Spring Boot service that makes use of Service Catalog when you deploy to Kubernetes. Everything really fits in with a Kubernetes native workflow, including the actual provisioning of your service. You don’t need to step out into an external portal to bring things online or worry about a stateful app going bad in your cluster. The addition of `svcat` makes things just a little nicer! You can even install it as a kubectl plugin if you want to stick entirely with that. 

While I think the experience provided by `svcat` and `cf` are pretty nice for developing your application and making supporting services available, ultimately Kubernetes needs to provide a higher level of abstraction for dependencies. You can get a taste of what this might look like with Helm [charts that are enhanced to create the service instance and bindings for you](https://github.com/Azure/helm-charts). This experience still isn’t perfect, however. You still need to create the secret mapping yourself though, and these aren't really portable between brokers since service plans and classes differ between brokers. The items contained in the secret generated by binding also aren't mandated to follow any sort of naming convention, so we probably need an abstraction on those as well. Our team is working on some proposals that we think will make some of these things better! Stay tuned for those. I think we'll get there eventually and you'll wonder how you got by without service brokers. 




