---
title: How can Terraform import help you with your daily cloud work?
date: 2021-03-19 00:00:00 +0200
categories: azure terraform
layout: post
---

You probably know that your infrastructure should be written as code (or configuration if you like), but it doesn’t mean that you should ditch using the cloud console completely. If you are using Terraform tooling and don’t know what terraform import does, then this article is a small primer into just that. Here I’ll describe how I like to work with Terraform in my daily work.

Terraform lets me write my cloud configuration as code. It’s a neat and powerful tool which allows me to configure my infrastructure once and apply it to multiple environments, and multiple times. The thing is, though, that to be efficient with Terraform, I need to know my cloud pretty well to configure my resources just right, or otherwise, I may end up with a configuration that doesn’t work quite as expected.

If I’m working with familiar resources, then it’s usually quite straight-forward to write the Terraform configuration for those resources. But this requires that I know exactly what I’m doing. Sometimes I don’t quite know how to wire up the services in practice, but I might have an vague idea how to do it. These times using the cloud UI is usually helpful as I try how the services work and build the mental model.

When I don’t know exactly how something needs to be done, then writing the terraform config can be slow and error prone. It has happened to me, more than once, that I have created a configuration with Terraform which looked ok, and was provisioned fine, but didn’t work as expected. For this reason, in situations where I don’t exactly know how the end result looks like, I like to configure resources in the cloud UI the first time I use that particular resource. When I configure the resources in the cloud UI, the resources usually work as expected, and then it’s easy to bring those resources to be managed by Terraform.

I’ll start by adding a “skeleton” of a resource into Terraform configuration. For example, if I’m about to figure out how Azure Storage Account works, I’d add the following resource into Terraform configuration:

```terraform
resource "azurerm_storage_account" "example" {
    name = "storageaccountname"
}
```

Then I’ll play, and tune the settings of that resource in the Azure Portal, and once I’m happy, I’ll use terraform import to pull those changes into that skeleton (well it actually updates the Terraform state, and not that configuration).

Terraform import takes two arguments plus a bunch of options, but I use it very often in its very basic form: terraform import address id. Address here is the resource id that you define in your Terraform config. For example, using the example above, the address would-be azurerm_storage_account.example if you define this resource in your Terraform root-level config. id would be the Azure resource id which you can find from the azure portal from that particular storage account’s settings. So the final import command would be something like:

```s
terraform import azurerm_storage_account.example /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myresourcegroup/providers/Microsoft.Storage/storageAccounts/myaccount
```

Once I run this command, Terraform will update the state file with the settings that the actual resource in the Azure cloud has. Now, this is useful, as next, I’ll run terraform plan to see the changes between my “skeleton” configuration and the actual resource. Then I need to do some copy/paste work to copy those changes into my Terraform configuration and run plan again. When I have my configuration written so that the plan doesn’t show any more changes, then I have successfully imported a resource into my Terraform configuration, and the final configuration might look something like this:

```terraform
resource "azurerm_storage_account" "example" {
    name = "storageaccountname"
    location = azurerm_resource_group.example.location
    resource_group_name = azurerm_resource_group.example.name
    account_kind = "Storage"
    account_tier = "Standard"
    account_replication_type = "LRS"

    tags = {
        Application = "MyExampleApp"
    }
}
```

When I have created the Terraform configuration this way, then it’s easy to just run that configuration to replicate my new resources for example to a higher level environment such as QA and Production.

So this how I usually work with Terraform if I’m dealing with something new. This is a trivial example but if I work with something more complex, like wiring together more than one service, this method makes it easier to understand what is really happening in the cloud.

The benefit of Terraform import is to allow me to provision the resources in the cloud UI and not messing with terraform configuration if it’s unclear how I need to wire things together. I don’t say that it isn’t sometimes faster or easier to wire everything together with Terraform directly, but terraform import is definitely a tool that I keep using very regularly. If you haven’t tried that, I suggest you to do so when you next work with something new in your cloud.

Thanks for reading, and please let me know your thought in the comments section below!
