---
title: "Fun With ACI"
date: 2018-02-09T21:53:28-07:00
draft: false
---

Friday is usually a pretty good day. It's the end of the week and the Azure Containers team has a show and tell with fun demos and people share rants. Today there were also bunch of jokes about [clippy](https://en.wikipedia.org/wiki/Office_Assistant) which resulted in a bunch of hilarious art from http://textart.io/cowsay/clippy. Such as... 

{{< tweet 962034432541519873 >}}

Then there were jokes that we needed a bot....And that we should run it on [ACI](https://azure.microsoft.com/en-us/services/container-instances/). 

That felt like a good joke and an excuse to use ACI...so with the help of the super useful [nlopes/slack](https://github.com/nlopes/slack) library, I made [one](https://github.com/jeremyrickard/clippy-bot) right before I left to pick up my son from school.

![heyyyyy](/images/aci/hey.png)

So, you want to run a Slack bot (or anything in a container...) somewhere. ACI makes it SUPER easy. 

Step 0. Make sure you have the [az cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed. 

Step 1. Make a resource group (if you want a special one..i like them so I did). 

```
az group create --name clippy --location eastus  
```

Step 2. Push your container somewhere

```
git clone https://github.com/jeremyrickard/clippy-bot
docker build -t .
docker tag clippy-bot jeremyrickard/clippy-bot
docker push jeremyrickard/clippy-bot
```

Step 3. Run your container in ACI!

```
az container create \
   --memory .5 \
   --resource-group clippy --name clippy \
   --image jeremyrickard/clippy-bot \
   -e API_KEY={SLACK_API_KEY} BOT_USER={BOT_USER} KEY_WORD={TRIGGER PHRASE}
```

Step 4. Check the status and wait for the container to start

```
$az container show --resource-group clippy --name clippy -o table
Name    ResourceGroup    ProvisioningState    Image                     CPU/Memory       OsType    Location
------  ---------------  -------------------  ------------------------  ---------------  --------  ----------
clippy  clippy           Succeeded            jeremyrickard/clippy-bot  1.0 core/0.5 gb  Linux     eastus
```

In a minute or two, I had my bot up and running in ACI and was making amazing ASCII art in Slack. I didn't need to spin up any infrastructure or have an existing Docker runtime environment. I'm getting per second billing, so I can use it for just as long as I need it (which is obviously all the time). The cost could add up if I ran it continously for 24x7, but the current [pricing](https://azure.microsoft.com/en-us/pricing/details/container-instances/) is pretty nice. Even if I ran this 24x7, it ends up being around a $1.55 USD a day. This container was pretty simple and didn't need to expose any ports, but you can [do that](https://docs.microsoft.com/en-us/cli/azure/container?view=azure-cli-latest#az_container_create) too. You could also mount an [Azure File Share](https://docs.microsoft.com/en-us/azure/storage/files/storage-how-to-create-file-share).    


