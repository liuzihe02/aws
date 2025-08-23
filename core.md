# AWS CDK

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