# Bootstrapping your environment for use with AWS CDK

[Documentation Link](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping-env.html)

You can run `cdk bootstrap` from the parent directory of a CDK project containing a cdk.json file

Bootstrapping prepares your AWS environment for AWS Cloud Development Kit (AWS CDK) stack deployments by provisioning necessary resources like S3 buckets, IAM roles, and ECR repositories.

## How to Bootstrap Your Environment

### Method 1: Using CDK CLI (Recommended)

#### Bootstrap from Any Directory
```bash
# Basic bootstrap command
$ cdk bootstrap aws://123456789012/us-east-1

# Multiple environments
$ cdk bootstrap aws://123456789012/us-east-1 aws://123456789012/us-east-2

# Using named profile
$ cdk bootstrap --profile prod
```

#### Bootstrap from CDK Project Directory
```bash
# Bootstrap using default sources (config/credentials files)
$ cdk bootstrap

# Bootstrap with specific profile
$ cdk bootstrap --profile prod
```

## When to Bootstrap

- **Must bootstrap before deploying**: Each AWS environment must be bootstrapped before deployment
- **Proactive bootstrapping recommended**: Bootstrap environments before you need them to prevent issues
- **Safe to re-bootstrap**: Bootstrapping multiple times is safe and will upgrade if necessary

### Error if Not Bootstrapped
```bash
$ cdk deploy
❌ Deployment failed: Error: BootstrapExampleStack: SSM parameter /cdk-bootstrap/hnb659fds/version not found. 
Has the environment been bootstrapped? Please run 'cdk bootstrap'
```

## Default Resources Created

### IAM Roles
- **CloudFormationExecutionRole**: Service role for CloudFormation deployments
- **DeploymentActionRole**: Role for performing deployments (cross-account capable)
- **FilePublishingRole**: Role for S3 bucket operations
- **ImagePublishingRole**: Role for ECR repository operations  
- **LookupRole**: ReadOnly role for context value lookups

### Resource Naming Convention
Resources use the format: `cdk-<qualifier>-<description>-<account-ID>-<Region>`

Example: `cdk-hnb659fds-assets-012345678910-us-west-1`

Where:
- **Qualifier**: `hnb659fds` (fixed 9-character string)
- **Description**: Short resource description (e.g., `container-assets`)
- **Account ID**: AWS account ID
- **Region**: AWS region

## Required Permissions for Bootstrapping

The IAM identity performing bootstrapping needs these minimum permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*",
        "ecr:*",
        "ssm:*",
        "s3:*",
        "iam:*"
      ],
      "Resource": "*"
    }
  ]
}
```

## Troubleshooting

### CDK Pipelines Cross-Account Error
```
Policy contains a statement with one or more invalid principals
```
**Solution**: Bootstrap the target environment with appropriate IAM roles.