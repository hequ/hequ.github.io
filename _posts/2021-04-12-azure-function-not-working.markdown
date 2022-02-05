---
title: Why Azure function does not log into App Insights?
date: 2021-04-12 00:00:00 +0200
categories: terraform azure
layout: post
---

TL;DR; How I fixed Azure functions not sending data to Azure AppInsights by adding a missing hidden-link tag.

Last time I wrote about how it makes sense to wire your cloud services together using the cloud console instead of instantly writing Terraform configuration. Today’s post is a bit about the same thing, but slightly different. It’s about how sometimes the abstractions leak, and you need to understand what happens behind the scenes to get things working. This is about how I wanted to attach Azure AppInsights to my Azure functions to monitor the functions app.

AppInsights is a monitoring tool that you can use to understand better what is happening in your application. You can monitor many different kinds of applications with it, including Azure functions. Azure functions provide a nice and easy way to set up AppInsights from the Azure portal. From the portal, you can set up the integration with only a couple of clicks. Reading the docs, you would think that the process is as simple as creating an AppInsights workspace and adding an instrumentation key to your Azure functions. That should be enough to start the telemetry data flowing from functions to AppInsights.

So I did that directly from the Azure portal, and everything worked as expected. Then I wanted to do the same thing from Terraform to get the Infrastructure as a Code’s benefits and replicate the same setup into another environment. To my surprise, that same setup did not work in a new environment. I wondered why and spent a good hour debugging and making sure that I understood the documentation. Once I double-checked everything and the monitoring still did not work, I started to search if others would have suffered from this issue. So I found a Github issue that explained my problem: behind the scenes, azure creates a hidden link tag to wire functions app and app insights together! So when you set up the integration from the Azure portal, it creates an Application Insights workspace, configures functions app with an instrumentation key, and creates a hidden link to the functions app, which binds the services together.

So after finding this out, the problem was pretty straightforward to fix. I added the required hidden tag into my functions app and deployed the change. After that change, the monitoring started to work as expected. At the time of writing, this bug (or limitation) is still open in Azure CLI, so this is not only a Terraform problem. This was a good reminder that most abstractions are never complete, and you should investigate how and why things actually work so you can fix them when they break.

Here’s a sample tag that you can use in your terraform config. The same syntax also applies if you insert the tag directly from the Azure portal or Azure CLI.

```terraform
tags = {
    "Name" = "My Functions App"
    "hidden-link:/subscriptions/${var.subscription_id}/resourceGroups/${var.rg_name}/providers/microsoft.insights/components/${app_insights_workspace_name}" = "Resource"
}
```
