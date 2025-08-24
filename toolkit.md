# AWS CDK Toolkit Library

## Overview

The AWS CDK Toolkit Library enables programmatic CDK actions through code instead of CLI commands. It allows you to create custom tools, build specialized CLI applications, and integrate CDK capabilities into development workflows.

## Key Features

### Core Actions
- **Synthesis** - Generate CloudFormation templates and deployment artifacts
- **Deployment** - Provision or update infrastructure using CloudFormation templates  
- **List** - View information about stacks and their dependencies
- **Watch** - Monitor CDK apps for local changes
- **Rollback** - Return stacks to their last stable state
- **Destroy** - Remove CDK stacks and associated resources

### Benefits
- Control through code - Integrate infrastructure management into applications
- Manage cloud assemblies - Create, inspect, and transform infrastructure definitions
- Customize deployments - Configure parameters, rollback behavior, and monitoring
- Handle errors precisely - Structured error handling with detailed diagnostic information
- Connect with AWS - Configure profiles, Regions, and authentication programmatically

## Installation

```bash
npm install --save @aws-cdk/toolkit-lib
```

## Getting Started

### Basic Setup

```typescript
import { Toolkit } from '@aws-cdk/toolkit-lib';
import { App, Stack } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

// Create and configure the CDK Toolkit
const toolkit = new Toolkit();

// Create a cloud assembly source with an inline app
const cloudAssemblySource = await toolkit.fromAssemblyBuilder(async () => {
   const app = new App();
   const stack = new Stack(app, 'SimpleStorageStack');
   
   // Create an S3 bucket in the stack
   new s3.Bucket(stack, 'MyFirstBucket', {
      versioned: true
   });
   
   return app.synth();
});

// Deploy the stack
await toolkit.deploy(cloudAssemblySource);
```

### Configuration Options

```typescript
// Create toolkit with AWS profile
const toolkit = new Toolkit({
  sdkConfig: { profile: "my-profile" },
});
```

## Cloud Assembly Sources

### From Existing CDK App

```typescript
// TypeScript app
const cloudAssemblySource = await toolkit.fromCdkApp("ts-node app.ts");

// JavaScript app
const cloudAssemblySource = await toolkit.fromCdkApp("node app.js");

// Python app
const cloudAssemblySource = await toolkit.fromCdkApp("python app.py");
```

### From Inline Assembly Builder

```typescript
const cloudAssemblySource = await toolkit.fromAssemblyBuilder(async () => {
  const app = new App();
  new MyStack(app, 'MyStack');
  return app.synth();
});
```

### From Assembly Directory

```typescript
const cloudAssemblySource = await toolkit.fromAssemblyDirectory("cdk.out");
```

## Stack Selection Strategies

```typescript
import { StackSelectionStrategy } from '@aws-cdk/toolkit-lib';

// Select all stacks
await toolkit.deploy(cloudAssemblySource, {
  stacks: { strategy: StackSelectionStrategy.ALL_STACKS }
});

// Select by pattern
await toolkit.deploy(cloudAssemblySource, {
  stacks: {
    strategy: StackSelectionStrategy.PATTERN_MUST_MATCH,
    patterns: ["Dev-*", "Test-Backend"]
  }
});

// Select single stack (must be exactly one)
await toolkit.deploy(cloudAssemblySource, {
  stacks: { strategy: StackSelectionStrategy.ONLY_SINGLE }
});
```

## Core Actions

### Synthesis

```typescript
// Generate a cloud assembly
const cloudAssembly = await toolkit.synth(cloudAssemblySource);

// Use for multiple operations
await toolkit.list(cloudAssembly);
await toolkit.deploy(cloudAssembly);
await toolkit.diff(cloudAssembly);
```

### Deploy

```typescript
// Deploy with parameters
await toolkit.deploy(cloudAssemblySource, {
  parameters: StackParameters.exactly({
    "MyStack": {
      "BucketName": "amzn-s3-demo-bucket"
    }
  })
});

// Deploy with hotswap (development only)
await toolkit.deploy(cloudAssemblySource, {
  deploymentMethod: { method: "hotswap", fallback: true },
  stacks: {
    strategy: StackSelectionStrategy.PATTERN_MUST_MATCH,
    patterns: ["dev-stack"]
  }
});
```

### List Stacks

```typescript
const stackDetails = await toolkit.list(cloudAssemblySource, {
  stacks: {
    strategy: StackSelectionStrategy.PATTERN_MUST_MATCH,
    patterns: ["my-stack"]
  }
});

for (const stack of stackDetails) {
  console.log(`Stack: ${stack.id}, Dependencies: ${stack.dependencies}`);
}
```

### Watch for Changes

```typescript
const watcher = await toolkit.watch(cloudAssemblySource, {
  include: ["lib/**/*.ts"],
  exclude: ["**/*.test.ts"],
  deploymentMethod: { method: "hotswap" },
  stacks: { strategy: StackSelectionStrategy.ALL }
});

// Stop watching later
// await watcher.dispose();
```

### Rollback

```typescript
await toolkit.rollback(cloudAssemblySource, {
  orphanFailedResources: false,
  stacks: {
    strategy: StackSelectionStrategy.PATTERN_MUST_MATCH,
    patterns: ["failed-stack"]
  }
});
```

### Destroy

```typescript
await toolkit.destroy(cloudAssemblySource, {
  stacks: {
    strategy: StackSelectionStrategy.PATTERN_MUST_MATCH,
    patterns: ["dev-stack"]
  }
});
```

## Performance Optimization

### Caching Cloud Assemblies

```typescript
// Synthesize once and reuse
const cloudAssembly = await toolkit.synth(cloudAssemblySource);
try {
  // Multiple operations use the same assembly
  await toolkit.deploy(cloudAssembly);
  await toolkit.list(cloudAssembly);
} finally {
  // Clean up when done
  await cloudAssembly.dispose();
}
```

**Use cached assemblies when:**
- Performing multiple operations
- CDK app doesn't change frequently
- Performance is important

**Use sources directly when:**
- Performing single operations
- CDK app changes frequently
- Simplicity is preferred

## Error Handling

```typescript
import { ToolkitError } from '@aws-cdk/toolkit-lib';

try {
  await toolkit.deploy(cloudAssemblySource);
} catch (error) {
  if (ToolkitError.isAuthenticationError(error)) {
    console.error('Authentication failed. Check AWS credentials.');
  } else if (ToolkitError.isAssemblyError(error)) {
    console.error('CDK app error:', error.message);
  } else if (ToolkitError.isDeploymentError(error)) {
    console.error('Deployment failed:', error.message);
  } else if (ToolkitError.isToolkitError(error)) {
    console.error('CDK Toolkit error:', error.message);
  } else {
    console.error('Unexpected error:', error);
  }
}
```

## Complete Example

```typescript
import { Toolkit, StackSelectionStrategy } from '@aws-cdk/toolkit-lib';
import { App, Stack, RemovalPolicy } from 'aws-cdk-lib';
import { Bucket } from 'aws-cdk-lib/aws-s3';

async function deployInfrastructure(): Promise<void> {
    const toolkit = new Toolkit();
    let cloudAssembly;
    
    try {
        const cloudAssemblySource = await toolkit.fromAssemblyBuilder(async () => {
            const app = new App();
            const stack = new Stack(app, 'MyStack');
            
            new Bucket(stack, 'MyBucket', {
                versioned: true,
                removalPolicy: RemovalPolicy.DESTROY,
                autoDeleteObjects: true
            });
            
            return app.synth();
        });

        cloudAssembly = await toolkit.synth(cloudAssemblySource);
        
        await toolkit.list(cloudAssembly);
        await toolkit.deploy(cloudAssembly);
        
    } catch (error) {
        console.error("Failed to deploy:", error);
    } finally {
        if (cloudAssembly) {
            await cloudAssembly.dispose();
        }
    }
}

deployInfrastructure().catch(console.error);
```

## Best Practices

1. **Choose appropriate source method** - Select based on workflow requirements
2. **Cache cloud assemblies** - Generate once, reuse for multiple operations  
3. **Implement error handling** - Use structured error handling with clear messages
4. **Ensure version compatibility** - Match Toolkit Library version with CDK app version
5. **Manage resources properly** - Always dispose of cloud assemblies when finished
6. **Consider environment variables** - Be aware of CDK environment variable effects

## Use Cases

- **CI/CD Pipelines** - Automate infrastructure deployments
- **Custom Tools** - Build organization-specific deployment tools
- **Application Integration** - Integrate CDK actions into existing platforms
- **Custom Workflows** - Create specialized deployment workflows with validation
- **Multi-environment Management** - Implement advanced infrastructure management patterns

## Resources

- [CDK Toolkit Library npm package](https://www.npmjs.com/package/@aws-cdk/toolkit-lib)
- [AWS CDK Toolkit Library API Reference](https://docs.aws.amazon.com/cdk/api/toolkit-lib/)
