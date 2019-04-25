---
layout: post
title: ARM Templates and Default Values
#subtitle: Welcome to my tinkering journey
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

[Azure Quickstart Templates](https://azure.microsoft.com/en-us/resources/templates/) offer a convenient way of provisioning resources on Azure using community maintained templates. Today I played with the [Simple Ubuntu Linux VM](https://azure.microsoft.com/en-us/resources/templates/101-vm-simple-linux/) template and deployed it by using both, the Azure Portal and the CLI.

![The simple Ubuntu Linux VM template](/img/2019-04-25-simple-ubuntu-template.png)

The simplest way to deploy a template is by hitting the "Deploy on Azure" button.
This will pop up the Azure Portal and ask you to put in all required values, like `Admin Username`
in this case. It also shows you predefined default values and expressions that are evaluated when the
template is deployed (see `Location` here).

![Deploying an ARM via the Azure Portal](/img/2019-04-25-arm-portal.png)


If you deploy using the command-line, you won't see the default values. And this tripped me:

```
$ az group deployment create --resource-group myrg --template-uri https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-vm-simple-linux/azuredeploy.json
Please provide string value for 'adminUsername' (? for help): jackson
Please provide securestring value for 'adminPasswordOrKey' (? for help):
Please provide string value for 'dnsLabelPrefix' (? for help): jackson
Deployment failed. Correlation ID: XXXXXX. {
    "error": {
        "code": "InvalidParameter",
        "message": "The value of parameter linuxConfiguration.ssh.publicKeys.keyData is invalid.",
        "target": "linuxConfiguration.ssh.publicKeys.keyData"
     }
}
```

For `adminPasswordOrKey` I had typed in a password. Note that it complains about an invalid key
though. Turns out the (sensible) default is using an SSH key. You can see this from
the Portal screenshot above.

So the trick in this case is to override the default value for `adminPasswordOrKey` and set it to
`password` (or simply use an SSH key). This is achieved by using `--parameters` with
`KEY=VALUE` pairs:

    --parameters authenticationType=password

or using JSON

    --parameters'{"authenticationType": {"value":"password"}}'

Lesson learned.

