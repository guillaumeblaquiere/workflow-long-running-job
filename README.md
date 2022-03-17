# Overview

That project is a sample of a long running job on a VM initiated by Cloud Workflow and with a callback notification
when the job is completed. It can take up to 1 year to complete

A [Medium Article](https://medium.com/google-cloud/long-running-job-with-cloud-workflows-38b57bea74a5) describes the 
use case and the path to the target solution

# General design

The principle is to run a job on a VM and to wait for completion. To achieve that with Cloud Workflow, here the
major step to achieve
* Create a Workflow callback URL
* Create a Compute Engine with
  * One attribute for the command to run (the long run processing)
  * One other attribute with the callback URL
  * Startup Script that run the command and, when completed, request securely the callback URL
* Catch the callback notification
* Delete the VM

# Deployment

The deployment requires the permission to 
* Use Cloud Workflow
* Create service account
* Grant permissions on service accounts

## Security configuration

There are 2 layers in term of security:
* Cloud Workflow itself must be able to create a VM
* The VM must be authorised to call the callback URL

### The Cloud Workflow security

For the workflow itself, we will use a custom service account to increase the security and enforce the least privilege
principle. We will reuse it during the deployment of the workflow.

```shell
#Create the service account
gcloud iam service-accounts create long-run

# Grant the required permissions
gcloud projects add-iam-policy-binding gdglyon-cloudrun --member="serviceAccount:$(gcloud iam service-accounts list --filter="name:long-run" --format="value('email')")" --role="roles/compute.admin" --condition=None
gcloud projects add-iam-policy-binding gdglyon-cloudrun --member="serviceAccount:$(gcloud iam service-accounts list --filter="name:long-run" --format="value('email')")" --role="roles/iam.serviceAccountUser" --condition=None
```

### The Compute Engine security

In the code, I use the compute engine default service account. *You can create a custom service account if you wish
and update the yaml script*.

The used service account must have 
[the role `roles/workflows.invoker`](https://cloud.google.com/workflows/docs/access-control#roles). Add it like this

```bash
gcloud projects add-iam-policy-binding gdglyon-cloudrun --member="serviceAccount:$(gcloud iam service-accounts list --filter="name:-compute" --format="value('email')")" --role="roles/workflows.invoker" --condition=None
```

## Deployment on Cloud Workflow

From your terminal run that command to deploy the workflow

```shell
gcloud workflows deploy long-run --source=long-run.yaml --service-account=$(gcloud iam service-accounts list --filter="name:long-run" --format="value('email')")
```

## Test

Then run the workflow and wait for a while (about 3 minutes for that hello world sample). Use a unique machine name
in parameter of the workflow, else it will failed because the VM can't be created

```shell
gcloud workflows execute long-run --data='{"instanceName":"long-run-vm"}'
```

You will be able to see a VM automatically created in your project and then destroyed

# License

This library is licensed under Apache 2.0. Full license text is available in
[LICENSE](https://github.com/guillaumeblaquiere/workflow-long-running-job/tree/master/LICENSE).
