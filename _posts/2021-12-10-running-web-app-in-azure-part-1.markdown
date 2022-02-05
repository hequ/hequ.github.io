---
title: Running web app in Azure part 1
date: 2021-12-10 00:00:00 +0200
categories: azure docker
layout: post
---

In this hands-on article series, we will set up Microsoft Azure App Service to run our containerized web application

# Forewords

This is the first article of an article series that will focus on building a great web app infrastructure around Microsoft Azure AppService (AS). In this first chapter we are going to look at how we can wrap our Nodejs application into a docker container, and then run that in AppService. In the following chapters, we will look at how to make our application more observable by using Azure built-in monitoring and logging tools. In the last chapter of this series, we’ll focus on making our service more robust by using tools like Azure Virtual Networks, and Azure Frontdoor web application firewall. But for now, let’s start building our very first AppService.

## About tools used in this article

If you want to follow along actually building the App Service, you need to have the following tools available and installed:

- Terminal, preferably sh-based, but PowerShell will also do. Powershell commands in az are just a bit different.
- Docker, ability to run docker commands in terminal
- Azure-cli (az)
- Azure account
- Typescript-dockerfile

Now when these are set up, we can continue to start building our very first App Service!

# What is Azure AppService

AppService is a fully managed PaaS (platform as a service) to run our web applications. We can deploy our web app in many forms into App Service. The model of how AS operates behind the curtains is that we select an App Service Plan (ASP), which is a pre-defined set of resources that are allocated for our App Service(s). This means that our app will be run on a Virtual Machine (VM) that is fully managed for us. We don’t have to take care of managing or updating it in any way. We need to, however, select the VM instance size that is suitable for us. There are many different options to choose from, and we can for example run our VM in a shared environment if we want to run our test environment, or there are dedicated and isolated environments that are meant to run production-grade apps.

One important thing to understand about App Service is that when we create an App Service Plan, we are basically defining the available compute resources for our application. we can then run one or more instances of our app inside the ASP (those are called App Services), but those instances will share the available resources of the ASP. This means that we need to select a heavy enough ASP so it can handle those peak moments that our application might face. AppService supports autoscaling, but we need to understand that it scales between the pre-defined resources of the ASP, so it doesn’t add any extra App Service Plans if our currently selected ASP runs out of juice. If we need “true” autoscaling, then we need to look somewhere else. One good option is to run our own Kubernetes cluster with Azure Kubernetes Service (AKS).
That being said, there are a couple of ways to run our app in AS (including directly running a node.js app), but we’ll focus on building a Docker container because it offers us a couple of neat benefits.

# Running a Docker container

Running a Docker container is a good idea because when we wrap our application inside a Docker container, we are basically defining the run environment for our app and abstracting away the nitty-gritty details of how our app needs to be run (we of course need to write those details, but that is a different story). When our app is in a Docker container, we can just put that container to whatever service that supports containers, and it will work. That’s the beauty of Docker.

Docker also allows us to switch away from App Service in the future without huge pain if we want to do it. Because our app is already in a container, and we can just search for a new home for it.

# Working with Docker images in Azure

We have now wrapped our app into a Docker container (and my previous article shares an example of how to do that) and we want to use that in App Service. How do we do that?

Azure has a private docker registry called Azure Container Registry (ACR). If we don’t want to use ACR, we can of course host our Docker images in whatever registry we like, but because ACR is a service inside Azure, the user management is easier inside Azure.

One cool thing about ACR is that it can use Azure’s Managed Identities to manage access to the service. This means that we don’t have to use a username + password combination to authenticate services to each other. We can allow our App Service to read Docker images from ACR just by configuring App Service to use managed identity. However, in practice, this doesn’t quite work well with App Service just yet. There is an open ticket to allow App Service to read Docker images from ACR using managed identities. Until that is fixed, we need to use username + password to authenticate, but I’m optimistic that this will be fixed in the future and therefore I recommend using ACR if possible.

From here-on now, we need access to Azure, and if we don’t have an account already, we can go and sign-up here:

[azure.microsoft.com](https://azure.microsoft.com/)

# Drafting our Plan

Before we actually do anything in Azure, we want to have a plan of what we are trying to achieve. Our initial plan will be, that we will build and push a docker image into ACR, and then we configure AppService to use that freshly built image.

## Plan how to run the container in AppService

Pretty simple, right? Yep, it doesn’t have to be any more complicated.

## Azure CLI

We could be using the Azure portal to create all of our resources, but because we want to do things a bit easier, we’ll use an awesome tool called az. Az is an azure-cli tool (we run it in our terminal) that we can use to manage pretty much anything in the Azure cloud. Check here for installing instructions.

Now when we have Az installed, we need to login into our Azure account. Before using az, we need to be logged in to our Azure account with our browser as az completes the login in the browser. When we are, we can log in az by running

`az login`

and finish the login in our browser.

# Resource Group

In Azure, we will set up our resources in resource groups. A resource group (RG) is a way to bundle all the related services into one set. This way we can easily see from the Azure portal what resources we have in our current environment.

We can create a new resource group by issuing:

```s
az group create --location northeurope --resource-group YourResourceGroupName
```

When that is set up, let’s create our ACR.

Azure Container Registry
Like we drafted before, we need to set up ACR where we’ll push our Docker images. Let’s create one by issuing:

```s
az acr create --name YourACRName --resource-group YourResourceGroupName --sku Basic --admin-enabled true.
```

This command outputs the ACR login server URL. We need to save that as we use it in later steps.

We need to give our ResourceGroupName as a parameter to az when creating ACR, that way the newly created ACR will be placed into our RG. Notice that we are giving --admin-enabled true to enable username + password access to ACR. We will need this so we can set up our AppService to fetch images from this ACR later. Like we discussed earlier, a better way to access ACR would be to use managed identities, but because App Service does not quite support this (yet), I’d personally stick to using admin access until AppService better supports Managed Identities.

## Push Docker Image to ACR

ACR is now ready and we can build and push our sample web app into our ACR.

I use the typescript-dockerfile as our sample. It contains a sample web app that we can use for demonstration purposes (and it’s a good image to build our typescript web app on later). Let’s start by building that sample Dockerfile:

```s
docker build . -t youracrname.azurecr.io/yourapp:latest -t yourapp:latest --target final
```

Replace youracrname.azurecr.io with the ACR login server URL we got earlier. We are tagging this image with two separate tags. Another tag (..azurecr.io/..) is used when the image is pushed into ACR, and the other tag is a convenience tag to reduce typing when building the image in our local environment.

Before pushing this image to ACR, let’s try it locally, so we don’t push a broken image.

```s
docker run yourapp:latest -p3000:3000.
```

The app should start and if we visit `localhost:3000` with our browser, we should see a “hello world”, message.

When everything works, next we need to login into our ACR (we need Docker installed in order for this to work):

```s
az acr login --name YourACRName.
```

This logs our Docker client into ACR. If that succeeds, we can push our Docker image to ACR:

```s
docker push youracrname.azurecr.io/yourapp:latest
```

## App Service Plan

Before we launch the AppService itself, we need to configure App Service Plan (ASP). ASP is the host where our App Service will run. Like we discussed earlier, ASP defines all the resources that our AS can utilize. So when creating a plan we need to define how much CPU, memory, or other AS features we’ll need. We can always change plans to lower or higher tiers if we need to, so this choice is not set in stone. We’ll start with a free plan:

```s
az appservice plan create -g YourResourceGroupName -n YourASPlanName --sku F1 --is-linux
```

We specified the `--is-linux` flag because we want our AS to be running on a Linux host. Linux hosts are a bit cheaper than similar Windows hosts, and because we are running containers anyway, Linux is a better choice.

App Service
Now we get into the beef of the whole thing. We get to launch our AppService! We’ll use the image from ACR which we previously deployed.

Let’s spin up AppService using the image we have in our ACR:

```s
az webapp create -g YourResourceGroupname -p YourASPlanName -n UniqueWebAppName -i youracrname.azurecr.io/yourapp:latest
```

Now the app service is up and running, and it should have pulled the image from ACR. Before we can try it, we need to do one more thing. We must tell the App Service what port the container is listening to. The example I’m using listens on port 3000.

```s
az webapp config appsettings set --resource-group YourResourceGroupName --name UniqueWebAppName --settings WEBSITES_PORT=3000.
```

It’s good to know that when we change any of the app settings, that will restart the app. After a little wait, the app should be accessible at

https://uniquewebappname.azurewebsites.net

And one of the cool things here is that our app has an HTTPS endpoint with a load balancer by default! So if we’d want to scale our app into multiple instances, the load would be automatically balanced.

If the app didn’t start for some reason, we can get a hint of that by accessing the Docker logs:

```s
az webapp log tail --resource-group YourResourceGroupName --name UniqueWebAppName
```

This should give us an idea of why the app didn’t start, and how we can fix it.

What’s next
We have now set up App Service and deployed our first Docker container into the service. We can now go and delete all the resources from the resource group to save costs:

```s
az group delete -g YourResourceGroupName
```

This also demonstrates why Resource Groups are a nice abstraction. If we ever need to bring down our environment, we can do it safely without leaving any resources dangling in the cloud.

In the next article, we are going to go through how we can improve the observability of our AppService, by using tools like Azure Log Analytics and Application Insights.

If you liked this article, please let me know by hitting the like button, and by leaving a comment. You can also subscribe to this newsletter to get it delivered right into your inbox. I’m planning to publish 1-2 longer articles (like this) per month, so no spam, I promise!

Thank you for reading this far, I truly appreciate it!
