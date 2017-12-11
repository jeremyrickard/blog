---
title: "Learning Brigade"
date: 2017-12-09T18:47:03-08:00
draft: true
---

One of the things I'm currently looking into in my new job is how we might use [Brigade](https://github.com/Azure/brigade) to handle our CI needs instead of CircleCI. I won't copy the Brigade README here, but at a high level Brigade is an event based scripting system that users Kubernetes primitives like pods and secrets. It provides out of the box support for GitHub events like pull requests and pushes and allows you to build workflows (i.e. CI pipelines) using fairly simple JavaScript. Within a Brigade event handler, you can define one or more Jobs and execute them in parallel or sequence. Jobs are Docker containers with an associated set of __tasks__. Brigade makes it fairly easy to execute your build/test processes against your repo because if you configure your Brigade project to include a source code repository, Brigade will automatically mount it into the Job container for you. You just need to write one ore more jobs and Brigade will execute them for you.

The Brigade team has also just released a tool called [Kashti](https://github.com/Azure/kashti) that provides a UI/dashbaord for your Brigade projects. Both Brigade and Kashti are _fairly_ new and you might encounter issues when using them so feel free to submit issues on GitHub if you run into any!

## Getting Started

Brigade is a tool for Kubernetes, so you'll obviously need a Kubernetes cluster to get started. In order to take advantage of GitHub webhooks, you'll need to expose your Brigade gateway to the internet. While it is possible to do that with Minikube, I'll be using an [AKS](https://azure.microsoft.com/en-us/services/container-service/) cluster for this. I already created the cluster and I am using it test out Brigade for our projects. If you have an Azure subscription, it's really easy to create an AKS cluster following this [walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough). You'll also want kubectl installed and have it configured to talk to your cluster.  

The easiest way to install both Brigade and Kashti is to use [Helm](https://github.com/kubernetes/helm). The Helm GitHub repo has binaries for most operating systems and you can install it by simply unpacking the binary and putting into your path. If you are using a Mac, it's also really easy to install with [brew](https://brew.sh/). 

The Brigade [installation guide](https://github.com/Azure/brigade/blob/master/docs/topics/install.md) has great instructions on using Helm to install Brigade. Follow those, and you'll have Brigade up and running in your cluster. You'll probably also want the *_brig_* command line tool. That will let you trigger workflows manually and test out changes. I don't think the Brigade team is currently distributing binaries of the brig tool, so you'll need to build it yourself. To build the tool, you will need to install Go and npm installed. Building *_brig_* is pretty simple once you have Go and nom installed, you just need to clone the Brigade repo, make sure your GOPATH is set correctly and use the provided Makefile. I built it using the following commands:

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

You'll also want to clone the Kashti repo as the Helm charts are currently located within the GitHub repo. The Kashti [installation guide](https://github.com/Azure/kashti/blob/master/docs/install.md) has great instructions on how to install Kashti.

Once you have Brigade installed, you'll need to create a new project. The Brigade [tutorial](https://github.com/Azure/brigade/blob/master/docs/intro/tutorial03.md) has good information on this, including walking you through setting up integration with github. If you intend to integrate with GitHub, make sure to follow the instructions regarding generating a GitHub oauth token. If you want to report status back to GitHub, you'll also want to store that as a secret in your project called `githubToken`. More on that later. 

If you're using a private repo, you'll also need to create a deploy key so you can clone the repository. 

## Add some code and a brigade.js file

I decided to learn brigade using a [Spring Boot](https://projects.spring.io/spring-boot/) project. I generated one with [start.spring.io](http://start.spring.io/) and then created a new GitHub repo with the project that Spring Initalizr generates. You can see that project [here](https://github.com/jeremyrickard/brigade-demo). 

This project uses Gradle to build and comes pregenerated with simple test. If you clone the repo, you can build the project and run the test with:

```
$ ./gradlew build
```

You will need to have Java installed to do that, but `gradlew` is a gradle wrapper and will download Gradle if it doesn't exist. The build target will then build the project and run the tests. 

With that fact in mind, I wrote a pretty simple initial version of a brigade.js file

```
const { events, Job, Group } = require("brigadier")

events.on("pull_request", (event,project) => {
        var testJob = new Job("test", "openjdk")
        testJob.tasks = [
            "cd /src",
            "./gradlew build",
        ]

        testJob.run()
```

This adds a pull request handler to my brigade project. When the handler is triggered, it creates a new Job called testJob. It will use the [openjdk](https://hub.docker.com/_/openjdk/) image from Dockerhub. When this Job is run by Brigade, it will create a new Kubernetes pod using the openjdk image. It will then run the two commands specified in the tasks array. By default, Brigade will mount a copy of the GitHub repo at /src, so the first command is to change to that directory. Next, the Job will run the same `./gradlew build` command I mentioned above. If the code builds and the tests pass, the container will exit normally and Brigade should mark the job as successful. 

To test this out, I'll make a new branch of the repository, push a small non-breaking change to the branch and then open a pull request. GitHub will send the pull request event to Brigade and that should trigger my event handler. Looking at the pull request, GitHub shows that Brigade automatically updated the status to say it was building! That's pretty cool. Kashti will show the event and should also show if the Job was successful or not. 


Kashti shows that the Job was successful, but the status wasn't updated in GitHub. It turns out that while Brigade currently reports status back to GitHub when an event handler is triggered, it doesn't send subsequent reports. They intend to add this later on, but you need to handle this yourself for now. Luckily, Matt Butcher has you covered with [github-reporter](https://github.com/technosophos/github-notify). You can define a new Job in brigade using that container. You'll need a github oauth token in order for this to work, so if you didn't already add that to your brigade project config, you can do that now by adding the following block to your project values YAML file:

```
secrets:
  githubToken: YOUR_ACTUAL_TOKEN
```

Once you make that change, you can use helm upgrade to update the values of your project. Once you've done that, you can add the job to your brigade.js file like this:

```
const { events, Job, Group } = require("brigadier")

events.on("pull_request", (event,project) => {
        console.log("Handling a pull request for :" + event.commit)
        var testJob = new Job("test", "java")
        testJob.tasks = [
        "cd /src",
        "./gradlew clean build",
        ]

        var reporter = new Job("reporter", "technosophos/github-notify:latest");
        reporter.env = {
                GH_REPO: project.repo.name,
                GH_STATE: "success",
                GH_DESCRIPTION: "build successful",
                GH_CONTEXT: "brigade",
                GH_TOKEN: project.secrets.githubToken, // YOU MUST SET THIS IN YOUR PROJECT
                GH_COMMIT: event.commit

        }

        testJob.run().then( () => {
        	reporter.run()
	})
})

``` 

If I push a new change to the branch I used before, GitHub should send a new event to the webhook and trigger a new build. Once the testJob is done, it should then run the reporter job and update the GitHub status. 


Now, I should be able to push a change to the branch that will result in the Job failing. Once this happens, I'd expect to see the failure in KAshti and see the status updated in GitHub. After pushing the change, I see that indeed the failure shows up in Kashti, but GitHub still says it is building and never updates. What happened?

When you run a Job in Brigade, you get a JavaScript [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise). If the Job is successful, that Promise is fulfilled. If the job is not successful, it is rejected. In the above brigade.js file, I'm not handling the Promise rejection. The brigade examples don't really touch on this and I had not used JavaScript Promises before, so it took me a little to understand that. You can handle this a couple of ways:

You can catch an error from the promise like this:

```
const { events, Job, Group } = require("brigadier")

events.on("pull_request", (event,project) => {
        console.log("Handling a pull request for :" + event.commit)
        var testJob = new Job("test", "java")
        testJob.tasks = [
        "cd /src",
        "./gradlew clean build",
        ]

        var reporter = new Job("reporter", "technosophos/github-notify:latest");

        testJob.run().then(results => {
 		reporter.env = {
                	GH_REPO: project.repo.name,
                	GH_STATE: "success",
                	GH_DESCRIPTION: "build successful",
                	GH_CONTEXT: "brigade",
                	GH_TOKEN: project.secrets.githubToken, // YOU MUST SET THIS IN YOUR PROJECT
                	GH_COMMIT: event.commit
        	}
		reporter.run()
        }, error => {
		reporter.env = {
               		GH_REPO: project.repo.name,
                	GH_STATE: "failure",
                	GH_DESCRIPTION: "build failed",
                	GH_CONTEXT: "brigade",
                	GH_TOKEN: project.secrets.githubToken, // YOU MUST SET THIS IN YOUR PROJECT
                	GH_COMMIT: event.commit
        	}
		reporter.run()
        })
})

```

This does report the status back to GitHub, but Kashti shows the job as successful. When I added the rejection callback, Brigade no longer gets the error. So to do that, I need to make a slight modification to use the reporter.run() promise:

```
```

