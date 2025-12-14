# Simplifying Terraform State Management: Migrating from DynamoDB to S3 Native Locking

If you've been managing cloud infrastructure with Terraform, you're probably familiar with the classic AWS backend setup: an S3 bucket for state storage and a DynamoDB table for state locking.

## Why is State Management important in Terraform?

The state file captures a record of resources, tracks resources  and enables Terraform to manage deployed resources. It is important to store the state file in a safe place preferably with a functionalty to enables versioning. This way, anytime Terraform needs to update or configure resources, it has a reference for what is already deplyed (current state) and the desired/requested state.

Storing terraform in a centralized location enables team access to the files and improves collaboration. While this is desirable, what happens when multiple team members are woring on the same file at the same time? If this happens, there is a chance of infrastructure deployment conflicts. This is why it is preferable lock the state file anytime it is in use.


## The Traditional Approach: S3 + DynamoDB

For years, the standard Terraform backend configuration looked something like this:
```
terraform {
  backend "s3" {
    bucket         = "magnolia-radish"
    key            = "staging/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "magnolia"
  }
}
```
This approach comes with some overhead:
- **Extra infrastructure to maintain**: You need to provision and manage a DynamoDB table
- **Additional costs**: Even with on-demand pricing, you're paying for two services - S3 bucket and DynamoDB table
- **More complexity**: Extra IAM permissions, monitoring, and potential failure points

**Most importantly, the traditional way of locking state files on AWS S3 bucket is being deprecated so this is a good time to start making the switch!!!**

## Enter S3 Native Locking 
**S3 now offers native locking capabilities that can simplify your infrastructure**.

In this quick refresh, we go over steps to migrate existing terraform backend scripts over to the simplified native S3 backend locking. We will also validate our update to make sure it is indeed working.

S3 introduced native state locking support that eliminates the need for DynamoDB entirely. Instead of using a separate database, S3 leverages its own native object locking mechanisms to prevent concurrent state modifications.

The new configuration is refreshingly simple:

![S3 bucket backend](images/backend.tf.png)

Here is the code:

```
terraform {
  backend "s3" {
    bucket         = "magnolia-radish"
    key            = "staging/terraform.tfstate"
    region         = "us-east-1"
    use_lockfile = true
  }
}
```
## Steps:

Note: If you already have all files, feel free to skip ahead. In summary, the S3 + DynamoDB backend is the old method (soon to be deprecated). The S3 with use_lockfile = true argument is the new method. 

For step by step, we will create terraform files including a tradititional AWS S3 lock with DynamoDB backend. After initializing the backend using the traditional method, we will go ahead and switch to the native S3 bucket backend locking. Feel free to skip to the S3 Native locking if you already have a traditional backend resource you want to practice on. Remember to back up files when working on critical files and resources. Always test before implementing!

In this example, we will be using terraform code to create an AWS key pair. 

Create the folder **backend**

```
mkdir backend
```

Create terraform files inside your new folder

```
cd backend
```

```
touch provider.tf keypair.tf variable.tf output.tf
```
Add terraform code to keypair.tf

```
cat <<'EOF' > keypair.tf
resource "tls_private_key" "magnolia-key" {
  algorithm = "RSA"
  rsa_bits = 2048
}

resource "aws_key_pair" "magnolia-key-public" {
  key_name = var.key_name
  public_key = tls_private_key.magnolia-key.public_key_openssh
}

resource "local_file" "magnolia-key-private" {
  filename = "${var.key_name}.pem"
  file_permission = "0700"
  content = tls_private_key.magnolia-key.private_key_pem
}
EOF

```
Add terraform code to provider.tf

```
cat <<EOF > provider.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  profile = var.profile
  region  = var.region
}
EOF
```
Add terraform code to output.tf

```
cat <<EOF > output.tf
output "key_name" {
  value = aws_key_pair.magnolia-key-public.key_name
}
EOF
```
Add terraform code to variable.tf

```
cat <<EOF > variable.tf
variable "region" {
  default = "us-east-1"
}

variable "profile" {
  default = "default"
}

variable "key_name" {
  default = "magnolia-radish"
}

EOF

```
This code will add a public portion of the keypair in your AWS account and download a private portion to your local file. Feel free to adjust the variable file as you see fit.

**Now, we have to create the S3 bucket and the DynamoDB table to use for state file locking.**
Although Terraform can automate provision of resources in AWS, we have to first set up resources related to the backend before running Terraform. This is because the backend needs to be set up and initialized prior to resource creation stage.

Create S3 bucket, enable versioning and block public access. Update backend code with unique S3 bucket name.

![Create S3 bucket](images/create%20s3%20bucket%201.png)

Create a folder "staging" inside your bucket. Confirm folder name is same as in your code under "key" argument - see backend code below.

![Create folder under your S3 bucket](images/create%20s3%20bucket%20folder.png)

![Create folder under your S3 bucket](images/create%20s3%20bucket%20folder%202.png)

Create DynamoDB table "magnolia" . Partition key is **LockID**

![DynamoDB table](images/create%20dynamoDB%20table.png)


To create a backend block file with the traditional method (for practice) 

```
touch old_backend.tf

cat <<EOF > old_backend.tf
terraform {
  backend "s3" {
    bucket         = "magnolia-radish" #replace with your unique bucket name
    key            = "staging/terraform.tfstate" #create a folder named "staging" in your S3 bucket
    region         = "us-east-1"
    dynamodb_table = "magnolia" #replace with your dynamoDB table name
  }
}
EOF
```

**Let's Go!!**

```
# initialize backend using the traditional DynamoDB table lock method
terraform init
```
```
# to see resources - 3 to be created 
terraform plan
```
To create the AWS keypair and download the private pair to your local directory

```
terraform apply
```
Review output and "yes" to agree

Now, we have a state file saved on S3 bucket with a lock through the traditional DynamoDB table.

For this project, we want to update the state file lock to switch to Native S3 lock. Create a backend state file as below after deactivating the old_backend file

To replace old_backend.tf with new backend for S3 Native block, use code below. You can also adjust your old_backend file to remove dynamoDB agrument line and replace with "use_lockfile = true"

```
mv old_backend.tf backend.tf

cat <<EOF > backend.tf
terraform {
  backend "s3" {
    bucket         = "magnolia-radish" #replace with your unique bucket name
    key            = "staging/terraform.tfstate" #confirm you have a folder named "staging" in your S3 bucket
    region         = "us-east-1"
    use_lockfile   = true
  }
}
EOF
```
Remember to update bucket name and double check key/path as required.

To configure terraform to switch to the new state file, we have to re-initialize the backend

```
terraform init -migrate-state
```

This command tells Terraform that the backend configuration has changed and that it should migrate the existing state to the newly configured backend. The lock has now been switched to Native S3 locking.

![Terraform init migrate state](images/migrate_state.png)

```
terraform apply
```
"yes" to agree


To confirm, while running a Terraform apply, you can have a look at the AWS S3 bucket folder and should see the lock file appear briefly while the code is being applied.

![verify S3 backend lock](images/s3%20bucket%20lock%20verification.png)


## Key Benefits of S3 Native Locking

**Simplified Architecture**: One less service to manage means reduced operational overhead. Your state management becomes a single-service concern.

**Cost Optimization**: Eliminating DynamoDB removes those read/write charges. For teams with multiple state files, this can add up to meaningful savings.

**Reduced IAM Complexity**: Fewer services mean simpler permission structures. You only need S3 permissions, which makes security audits and troubleshooting easier.

**Better Performance**: No external service calls to DynamoDB means slightly faster lock acquisition and release.

## Migration Considerations

Before you make the switch, keep these points in mind:

**Terraform Version**: Ensure you're running a version that supports S3 native locking. Check the Terraform changelog for your specific version.

**State File Migration**: The actual state data doesn't change, but you'll need to update your backend configuration across all repositories and environments.

**Team Coordination**: Ensure all team members upgrade their Terraform versions before switching, as older versions won't recognize the new locking mechanism.

**Testing**: Start with non-critical environments. Test your CI/CD pipelines, ensure locks work correctly, and verify that concurrent operations are properly blocked.

## Making the Switch

Here's a practical migration approach:

1. **Audit your current setup**: Document all state files and their DynamoDB tables
2. **Update Terraform versions**: Ensure all team members and CI/CD systems are compatible
3. **Update backend configurations**: Roll out the new config systematically, starting with dev environments
4. **Monitor for issues**: Watch for any locking problems during the transition period
5. **Deprecate DynamoDB**: Once confident, you can safely remove the DynamoDB tables

## The Bottom Line

While the DynamoDB approach served us well, S3 native locking represents a step toward simpler, more cost-effective infrastructure management. It's not just about reducing complexity—it's about adopting patterns that scale efficiently as your infrastructure grows.

For teams managing dozens or hundreds of state files, this migration can meaningfully reduce both operational burden and AWS costs. If you haven't explored S3 native locking yet, it's worth evaluating for your next infrastructure refresh.

---

*Have you made this migration already? I'd love to hear about your experience in the comments—what challenges did you encounter, and what benefits have you realized?*

