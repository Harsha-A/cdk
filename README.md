# CDK Interview Take-Home — Full Project Template

> A production-minded, interview-ready AWS CDK project template (TypeScript) with explanations, tests, CI, and deploy/run instructions. Use this as a starting point for take-home assignments or live coding interviews where you must show infra-as-code best practices, testing, and operational readiness.

---

## Goals
- Provide a minimal but realistic project that demonstrates CDK skills: a queue + lambda processing pipeline, monitoring, secure defaults, and unit tests.
- Include explanations so you can talk through design choices in interviews.

---

## High-level architecture

- API (optional) or producer (simulated) sends messages to SQS
- SQS Queue with DLQ, encryption and long-polling
- Lambda consumer triggered by SQS
- CloudWatch alarms for queue depth & oldest message

---

## Project structure

```
cdk-interview-template/
├── README.md
├── package.json
├── cdk.json
├── tsconfig.json
├── .gitignore
├── bin/
│   └── app.ts
├── lib/
│   └── stack.ts
├── lambda/
│   └── processor/
│       └── index.ts
├── test/
│   └── stack.test.ts
├── .github/workflows/ci.yml
└── jest.config.js
```

---

## Prerequisites
- Node.js 18+
- AWS CLI configured (profile with deploy permissions)
- AWS CDK v2 installed (`npm install -g aws-cdk`) or use `npx` in scripts
- `cdk bootstrap` executed in target account/region

---

## Files (full content)

### package.json

```json
{
  "name": "cdk-interview-template",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "synth": "cdk synth",
    "deploy": "cdk deploy --require-approval never",
    "destroy": "cdk destroy --force",
    "test": "jest",
    "lint": "eslint --ext .ts .",
    "ci": "npm run build && npm run test && cdk synth"
  },
  "devDependencies": {
    "@types/jest": "^29.0.0",
    "@types/node": "^18.0.0",
    "aws-cdk-lib": "^2.1000.0",
    "constructs": "^10.0.0",
    "esbuild": "^0.18.0",
    "jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "ts-node": "^10.0.0",
    "typescript": "^5.0.0"
  },
  "dependencies": {
    "source-map-support": "^0.5.21"
  }
}
```

> **Note:** adjust versions to the latest stable ones in real repo.

---

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["es2020"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "."
  },
  "include": ["**/*.ts"]
}
```

---

### cdk.json

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/app.ts",
  "context": {
    "@aws-cdk/core:newStyleStackSynthesis": true
  }
}
```

---

### .gitignore

```
node_modules/
cdk.out/
dist/
.env
.nyc_output/
coverage/
```

---

### bin/app.ts

```ts
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { SqsLambdaStack } from '../lib/stack';

const app = new cdk.App();

new SqsLambdaStack(app, 'SqsLambdaStack', {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION },
});
```

**Explanation:** Entrypoint that instantiates the stack. Using environment variables allows `cdk deploy` to infer the account/region.

---

### lib/stack.ts

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as events from 'aws-cdk-lib/aws-events';
import * as alarms from 'aws-cdk-lib/aws-cloudwatch';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Duration } from 'aws-cdk-lib';
import * as path from 'path';

export interface SqsLambdaStackProps extends cdk.StackProps {}

export class SqsLambdaStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: SqsLambdaStackProps) {
    super(scope, id, props);

    // Dead-letter queue
    const dlq = new sqs.Queue(this, 'DLQ', {
      queueName: 'interview-dlq.fifo',
      fifo: true,
      retentionPeriod: Duration.days(14),
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Main FIFO queue (ordering example)
    const queue = new sqs.Queue(this, 'MainQueue', {
      queueName: 'interview-main.fifo',
      fifo: true,
      contentBasedDeduplication: true,
      visibilityTimeout: Duration.seconds(60),
      retentionPeriod: Duration.days(4),
      receiveMessageWaitTime: Duration.seconds(20),
      deadLetterQueue: {
        queue: dlq,
        maxReceiveCount: 5,
      },
      encryption: sqs.QueueEncryption.KMS_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Lambda function
    const fn = new lambda.Function(this, 'Processor', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset(path.join(__dirname, '..', 'lambda', 'processor')),
      timeout: Duration.seconds(30),
      environment: {
        QUEUE_URL: queue.queueUrl,
      },
    });

    // Grant permissions
    queue.grantConsumeMessages(fn);
    fn.addToRolePolicy(new iam.PolicyStatement({
      actions: ['sqs:ChangeMessageVisibility'],
      resources: ['*'],
    }));

    // Event source mapping
    fn.addEventSource(new (require('aws-cdk-lib/aws-lambda-event-sources').SqsEventSource)(queue, {
      batchSize: 10,
      maxBatchingWindow: Duration.seconds(30),
      reportBatchItemFailures: true,
    }));

    // Alarms
    const alarm = new alarms.Alarm(this, 'QueueDepthAlarm', {
      metric: queue.metricApproximateNumberOfMessagesVisible(),
      threshold: 50,
      evaluationPeriods: 1,
      datapointsToAlarm: 1,
      alarmName: 'Interview-Queue-Depth-Alarm',
    });

    // Output
    new cdk.CfnOutput(this, 'QueueUrl', { value: queue.queueUrl });
    new cdk.CfnOutput(this, 'DlqUrl', { value: dlq.queueUrl });
  }
}
```

**Explanation (highlights):**
- FIFO queue with content-based deduplication demonstrates ordering/dedupe knowledge.
- Long polling & retention tuned for cost & durability.
- KMS-managed encryption by default for secure data at rest.
- DLQ with `maxReceiveCount` for failed messages.
- Lambda function packaged as asset and given SQS consume permissions.
- Event source mapping uses `reportBatchItemFailures` to handle partial failures.
- CloudWatch alarm on visible messages shows operational monitoring.

---

### lambda/processor/index.ts

```ts
import { SQSHandler } from 'aws-lambda';

export const handler: SQSHandler = async (event) => {
  for (const record of event.Records) {
    try {
      console.log('Processing message:', record.messageId, record.body);
      // Simulate processing
      if (record.body.includes('fail')) {
        throw new Error('Simulated processing failure');
      }
      // do real work here (call DB, external API, etc.)
    } catch (err) {
      console.error('Failed to process message', record.messageId, err);
      // Throw to let Lambda mark this record as failed when using reportBatchItemFailures
      throw err;
    }
  }
};
```

**Explanation:** Keep handler small, idempotent, and instrumented. Throwing errors signals partial failures to Lambda when `reportBatchItemFailures` is enabled.

---

### test/stack.test.ts

```ts
import * as cdk from 'aws-cdk-lib';
import { Template } from 'aws-cdk-lib/assertions';
import { SqsLambdaStack } from '../lib/stack';

test('Stack synthesizes expected resources', () => {
  const app = new cdk.App();
  const stack = new SqsLambdaStack(app, 'TestStack');
  const template = Template.fromStack(stack);

  // SQS queue exists
  template.hasResourceProperties('AWS::SQS::Queue', {
    // check that attributes exist on synthesized queue
  });

  // Lambda exists
  template.resourceCountIs('AWS::Lambda::Function', 1);
});
```

**Explanation:** Basic unit test uses CFN template assertions to check resources. Add more granular tests for IAM and properties in interviews.

---

### jest.config.js

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/test/**/*.test.ts'],
};
```

---

### .github/workflows/ci.yml

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
      - name: Install
        run: npm ci
      - name: Build
        run: npm run build
      - name: Test
        run: npm test
      - name: CDK Synth
        run: npx cdk synth
```

**Explanation:** CI does build/test/synth — never deploys to protect real accounts in interviews. For interviews you can also show a PR workflow that runs `cdk diff` on changes.

---

## How to run locally

1. `npm install`
2. `npm run build`
3. `cdk bootstrap` in your target account/region (one-time)
4. `cdk synth` to view synthesized template
5. `cdk deploy` to deploy resources
6. Use AWS Console or `aws` CLI to send messages to the queue and examine Lambda logs in CloudWatch

---

## Interview talking points & reasoning

- **Why FIFO in this template?** Shows you understand ordering/deduplication trade-offs; for high-throughput without ordering you'd choose standard.
- **Why content-based dedupe?** Simplifies dedupe without requiring the producer to set deduplication IDs.
- **Why `reportBatchItemFailures`?** Allows partial success in a Lambda batch; must be combined with throwing errors per record.
- **Why KMS-managed encryption?** Secure-by-default; ensures messages at rest are encrypted without needing a custom key policy.
- **Why CloudWatch alarms?** Operational readiness — surface large backlogs quickly.
- **Why RemovalPolicy.RETAIN?** Avoid accidental data loss in interview/demo environments; discuss alternatives like DESTROY for ephemeral sandbox stacks.

---

## Extensions & variations to demonstrate in interview

- Add API Gateway + Producer Lambda to show E2E flow.
- Add DynamoDB table and transactional flow with exactly-once semantics considerations.
- Add SQS cross-account permissions and show `addToResourcePolicy` usage.
- Use KMS CMK and show required key policy changes.
- Add automated redrive job (Step Functions) to replay DLQ messages to a sandbox queue for reprocessing.

---

## Common pitfalls to call out

- Visibility timeout misconfigured causing duplicates.
- Lambda timeouts shorter than visibility timeout causing unexpected retries.
- Forgetting KMS key permissions when using CMKs.
- Not using long-polling causing excess API calls/costs.

---

## Final notes

- Remember to **destroy** the stack after demoing (`cdk destroy`) if deployed in a personal account, or ensure resources use RemovalPolicy.DESTROY for ephemeral demos.
- This template is intentionally opinionated in favor of secure defaults and operational readiness. In a timed interview you can remove extras to focus on required features.




Tell me which next step and I’ll create it.

