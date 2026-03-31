# Save Cost with Azure Bicep Deployment Stacks ?

This time I'm here to talk about how to save cost using Azure Deployment Stacks. Assume everyone has a pretty good understanding about deployment stacks, let's see how we can use it for our advantage. To talk about it, I'm going to use a scenario. I believe it's the best way to explain and simulate the situation.

## Scenario

In our Azure environment we have few virtual machines, and we use Azure Bastion to access the VMs, but we only work between 8am to 5pm. So, there is no point running Azure Bastion 24/7 rather we save some money.

The same concept applies to other resources too. Think App Service Plans, App Services, or any resource that gets charged once provisioned. In this repo I've expanded the idea beyond just Bastion to show how you can conditionally deploy App Services and App Service Plans as well.

## Here is the plan of attack

- Bicep template with a `deployResources` boolean parameter
- Resources wrapped in a conditional `if (deployResources)` block
- Deploy everything using an Azure Deployment Stack with `--action-on-unmanage deleteAll`
- Create a scheduled DevOps pipeline and change the parameter accordingly

## The Bastion Example

Following is my original Bastion bicep template (`deployment-stack-cost-saving/bastion.bicep`), it's a simple file just to show the process. There may be a lot of hard coded values.

![image](https://github.com/user-attachments/assets/0eccd10b-2e7c-4d9f-b7e7-f06b1c6ca541)

Important variable here is `deployResources` Bool value. When it's `true`, the Bastion and its Public IP get deployed. When it's `false`, they are excluded from the template and the deployment stack cleans them up.

## The Expanded Example (App Service)

I've taken the same concept further in `main.bicep`. This template is subscription-scoped and deploys:

**Always-on resources (deployed regardless):**
- Resource Group
- Key Vault (using AVM public module `avm/res/key-vault/vault`)
- Storage Account (using AVM public module `avm/res/storage/storage-account`)

**Conditional resources (controlled by `deployResources`):**
- App Service Plan (using a custom module in `module/web/app-service-plan/`)
- App Service (using a custom module in `module/web/app/`)

So the Key Vault and Storage Account are always there, but the App Service Plan and App Service only get deployed when `deployResources` is `true`.

## Parameters

Parameters are managed using `.bicepparam` files which is a nice way to keep things clean:

- `global.bicepparam` extends `global.bicep` for shared values like location shortname, environment, and company prefix
- `main.bicepparam` extends `global.bicepparam` and sets the `deployResources` flag. It also pulls a secret from Key Vault using `getSecret()`

## Deployment Stack CLI

Here is how you create/update the deployment stack. I've included sample commands in `stack.cli`:

```bash
az stack sub create \
  --name 'sampledeployment' \
  --location 'australiaeast' \
  --template-file './main.bicep' \
  --parameters './main.bicepparam' \
  --parameters deployResources=false \
  --deny-settings-mode 'none' \
  --yes \
  --action-on-unmanage deleteAll
```

The magic here is `--action-on-unmanage deleteAll`. When a resource is no longer in the template (because `deployResources` is `false`), the stack automatically deletes it. No orphaned resources, no wasted money.

## Pipeline

Here is my pipeline files

I create 2 files to run on 2 schedules. One at morning 6am

> **Note:** in this pipeline my deploy resources parameter is set to true, as I wanted to get bastion deployed in the morning

![image](https://github.com/user-attachments/assets/db7a7285-6689-409e-83c9-a220d2dd43b7)

And second one at 6pm

> **Note:** in this pipeline my deploy resources parameter is set to false, as I wanted to delete bastion in the afternoon

I've also included a `testing-pipeline.yml` that demonstrates a CI pipeline with:
- Security scanning using Microsoft Security DevOps (Template Analyzer for IaC)
- Bicep validation (builds all `.bicep` files to check for errors)
- What-If analysis to preview changes before deployment

Based on how deployment stack works following is the output at

**6am everyday**
![image](https://github.com/user-attachments/assets/96c45206-5022-4438-9a05-df5af96ec748)

**6pm everyday**
![image](https://github.com/user-attachments/assets/a42d1320-344e-46e0-83e3-be0366b18550)

## Repository Structure

```
main.bicep                  - Main subscription-scoped template
main.bicepparam             - Parameters (Key Vault secret, deployResources flag)
global.bicep                - Shared parameter definitions
global.bicepparam           - Shared parameter values
stack.cli                   - Sample az stack CLI commands
testing-pipeline.yml        - CI pipeline (security scan, validation, What-If)
bicepconfig.json            - Bicep configuration

deployment-stack-cost-saving/
  bastion.bicep             - Original Bastion conditional deploy example

module/
  bastion/main.bicep        - Reusable Bastion module
  web/
    app/                    - App Service modules (Windows, Private Endpoint)
    app-service-plan/       - App Service Plan module
    config/                 - Web config module
    function/               - Function App modules (VNet integrated, premium)

working-files/              - Draft/working versions of main templates
```

## You can use this methodology to many scenarios

- If you have dev environments and wanna save more cost
- Unwanted resources during office hours like Azure Bastions etc. (which gonna get charged once provisioned)
- App Service Plans and App Services that are only needed during business hours
- Any pay-per-hour Azure resource

And you might think why not use the same pipeline, I have few reasons behind it

- Wanted to differentiate runs easily
- We have more flexibility when running things or scheduling things
- Keep things simple

And definitely there is no problem using the same pipeline and achieve the same thing.

Also in the pipeline you can improve it by using pipeline templates and use variables instead of parameters etc.

## Conclusion

I wanted to show the power behind using Azure IaC, Deployment Stacks and Pipelines, anyone who is leveraging these practices can manage your Azure cloud environment effectively and cost efficiently. The Bastion example shows saving cost by half, and the App Service example shows the same pattern works for any conditionally deployed resource. So depending on your scenario and use case you can save more money without need to think about reservations etc.