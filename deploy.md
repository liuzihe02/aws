# AWS CDK Deployment Guide
[Document Link](https://docs.aws.amazon.com/cdk/v2/guide/deploy.html)

## Basic CDK Deployment

### How AWS CDK Deployments Work

AWS CDK utilizes the **AWS CloudFormation service** to perform deployments. The process involves:

1. **Synthesis**: Creates CloudFormation templates and deployment artifacts for each CDK stack
2. **Asset Upload**: Assets are uploaded to bootstrapped resources (S3, ECR)
3. **CloudFormation Deployment**: Templates are submitted to CloudFormation to provision AWS resources

### App Lifecycle

When you perform synthesis, your CDK app goes through these phases:

1. **Construction (Initialization)**: Code instantiates all constructs and links them together
2. **Preparation**: Final round of modifications to set up final state
3. **Validation**: Constructs validate themselves to ensure correct deployment
4. **Synthesis**: Final stage that produces CloudFormation templates and deployment artifacts

### Key Commands

```bash
# Synthesize CloudFormation templates
cdk synth

# Deploy your application
cdk deploy

# Deploy specific stack
cdk deploy MyStackName

# Deploy multiple stacks
cdk deploy Stack1 Stack2
```

### Running Your App

The CDK CLI needs to know how to run your app via `cdk.json`:

```json
// TypeScript
{
  "app": "npx ts-node --prefer-ts-exts bin/my-app.ts"
}
```

## CDK Pipelines (CI/CD)

CDK Pipelines provide **self-updating** continuous delivery of AWS CDK applications. Key features:

- Automatically build, test, and deploy new versions from source control
- Self-mutating: pipeline reconfigures itself when you add new stages or stacks
- Supports GitHub, CodeCommit, and CodeStar as source repositories

### Bootstrap Requirements

Pipeline environments require additional resources:
- Amazon S3 bucket for artifacts
- Amazon ECR repository for container images
- IAM roles for pipeline permissions

## Prerequisites

### 1. Configure Security Credentials

Set up AWS credentials for the CDK CLI. See AWS documentation for credential configuration.

### 2. Bootstrap Your AWS Environment

Bootstrap provisions resources needed for CDK deployments:

```bash
# Basic bootstrapping
cdk bootstrap

# Bootstrap for pipelines with admin access
npx cdk bootstrap aws://ACCOUNT-NUMBER/REGION --profile ADMIN-PROFILE \
    --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess

# Bootstrap additional environments for pipeline deployment
npx cdk bootstrap aws://ACCOUNT-NUMBER/REGION --profile ADMIN-PROFILE \
    --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess \
    --trust PIPELINE-ACCOUNT-NUMBER
```

### 3. Enable Termination Protection

Protect your bootstrap stack from accidental deletion:

```bash
cdk bootstrap --termination-protection
```

### 4. Configure AWS Environments

Each CDK stack must be associated with an environment (account + region).

## Code Examples


### Pipeline Definition

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { CodePipeline, CodePipelineSource, ShellStep } from 'aws-cdk-lib/pipelines';

export class MyPipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const pipeline = new CodePipeline(this, 'Pipeline', {
      pipelineName: 'MyPipeline',
      synth: new ShellStep('Synth', {
        input: CodePipelineSource.gitHub('OWNER/REPO', 'main'),
        commands: ['npm ci', 'npm run build', 'npx cdk synth']
      })
    });
  }
}
```

### Application Stage Definition

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from "constructs";
import { MyLambdaStack } from './my-lambda-stack';

export class MyPipelineAppStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props?: cdk.StageProps) {
    super(scope, id, props);
    
    const lambdaStack = new MyLambdaStack(this, 'LambdaStack');
  }
}
```

### Lambda Stack Example

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { Function, InlineCode, Runtime } from 'aws-cdk-lib/aws-lambda';

export class MyLambdaStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new Function(this, 'LambdaFunction', {
      runtime: Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: new InlineCode('exports.handler = _ => "Hello, CDK";')
    });
  }
}
```

### Adding Stages to Pipeline

```typescript
// Add stage to pipeline
pipeline.addStage(new MyPipelineAppStage(this, "test", {
  env: { account: "111111111111", region: "eu-west-1" }
}));

// Add manual approval step
const testingStage = pipeline.addStage(new MyPipelineAppStage(this, 'testing', {
  env: { account: '111111111111', region: 'eu-west-1' }
}));

testingStage.addPost(new ManualApprovalStep('approval'));
```

### Parallel Deployment with Waves

```typescript
const wave = pipeline.addWave('wave');
wave.addStage(new MyApplicationStage(this, 'MyAppEU', {
  env: { account: '111111111111', region: 'eu-west-1' }
}));
wave.addStage(new MyApplicationStage(this, 'MyAppUS', {
  env: { account: '111111111111', region: 'us-west-1' }
}));
```

### Testing Deployments

```typescript
// Simple validation
stage.addPost(new ShellStep("validate", {
  commands: ['./tests/validate.sh'],
}));

// Using CloudFormation outputs
this.loadBalancerAddress = new cdk.CfnOutput(lbStack, 'LbAddress', {
  value: `https://${lbStack.loadBalancer.loadBalancerDnsName}/`
});

stage.addPost(new ShellStep("lbaddr", {
  envFromCfnOutputs: {lb_addr: lbStack.loadBalancerAddress},
  commands: ['echo $lb_addr']
}));
```

### Pipeline Initialization

```typescript
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { MyPipelineStack } from '../lib/my-pipeline-stack';

const app = new cdk.App();
new MyPipelineStack(app, 'MyPipelineStack', {
  env: {
    account: '111111111111',
    region: 'eu-west-1',
  }
});

app.synth();
```