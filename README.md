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
To demonstrate, we will create a tradititional AWS S3 lock with DynamoDB, then after initializing the backend using the traditional method, we will go ahead and switch to the native S3 bucket locking. Feel free to skip to the S3 Native locking if you already have a traditional backend resource you want to practice on. Remember to back up files when working on critical files and resources. Always test before implementing!

In this example, we will be using terraform code that created an AWS key pair. You might want to create a folder to organize your files. 

Create the folder **backend**

```
mkdir backend
```

Create terraform files inside your new folder

```
cd backend
```

```
touch backend.tf provider.tf keypair.tf variable.tf output.tf
```

```
This is the text that users can copy with one click.
It can span multiple lines.
```


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
