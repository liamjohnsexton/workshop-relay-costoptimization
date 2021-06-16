# Lab 2 - Cloud Compliance with AWS - WIP doc

## Backstory - "Guardrails not gates"

Say a customer is tasked with building a common cloud service platform for a federated set of their firms (based in UK, Canada, America). Rather than imposing a set of gates or prescriptive tools that everyone must use, they want to enable teams to use the tools they want while providing guard rails behind the scenes to ensure they remain secure and compliant – all while expanding from AWS to Azure, GCP, AliCloud etc.

Relay can be used to listen to events coming from a cloud provider (e.g. new ec2 instance in AWS) and automatically respond to events (e.g. stopping the ec2 instance that doesn't conform to the governance policy). In order to simplify the scenario, we're going to use a [schedule cron trigger](https://relay.sh/docs/using-workflows/using-triggers/#schedule-triggers) instead of the [AWS Event Bridge](https://relay.sh/integrations/aws-eventbridge/).

## Prerequisites (might need to be changed)

Before you run this workflow, you will need the following:
- User account on Relay SE account
- AWS account

## Configure the workflow  

Follow these steps to configure the workflow. Doing this will enable Relay to connect to your AWS account.

You may see a warning that you are missing a required connection. This means you will need to add your Azure credentials as a Relay connection.


- Click **Fill in missing connections** or click **Settings** in the side nav.

![Fill in missing connections](/images/missing-connection.png)

![Click settings from side nav](/images/settings-sidenav.png)

- Find the Connection named `my-aws-account` and click the plus sign **(+)**. 

![Guide connections](/images/guide-connections.png)

- Fill out the form:  

   - **Name** - You can’t change this with the form. The name is supplied by the YAML. If you wanted to change it you would need to do so in the Code tab.
   - **Access Key ID** - Enter your AWS Account Access Key ID
   - **Secret Access Key** - Enter your AWS Account Secret Access Key 

-  Click **Save** 

## Run the workflow manually with `dryRun` turned on

Navigate back to the Relay workflow. Follow these steps to run this workflow.

- Click **Run workflow** and wait for the workflow run page to appear.  

    ![Run workflow](/images/run-workflow-action.png)

- Supply values for the parameters fields when the modal appears:  

    ![Supply modal values](/images/dry-run-modal.png)

    - **dryRun** - `true` or `false`. To start with, let's use `true`.
       - `true` if you dont want to actually delete the resources. Use this to test the workflow and ensure it is behaving as expected.
       - `false` if you want the resources to be immediately deleted. 

    - **resourceGroup** - Specify the name of the resource group you created above.  

> **WARNING!** Be careful setting `dryRun` to `false`. Though the workflow comes with an approval step, once approved the resources will be terminated. Please use caution.

### Outcome
- By clicking on the `describe-instances` step, you should see ec2 instances active.
- By clicking on the `filter-instances` step, you should see an ec2 instance that is untagged. 

## Run the workflow manually with `dryRun` turned off 

Now, run the workflow again. This time specify the `dryRun` parameter to be turned to `false`. Configure the `resourceGroup` with the same name you specified before. 

When the `Approval` step is reached, select Yes to move to the Delete step. 
### Outcome
- After the workflow is completed, check the (AWS management console?) to see that the ec2 instance has been deleted.

## Run the workflow on a schedule  
Once we've confirmed that this workflow works, follow these steps to run this workflow on a schedule:  
- Un-comment out the included Trigger block in the workflow YAML. You can do this in the **Code** tab.

![Code tab](/images/code-tab.png)

> **TIP:** If you're using the Relay code editor, highlight the `triggers` section and type `⌘ + /` (Mac) or `Ctrl + /` (Windows) to uncomment.  

```yaml
# triggers:
# - name: schedule
#   source:
#     type: schedule
#     schedule: '0 * * * *'
#   binding:
#     parameters:
#       dryRun: true
#       resourceGroup: your-name
```

-  Configure the `schedule` trigger:  
   - Supply the run interval in [cron format](https://crontab.guru/).  
   - Configure the following parameter bindings:  
      - Specify whether `dryRun` should be set to `true` or `false`.  
      - Specify the `resourceGroup` to look for VMs under
```yaml
  binding:
    parameters:
      dryRun: true
      resourceGroup: your-name
```

- Click **Save changes**

### Outcome
- Once an hour, the workflow will run 
