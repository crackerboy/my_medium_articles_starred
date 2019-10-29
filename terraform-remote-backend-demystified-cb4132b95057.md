Unknown markup type 10 { type: [33m10[39m, start: [33m151[39m, end: [33m168[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m82[39m, end: [33m91[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m339[39m, end: [33m353[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m63[39m, end: [33m121[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m36[39m, end: [33m87[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m127[39m, end: [33m143[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m173[39m, end: [33m210[39m }
Unknown markup type 10 { type: [33m10[39m, start: [33m32[39m, end: [33m46[39m }

# Terraform Remote Backend Demystified

Terraform Remote Backend Demystified

### A quick start guide to explain Terraform backends and how your deployments can benefit from a non-local state file storage.

***You are reading this because:**
 — you already know what [Terraform](https://www.terraform.io/) is, and your knowledge of it wedges somewhere in the middle between beginner and intermediate.*

## About Terraform

The concept behind the *Infrastructure as Code* has narrowed the distance between developers, architects, systems, and network administrators.

**Today this sounds already like a tiresome, old mantra.**

With a terminal, a few bytes of code and **no ssh logins**, we can spin up, in minutes, a production-ready Kubernetes cluster, capable of heavy workloads.

So here it comes to **Terraform**: a potent tool written in Go, that helps us writing, planning, creating highly **predictable** and **stateful** Infrastructures as Code.

## Stage 1: Terraform State

Terraform keeps the **state** of the managed infrastructure and its configuration. By default, the state is stored locally, in a JSON formatted file named terraform.tfstate. From the first lift to the latest change of our infrastructure’s plan, every chunk of data populates the state file, allowing the mapping of real-world resources to the terraform configuration and to keep track of metadata. A feature that brings enormous benefits and opens to scenarios like versioning, debugging, performance monitoring, rollbacks, rolling updates, immutable deployments, traceability, self-healing capabilities, among others.

### *Let’s scratch the surface to see how it works*

Terraform uses the **state** to create **plans,** which are the representation of the resources we are deploying and the changes we are applying.

Before any operation, a **refresh** is performed to synchronize the **state** with the running resources, if any. Hence, the state file contains sensitive information and plays a crucial role in the entire management process of our infrastructure. Trivial topics then, like “how to store the state,” “where to store state,” “how to secure it,” should be addressed paying the right amount of attention.

In a wide range of scenarios, the default strategy to store the state file locally is far from being a good idea.
> # *What if the hard drive gets corrupted?*
> # *What if a breach happens?*
> # *What if I am working in a team?*

It is enough running a small pipeline within a personal project to make the local state a loser strategy.
What determines how the state is stored and loaded? How do we modify the default behavior? The answer is “Backend.”

## Stage 2: Terraform Backends

A “backend” in Terraform is an abstraction that determines the handling of the state and the way certain operations are executed, enabling many essential features.
Let’s walk through the gifts we get out of the box when using a remote backend.
 
***Backends can store the state remotely and protect it with locks to prevent corruption;*** it makes it possible for a team to work with ease, or, for instance, to run Terraform within a pipeline.

***Better protection for sensitive data**. *When the state is retrieved from a backend, it gets stored in memory and never persisted on a local drive. Even though the memory is susceptible to exploits, this makes it certainly more complicated, for bad actors, to exfiltrate or corrupt the content held within the state. However, a misconfiguration of the remote storage can equally lead to breaches.

***Remote operations**:* For large infrastructures or when pushing specific changes, terraform apply can take quite a long time. Typically, for the entire duration of the deployment, it is not possible to disconnect the client without compromising its execution. Some of the backends available in terraform however, can be delegated to execute remote operations, allowing the client to go offline safely. Along with remote state storage and locking, remote operations are a big help for teams and more structured environments.

## Stage 3: Configuration

Backends configuration resides in Terraform files with the HLC syntax, within the terraform section. The following, simplified, snippet shows how a remote backend can be enabled leveraging an AWS s3 bucket, where the *terraform.tfstate* persists.

    terraform {  
        backend "s3" {
            bucket = "<your_bucket_name>"
            key    = "terraform.tfstate"    
            region = "<your_aws_region>"  
        }
    }

When configuring a backend rather than the default for the first time, Terraform provides the option to migrate the current state to the new backend, in order to transfer the existing data and not losing any information.
It is recommended, though not mandatory, to manually back up the state before initializing any new backend by running terraform init. It is enough to copy the state file outside the scope of the project, in a different folder. The initialization process should create a backup as well.

Now it is time for coding, and demonstrate how to set up a remote backend, with a real-life example.

## Stage 4: The AWS s3 Backend

Setting up a Terraform backend it is relatively easy. Let’s see how to implement one with AWS s3.
To run the code of the example, be sure to have available AWS IAM credentials with enough permissions to create/ delete s3 buckets and put bucket policies.
> This example leverages the **aws cli**, assuming it is already installed and configured. To set it up, please refer to the official online documentation at [this link](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).

Let’s create a bucket, for instance in the region *eu-west-1* *(EU — Ireland),* named *terraform-backend-store. *To do so, open a terminal and run the following command:

    aws s3api create-bucket --bucket terraform-backend-store \
        --region eu-west-1 \
        --create-bucket-configuration \
        LocationConstraint=eu-west-1

    # Output:
    {
        "Location": "[http://terraform-backend-store.s3.amazonaws.com/](http://terraform-backend-store.s3.amazonaws.com/)"
    }

Once the bucket is in place, it needs a proper configuration.

For a bucket that holds the Terraform state, it’s a *good idea* to enable the server-side encryption. To do so, and keeping it simple, let’s get back to the terminal and set the server-side encryption to AES256 (*Although it’s out of scope for this story, I recommend to use the kms and implement a proper key rotation)*:

    aws s3api put-bucket-encryption \
        --bucket terraform-backend-store \
        --server-side-encryption-configuration={\"Rules\":[{\"ApplyServerSideEncryptionByDefault\":{\"SSEAlgorithm\":\"AES256\"}}]}

    # Output: expect none when the command is executed successfully

Next, it’s important to restrict the access to the bucket.
Create an unprivileged IAM user:

    aws iam create-user --user-name terraform-deployer

    # Output:
    **{**
        "User"**:** **{**
            "UserName"**:** "terraform-deployer"**,**
            "Path"**:** "/"**,**
            "CreateDate"**:** "2019-01-27T03:20:41.270Z"**,**
            "UserId"**:** "AIDAIOSFODNN7EXAMPLE"**,**
            "Arn"**:** "arn:aws:iam::123456789012:user/terraform-deployer"
        **}**
    **}**

Take note of the **Arn** from the command’s output (it looks like: “Arn”**:** “arn:aws:iam::123456789012:user/terraform-deployer”).

To properly interact with the s3 service and DynamoDB at a later stage, our IAM user must hold a sufficient set of permissions. It is recommended to have **severe restrictions** in place in production environments, though, for the sake of simplicity, in this example, we generously assign *AmazonS3FullAccess *and* AmazonDynamoDBFullAccess*:

    aws iam attach-user-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --user-name terraform-deployer

    # Output: expect none when the command execution is successful

    aws iam attach-user-policy --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess --user-name terraform-deployer

    # Output: expect none when the command execution is successful

Back to our bucket now. The freshly created IAM user must be allowed to execute the necessary actions, and have the terraform backend running smoothly.
Let’s create a policy file as follows:

    cat <<-EOF >> policy.json
    {
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::123456789012:user/terraform-deployer"
                },
                "Action": "s3:*",
                "Resource": "arn:aws:s3:::terraform-remote-store"
            }
        ]
    }
    EOF

It grants to the principal with arn* *“arn:aws:iam::123456789012:user/terraform-deployer”, to execute all the available actions (“Action”: “s3:*") against the bucket with arn* *“arn:aws:s3:::terraform-remote-store”. Again, in production is desired to force way stricter policies. For reference, have a look to the [AWS Policy Generator](https://awspolicygen.s3.amazonaws.com/policygen.html).
Back to the terminal and run the command as shown below, to enforce the policy in our bucket:

    aws s3api put-bucket-policy --bucket terraform-remote-store --policy file://policy.json

    # Output: none

As the last step, we enable the bucket’s versioning:

    aws s3api put-bucket-versioning --bucket terraform-remote-store --versioning-configuration Status=Enabled

It allows saving different versions of the infrastructure’s state and rollback easily to a previous stage without struggling.

The AWS s3 bucket is ready, time to integrate it with Terraform. Listed below, is the minimal configuration required to set up this remote backend:

    # terraform.tf

    terraform {  
        backend "s3" {
            bucket  = "terraform-remote-store"
            encrypt = true
            key     = "terraform.tfstate"    
            region  = "eu-west-1"  
        }
    }

    ...

    # the rest of your configuration and resources to deploy

Once in place, terraform must be initialised *(again)*.

    terraform init

The remote backend is ready for a ride, test it.

### What about locking?

Storing the state remotely brings a pitfall, especially when working in scenarios where several tasks, jobs, and team members have access to it. Under these circumstances, the risk of multiple concurrent attempts to make changes to the state, is high. Here comes to help the **lock**, a feature that prevents opening the state file while already in use.

Back to our example, the lock can be implemented by creating an **AWS DynamoDB Table**, used by terraform to set and unset the locks.

We can create the table resource using terraform itself:

    # create-dynamodb-lock-table.tf

    resource "aws_dynamodb_table" "dynamodb-terraform-state-lock" {
      name           = "terraform-state-lock-dynamo"
      hash_key       = "LockID"
      read_capacity  = 20
      write_capacity = 20

    attribute {
        name = "LockID"
        type = "S"
      }

    tags {
        Name = "DynamoDB Terraform State Lock Table"
      }
    }

and deploy it:

    terraform plan -out "planfile" ; terraform apply -input=false -auto-approve "planfile"

Once the command execution is completed, the locking mechanism must be added to our backend configuration as follow:

    # terraform.tf

    terraform {  
        backend "s3" {
            bucket         = "terraform-remote-store"
            encrypt        = true
            key            = "terraform.tfstate"    
            region         = "eu-west-1"
            dynamodb_table = "terraform-state-lock-dynamo"
        }
    }

    ...

    # the rest of your configuration and resources to deploy

All done. Remember to run again terraform init and enjoy the remote backend’s features and joys.

~ Till the next time.

![](https://cdn-images-1.medium.com/max/2000/0*Piks8Tu6xUYpF4DU)

**Join our community Slack and read our weekly Faun topics ⬇**

![](https://cdn-images-1.medium.com/max/3200/0*oSdFkACJxs5iy1oR)

### If this post was helpful, please click the clap 👏 button below a few times to show your support for the author! ⬇
