---
layout: docs
title: Deploying Functions
permalink: /docs/deploying-functions/
---

## Deploying Functions 

Assuming you already have an [AWS account](https://aws.amazon.com/) and have installed [sbt](https://www.scala-sbt.org/1.x/docs/Setup.html) and [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html), you can have a function deployed in less than 5 minutes (for real, we did it live during a few internal presentations to prove it).

**Run the `scalambdaTerraform` SBT task.** This will generate a Terraform module (by default it will be placed inside your project's `target/` folder). You can then use this module to deploy your project. Super easy, right?

You can do pretty much anything you want with the Terraform that Scalambda generates. You could package it and hand it off to your dev-ops team, throw it in a Docker image, whatever you'd like! 

**At a high-level, the deploy process for almost all Scalambda application** is: generate a Terraform module, then apply the Terraform module.  

## An Opinionated Deployment Process

In this section, __we're going to show you what we (at Carpe Data) do with the Terraform__, just so you can get an idea of how it might be used before you brainstorm for yourself how it best fits your workflow.

First, some prerequisites. Make sure you've done the following:
- [Install](https://www.scala-sbt.org/1.x/docs/Setup.html) sbt
- [Install](https://learn.hashicorp.com/terraform/getting-started/install.html) Terraform
- [Install](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) AWS CLI (Optional)

### Step 1) Configure Terraform

Let's create a `main.tf` file that references the generated terraform, using it as a [module](https://www.terraform.io/docs/configuration/modules.html). This is what we'll use to set up the [provider](https://www.terraform.io/docs/configuration/providers.html) (and maybe a [backend](https://www.terraform.io/docs/backends/index.html)?), which will allow us to communicate with the AWS API, and track our infrastructure's state. 

```hcl
# This block of terraform is a backend. It's totally optional, but highly recommended!
#
terraform {
  backend "s3" {
    # the region you'd like to store the state files in (not necessarily where your infrastructure will be provisioned)
    region               = "us-west-2"
    # replace the value with the name of an S3 Bucket
    bucket               = "<name of S3 bucket you'd like to place the state in>"
    # replace these two values with your project's name
    workspace_key_prefix = "<your-project-name>"
    key                  = "<your-project-name>.tfstate"

    encrypt = true
  }
}

# This block is the provider. It will be used by Scalambda to provision your infrastructure (i.e. the Lambda Functions)  
# for you. 
#
# Disclaimer: It will pull AWS credentials from whatever machine you run this terraform, so if you've setup the AWS CLI,
# you don't have to worry about providing it credentials or anything.
# 
provider "aws" {
  # This will be the region that your infrastructure is provisioned in. Replace it with your desired region if you want!
  region = "us-west-2"
}

# This will be what connects the two resources above to the Terraform you generated in step 1.
#
# Note: You can name this module whatever you would like! Just replace "my_lambda_functions" with whatever you'd like. 
#
module "my_lambda_functions" {
  # This should be the relative path to the terraform generated by Scalambda.
  #
  # In this example, this main.tf file will just be in the root of our project, so it will be in the same directory as 
  # our project's `src` and `target` folders. Which makes this source path really easy.
  source = "./target/terraform"

  # Based on how you configured your Lambda Functions in your build.sbt file, you may or may not need to provide some
  # inputs to this module. You can do that now if you already know what you need, or you can wait for Terraform to show
  # you what it needs via some nice error messages during the init/apply step (which is what's next on this guide).
}
```

**If you're at all confused**: You can generate a project with our [Giter8 Template](https://github.com/carpe/scalambda.g8/), or just check out at [the files here](https://github.com/carpe/scalambda.g8/tree/master/src/main/g8) to get an idea of what the layout looks like.

### Step 2) Apply the Terraform

Now all that's left to do is initialize and apply the Terraform. From the same directory as your `main.tf` file, run the following:

```bash
# Initialize the Terraform
terraform init

# Apply the Terraform
terraform apply
```

It's likely that after running `terraform init` or `terraform apply`, you will see some errors, don't worry! We're super close to being done anyway, and the errors should be fairly easy to fix. Check out the section below if you need help, or consider [opening an issue](https://github.com/carpe/scalambda/issues/new/choose) on our Github repository.

### Step 3) Done!

After you've fixed the errors and your Terraform has been applied, you should be able to see your newly provisioned Lambda Functions in the AWS Web Console. **For subsequent deploys**, all you need to do is run the following:

```bash
# Use Scalambda to generate the Terraform, which will
# recompile your code to get the latest changes
sbt scalambdaTerraform

# Apply the newly generated Terraform.
terraform apply
```

Since Terraform manages your infrastructure's state and performs incremental changes, this will be MUCH faster than the first deploy! Which means you can quickly and iteratively make changes to your code and test them out on AWS Lambda itself if you'd like.

### Common Problems/Solutions

**Problem:** Terraform is complaining about missing variables for the module we created \
**Solution:** Fill those missing variables in! You can find some sensible defaults for stuff like your function's role over on the [Giter8 Template](https://github.com/carpe/scalambda.g8/).

**Problem:** Terraform is complaining about missing AWS credentials \
**Solution:** If you have the aws cli installed, try running something simple like `aws s3 ls` to make sure your credentials are valid. If not, go ahead and [install the aws cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) as that will allow you to run `aws configure`. From our experience, it's both the fastest, and most effective way to solve this problem. Alternatively, you could also [check out the documentation](https://www.terraform.io/docs/providers/aws/index.html#authentication) on managing credentials for the AWS Terraform Provider. 


