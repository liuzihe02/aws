# AWS CDK

Most of the below are summarized from the [core concepts documentation](https://docs.aws.amazon.com/cdk/v2/guide/core-concepts.html)

- AWS CDK Construct Library – A collection of pre-written modular and reusable pieces of code, called constructs

- AWS CDK Toolkit - Tools that you can use to manage and interact with your CDK apps, such as performing synthesis or deployment. The CDK Toolkit consists of a command line tool (CDK CLI) and a programmatic library (CDK Toolkit Library)

- AWS Toolkit Library - `@aws-cdk/toolkit-lib` which allows you to do CDK actions (synthesis, deploy etc) via code instead of using CLI commands

## AWS CDK Library

The AWS CDK library is `aws-cdk-lib`. The constructs library is `constructs` which is a library for defining and composing cloud infra components

## AWS CDK Apps

The AWS CDK application or *app* is a collection of one or more CDK stacks. Stacks are a collection of one or more constructs, which define AWS resources and properties. Therefore, the overall grouping of your stacks and constructs are known as your CDK app.

### How to Create a CDK App

You create an app by defining an app instance in the application file of your project. To do this, you import and use the `App` construct from the AWS Construct Library. The `App` construct doesn't require any initialization arguments. It is the only construct that can be used as the **root**

The `App` and `Stack` classes from the AWS Construct Library are unique constructs. They don't configure AWS resources on their own. Instead, they are used to provide context for your other constructs. All constructs that represent AWS resources must be defined, directly or indirectly, within the scope of a `Stack` construct. `Stack` constructs are defined within the scope of an `App` construct.

Basic app creation example:
```ts
const app = new App();
new MyFirstStack(app, 'hello-cdk');
app.synth();
```

Apps are synthesized to create AWS CloudFormation templates for your stacks. Stacks within a single app can easily refer to each other's resources and properties. The AWS CDK infers dependencies between stacks so that they can be deployed in the correct order. You can deploy any or all of the stacks within an app with a single `cdk deploy` command.

### The Construct Tree

Constructs are defined inside of other constructs using the `scope` argument that is passed to every construct, with the `App` class as the root. This creates a hierarchy of constructs known as the *construct tree*.

Constructs are always explicitly defined within the scope of another construct, which creates relationships between constructs. You should typically pass `this` (in Python, `self`) as the scope, indicating that the new construct is a child of the current construct.

Key properties of the construct tree:
- `node.children` – The direct children of the construct
- `node.id` – The identifier of the construct within its scope
- `node.path` – The full path of the construct including the IDs of all of its parents
- `node.scope` – The scope (parent) of the construct, or undefined if the node is the root

## Stacks

- An AWS CDK stack is the smallest single unit of deployment
- It represents a collection of AWS resources that you define using CDK constructs
- When you deploy CDK apps, the resources within a CDK stack are deployed together as an AWS CloudFormation stack

Here, we extend or inherit the Stack class and define a constructor that accepts `scope`, `id`, and `props`

```ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { MyCdkStack } from '../lib/my-cdk-stack';

const app = new cdk.App();
new MyCdkStack(app, 'MyCdkStack', {
});
// a second stack
new MySecondStack(app, 'stack2');
app.synth();
```

```ts
import { App, Stack } from 'aws-cdk-lib';
import { Construct } from 'constructs';

interface EnvProps {
  prod: boolean;
}

// imagine these stacks declare a bunch of related resources
class ControlPlane extends Stack {}
class DataPlane extends Stack {}
class Monitoring extends Stack {}

//each environment has a set of stacks
class MyService extends Construct {

  constructor(scope: Construct, id: string, props?: EnvProps) {

    super(scope, id);

    // we might use the prod argument to change how the service is configured
    new ControlPlane(this, "cp");
    new DataPlane(this, "data");
    new Monitoring(this, "mon");  }
}

const app = new App();
new MyService(app, "beta");
new MyService(app, "prod", { prod: true });

app.synth();
```

will produce 6 stacks, 3 for each environment:
```
$ cdk ls

betacpDA8372D3
betadataE23DB2BA
betamon632BD457
prodcp187264CE
proddataF7378CE5
prodmon631A1083
```

## Stages

An AWS Cloud Development Kit (AWS CDK) stage represents a group of one or more CDK stacks that are configured to deploy together. Use stages to deploy the *same grouping of stacks* to multiple environments, such as development, testing, and production.

Use the `Stage` construct to do this:

` cdk-demo-app/lib/my-stage.ts`
```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { Stage } from 'aws-cdk-lib';
import { AppStack } from './app-stack';
import { DatabaseStack } from './database-stack';

// Define the stage - the id will determine the naming of the stacks
export class MyAppStage extends Stage {
  constructor(scope: Construct, id: string, props?: cdk.StageProps) {
    super(scope, id, props);

    // Add both stacks to the stage
    new AppStack(this, 'AppStack');
    new DatabaseStack(this, 'DatabaseStack');
  }
}
```
`cdk-demo-app/bin/cdk-demo-app.ts`
```ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { MyAppStage } from '../lib/my-stage';

// Create a CDK app
const app = new cdk.App();

// Create the development stage
new MyAppStage(app, 'Dev', {
  env: {
    account: '123456789012',
    region: 'us-east-1'
  }
});

// Create the production stage
new MyAppStage(app, 'Prod', {
  env: {
    account: '098765432109',
    region: 'us-east-1'
  }
});
```

When we run `cdk synth`, the 2 cloud assemblies (one for each stage) are then created in `cdk.out`

## Constructs

Constructs are the basic building blocks of AWS Cloud Development Kit (AWS CDK) applications. A construct is a component within your application that represents one or more AWS CloudFormation resources and their configuration.

Construct levels:
- L1 constructs (aka CFN resources) are the lowest level constructs and offer no abstration
  - each L1 constructs maps to *one* CloudFormation resouirce
  - L1 constructs are named starting with `Cfn` like `CfnBucket`
- L2 constructs are curated constructs that map to *one* CloudFormation resource, but has a intuitive intent based API
  - already includes alot of boilerplate code
  - e..g. `s3.Bucket`
- L3 constructs are patterns that have the highest level of abstraction

Note that the `App` and `Stack` objects are also constructs that provide context for other constructs (they dont confugre AWS resources on their own). All constructs (whether indirectly or directly) must be defined within the scope of a `Stack` construct

### Defining Constructs

**Composition** is used where a high level construct can be composed from a bunch of lower-level constructs (not using inheritence)

Instantiating a construct involves:
- `scope` which is usually `this` or `self` which refers to the parent, as you are creating a child construct within the parent definition
- `id` which helps as namespace and forming the final id of stuff
- `props` optional config object

here's an example writing your own construct:
```ts
// we extend the construct object rather than inheriting from the `Bucket` class. this is composition not inheritance
export class NotifyingBucket extends Construct {
  public readonly topic: sns.Topic;

  constructor(scope: Construct, id: string, props: NotifyingBucketProps) {
    super(scope, id);
    const bucket = new s3.Bucket(this, 'bucket');
    this.topic = new sns.Topic(this, 'topic');
    bucket.addObjectCreatedNotification(new s3notify.SnsDestination(this.topic), { prefix: props.prefix });
  }
}
```

## Environments

An environment consists of the AWS account and AWS Region that you deploy an AWS Cloud Development Kit (AWS CDK) stack to.

```ts
const envEU = { account: '238383838383', region: 'eu-west-1' };
const envUSA = { account: '837873873873', region: 'us-west-2' };

new MyFirstStack(app, 'first-stack-us', { env: envUSA });
new MyFirstStack(app, 'first-stack-eu', { env: envEU });
```

## Bootstrapping

Bootstrapping is the process of preparing your AWS environment for usage with the AWS Cloud Development Kit (AWS CDK). Before you deploy a CDK stack into an AWS environment, the environment must first be bootstrapped (note that each CDK environment is independent)

Bootstrapping prepares your AWS environments for deployment of your CDK resources, e.g.
- AWS S3 bucket to store your CDK code
- AWS ECR to store Docker images
- IAM roles

Use the `cdk bootstrap` command to do this

## Resources

Resources are what you configure to use AWS services in your applications. Resources are a feature of AWS CloudFormation. By configuring resources and their properties in a AWS CloudFormation template, you can deploy to AWS CloudFormation to provision your resources. With the AWS Cloud Development Kit (AWS CDK), you can configure resources through constructs. You then deploy your CDK app, which involves synthesizing a AWS CloudFormation template and deploying to AWS CloudFormation to provision your resources.

To configure instances of resources, pass in the scope as the first argument, the logical ID of the construct, and a set of configuration properties (props).

### Referencing Resources

- By passing a resource defined in your CDK app, either in the same stack or in a different one

```ts
const cluster = new ecs.Cluster(this, 'Cluster', { /*...*/ });
const service = new ecs.Ec2Service(this, 'Service', { cluster: cluster });
```
referencing resources in a different stack
```ts
const prod = { account: '123456789012', region: 'us-east-1' };

const stack1 = new StackThatProvidesABucket(app, 'Stack1', { env: prod });

// stack2 will take a property { bucket: IBucket }
const stack2 = new StackThatExpectsABucket(app, 'Stack2', {
  bucket: stack1.bucket,
  env: prod
});
```


By passing a proxy object referencing a resource defined in your AWS account, created from a unique identifier of the resource (such as an ARN). Maybe you want to use a resource already available in your AWS CDK app
```ts
// Construct a proxy for a bucket by its name (must be same account)
s3.Bucket.fromBucketName(this, 'MyBucket', 'amzn-s3-demo-bucket1');

// Construct a proxy for a bucket by its full ARN (can be another account)
s3.Bucket.fromBucketArn(this, 'MyBucket', 'arn:aws:s3:::amzn-s3-demo-bucket1');

// Construct a proxy for an existing VPC from its attribute(s)
ec2.Vpc.fromVpcAttributes(this, 'MyVpc', {
  vpcId: 'vpc-1234567890abcde',
});
```
 the VPC construct has a `fromLookup` static method (Python: from_lookup) that lets you look up the desired Amazon VPC by querying your AWS account at synthesis time.
 ```ts
 ec2.Vpc.fromLookup(this, 'DefaultVpc', {
  isDefault: true
});
```

## Identifiers

- Construct IDs
  - This is the `id` argument that only needs to be unqiue within the `scope`
  - so multiple stacks (different scopes) can have the same `id`
- Paths
  - We refer to the collection of IDs from a given construct, its parent construct, its grandparent, and so on to the root of the construct tree, as a path.
  - `const path: string = myConstruct.node.path;`

## Context Values

Context values are key-value pairs that can be associated with an app, stack, or construct. They may be supplied to your app from a file (usually either cdk.json or cdk.context.json in your project directory) or on the command line.

Because these values are provided by your AWS account, they can change between runs of your CDK application. This makes them a potential source of unintended change. The CDK Toolkit’s caching behavior "freezes" these values for your CDK app until you decide to accept the new values.

> Cached context values are managed by the AWS CDK and its constructs, including constructs you may write. Do not add or change cached context values by manually editing files. It can be useful, however, to review cdk.context.json occasionally to see what values are being cached. Context values that don’t represent cached values should be stored under the context key of cdk.json. This way, they won’t be cleared when cached values are cleared.

Context Key: `availability-zones:account=123456789012:region=eu-central-1`

### Sources of Context Values

Context values can be provided to your AWS CDK app in six different ways:

1. Automatically from the current AWS account.

2. Through the --context option to the cdk command. (These values are always strings.)

3. In the project’s cdk.context.json file.

4. In the context key of the project’s cdk.json file.

5. In the context key of your ~/.cdk.json file.

6. In your AWS CDK app using the construct.node.setContext() method.

> Because they’re part of your application’s state, cdk.json and cdk.context.json must be committed to source control along with the rest of your app’s source code. Otherwise, deployments in other environments (for example, a CI pipeline) might produce inconsistent results.

Use the `cdk context` command to view and manage the information in your `cdk.context.json` file.
```ts
Context found in cdk.json:

┌───┬─────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────┐
│ # │ Key                                                         │ Value                                                   │
├───┼─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ 1 │ availability-zones:account=123456789012:region=eu-central-1 │ [ "eu-central-1a", "eu-central-1b", "eu-central-1c" ]   │
├───┼─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
│ 2 │ availability-zones:account=123456789012:region=eu-west-1    │ [ "eu-west-1a", "eu-west-1b", "eu-west-1c" ]            │
└───┴─────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────┘

Run

  cdk context --reset KEY_OR_NUMBER

 to remove a context key. If it is a cached value, it will be refreshed on the next

  cdk synth

```

To use `--context` to pass runtime context values to your CDK app during synthesis or deployment:
```bash
cdk synth --context key1=value1 --context key2=value2 MyStack
```
Context values passed from the command line are always strings