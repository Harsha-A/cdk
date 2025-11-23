Nice — I’ll give you a complete, **step-by-step, production-minded** AWS CDK (TypeScript) implementation that creates a Materials Management System (MMS) with:

* DynamoDB (materials table)
* REST API (API Gateway → Lambda) for CRUD of materials (material types, color variants, factories)
* SNS → SQS → Lambda asynchronous pipeline to ingest/transform events
* OpenSearch domain and a Lambda that indexes events into OpenSearch
* VPC, subnets, security groups, IAM permissions and least-privilege grants
* Node.js Lambdas (handler code included)
* Operational bits: long polling, DLQ, CloudWatch logging

I’ll include all the CDK code you need in one page: `package.json`, `bin/app.ts`, `lib/stack.ts` (the big one), and the Lambda handlers (Node.js). After the code, I’ll give deployment & testing steps and notes on improvements and security considerations.

> Assumptions / tradeoffs (keeps example focused and interview-friendly)
>
> * CDK v2 (aws-cdk-lib) in TypeScript
> * All infra in one stack for simplicity (you can split stacks by environment later)
> * OpenSearch domain is created inside the VPC to allow Lambda to reach it privately
> * Fine-grained OpenSearch auth is *not* enabled in this sample; the indexer Lambda calls OpenSearch using IAM SigV4 (recommended extension: enable fine-grained security and use signed requests / Cognito)
> * Use KMS-managed encryption for SQS/DynamoDB (defaults) and managed encryption for OpenSearch
> * Use FIFO queue for ordered ingestion if you need ordering; sample uses standard queue (changeable)

---

# Project layout (what this response produces)

```
mms-cdk/
├── package.json
├── cdk.json
├── tsconfig.json
├── bin/
│   └── app.ts
├── lib/
│   └── mms-stack.ts
├── lambda/
│   ├── api-handler/
│   │   └── index.js
│   └── ingest-processor/
│       └── index.js
└── README (instructions below)
```

---

## package.json

```json
{
  "name": "mms-cdk",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "build": "tsc",
    "synth": "cdk synth",
    "deploy": "cdk deploy --require-approval never",
    "destroy": "cdk destroy --force",
    "watch": "tsc -w",
    "test": "echo \"no tests\""
  },
  "devDependencies": {
    "aws-cdk-lib": "^2.80.0",
    "constructs": "^10.1.0",
    "typescript": "^5.0.0",
    "@types/node": "^18.0.0",
    "ts-node": "^10.0.0"
  },
  "dependencies": {
    "source-map-support": "^0.5.21"
  }
}
```

---

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["es2020"],
    "outDir": "dist",
    "rootDir": ".",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["**/*.ts"]
}
```

---

## cdk.json

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/app.ts"
}
```

---

## bin/app.ts

```ts
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { MmsStack } from '../lib/mms-stack';

const app = new cdk.App();

new MmsStack(app, 'MmsStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

---

## lib/mms-stack.ts  (BIG — main CDK stack)

> This file creates: VPC, SecurityGroups, DynamoDB table, SNS topic, SQS queue + DLQ, API Gateway + Lambda, ingest Lambda (SQS consumer), OpenSearch domain, roles/permissions, and outputs.

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw from 'aws-cdk-lib/aws-apigateway';
import * as ddb from 'aws-cdk-lib/aws-dynamodb';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as subs from 'aws-cdk-lib/aws-sns-subscriptions';
import * as events from 'aws-cdk-lib/aws-events';
import * as es from 'aws-cdk-lib/aws-opensearchservice';
import * as esNode from 'aws-cdk-lib/aws-opensearchservice/lib/domain';
import * as logs from 'aws-cdk-lib/aws-logs';
import { Duration } from 'aws-cdk-lib';

export class MmsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // -------------------------
    // VPC & Security
    // -------------------------
    const vpc = new ec2.Vpc(this, 'MmsVpc', {
      maxAzs: 2,
      natGateways: 1,
      subnetConfiguration: [
        { name: 'public', subnetType: ec2.SubnetType.PUBLIC },
        { name: 'private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      ],
    });

    const sgLambdaToES = new ec2.SecurityGroup(this, 'LambdaToEsSG', {
      vpc,
      allowAllOutbound: true,
      description: 'Allow Lambdas to connect to OpenSearch',
    });

    // Allow ingress to ES from Lambda SG (we'll use this later)
    // No direct port openings now (OpenSearch depends on internal endpoint)

    // -------------------------
    // DynamoDB (Materials Table)
    // -------------------------
    // Table design:
    // PK: PK (e.g., MATERIAL#<id>)
    // SK: METADATA#<type> or VARIANT#<variantId> — flexible single-table design
    const materialsTable = new ddb.Table(this, 'MaterialsTable', {
      partitionKey: { name: 'PK', type: ddb.AttributeType.STRING },
      sortKey: { name: 'SK', type: ddb.AttributeType.STRING },
      billingMode: ddb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      pointInTimeRecovery: true,
    });

    // Global secondary index for queries by material type
    materialsTable.addGlobalSecondaryIndex({
      indexName: 'GSI1',
      partitionKey: { name: 'GSI1PK', type: ddb.AttributeType.STRING },
      sortKey: { name: 'GSI1SK', type: ddb.AttributeType.STRING },
      projectionType: ddb.ProjectionType.ALL,
    });

    // -------------------------
    // SNS Topic & SQS pipeline
    // -------------------------
    const ingestTopic = new sns.Topic(this, 'IngestTopic', {
      displayName: 'MMS Ingest Topic',
      topicName: 'mms-ingest-topic',
    });

    // DLQ
    const ingestDlq = new sqs.Queue(this, 'IngestDLQ', {
      queueName: 'mms-ingest-dlq',
      retentionPeriod: Duration.days(14),
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Main SQS queue subscribed to SNS
    const ingestQueue = new sqs.Queue(this, 'IngestQueue', {
      queueName: 'mms-ingest-queue',
      visibilityTimeout: Duration.seconds(60),
      retentionPeriod: Duration.days(4),
      deadLetterQueue: {
        maxReceiveCount: 5,
        queue: ingestDlq,
      },
      receiveMessageWaitTime: Duration.seconds(20),
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Subscribe SQS to SNS
    ingestTopic.addSubscription(new subs.SqsSubscription(ingestQueue));

    // -------------------------
    // OpenSearch Domain
    // -------------------------
    // Create a domain inside the VPC so Lambdas in same VPC can reach it privately
    const domain = new es.Domain(this, 'MmsOpenSearchDomain', {
      version: es.EngineVersion.OPENSEARCH_1_0,
      capacity: {
        dataNodeInstanceType: 't3.small.search',
        dataNodes: 1,
      },
      ebs: {
        volumeSize: 10,
      },
      nodeToNodeEncryption: true,
      encryptionAtRest: {
        enabled: true,
      },
      enforceHttps: true,
      domainName: 'mms-materials-domain',
      vpc,
      vpcSubnets: [{ subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS }],
      securityGroups: [sgLambdaToES], // allow access using SG
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Allow connections from the Lambda security group to the OpenSearch domain
    // Note: Domain construct already creates a security group; we add rule on that group
    const esSecurityGroup = ec2.SecurityGroup.fromSecurityGroupId(
      this,
      'ImportedEsSG',
      (domain.node.findChild('DomainEndpoint') as any)?.securityGroupId || domain.vpc?.vpcCidrBlock || ''
    );
    // The above attempt to import SG id may not work across CDK versions.
    // Instead, allow es domain inbound from the Lambda SG by adding ingress on the lambda SG to ES domain's endpoint.
    // For simplicity: add permission for Lambdas' security group to connect via port 443 (HTTPS) outbound (default), and trust domain.

    // -------------------------
    // Lambda: API Handler (CRUD)
    // -------------------------
    const apiLambdaRole = new iam.Role(this, 'ApiLambdaRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      description: 'Role for API Lambdas to access DynamoDB and publish to SNS',
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
      ],
    });

    // Allow access to DynamoDB (narrow to table)
    materialsTable.grantReadWriteData(apiLambdaRole);

    // Allow Publish to SNS
    ingestTopic.grantPublish(apiLambdaRole);

    // Lambda security group inside VPC for accessing OpenSearch if needed
    const apiLambdaSG = new ec2.SecurityGroup(this, 'ApiLambdaSG', {
      vpc,
      allowAllOutbound: true,
      description: 'Security group for API Lambdas',
    });

    const apiLambda = new lambda.Function(this, 'ApiLambda', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/api-handler'),
      memorySize: 512,
      timeout: Duration.seconds(10),
      environment: {
        TABLE_NAME: materialsTable.tableName,
        TOPIC_ARN: ingestTopic.topicArn,
      },
      role: apiLambdaRole,
      vpc,
      securityGroups: [apiLambdaSG],
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
    });

    // -------------------------
    // API Gateway REST API -> Lambda (proxy)
    // -------------------------
    const api = new apigw.LambdaRestApi(this, 'MmsApi', {
      handler: apiLambda,
      proxy: false,
      restApiName: 'MMS API',
      defaultCorsPreflightOptions: {
        allowOrigins: apigw.Cors.ALL_ORIGINS,
        allowMethods: apigw.Cors.ALL_METHODS,
      },
    });

    // /materials resource
    const materials = api.root.addResource('materials');

    // POST /materials -> create a material (id, type, colors[], factories[])
    materials.addMethod('POST', new apigw.LambdaIntegration(apiLambda));

    // GET /materials -> list (simple scan or query)
    materials.addMethod('GET', new apigw.LambdaIntegration(apiLambda));

    // /materials/{id}
    const single = materials.addResource('{id}');
    single.addMethod('GET', new apigw.LambdaIntegration(apiLambda));
    single.addMethod('PUT', new apigw.LambdaIntegration(apiLambda));
    single.addMethod('DELETE', new apigw.LambdaIntegration(apiLambda));

    // -------------------------
    // Lambda: Ingest processor - consumes SQS and indexes to OpenSearch
    // -------------------------
    // IAM role for ingest processor
    const ingestRole = new iam.Role(this, 'IngestProcessorRole', {
      assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        iam.ManagedPolicy.fromAwsManagedPolicyName('service-role/AWSLambdaBasicExecutionRole'),
      ],
    });

    // SQS permissions
    ingestQueue.grantConsumeMessages(ingestRole);

    // Allow reading from DynamoDB if needed
    materialsTable.grantReadData(ingestRole);

    // Allow the role to call OpenSearch using SigV4 (es:ESHttp* permissions)
    ingestRole.addToPolicy(
      new iam.PolicyStatement({
        actions: ['es:ESHttpPost', 'es:ESHttpPut', 'es:ESHttpGet'],
        resources: [domain.domainArn + '/*'],
      })
    );

    // Create security group for ingest lambda
    const ingestLambdaSG = new ec2.SecurityGroup(this, 'IngestLambdaSG', {
      vpc,
      allowAllOutbound: true,
      description: 'SG for ingest Lambda to access OpenSearch',
    });

    // Make sure the lambda SG can reach OpenSearch endpoint over HTTPS (port 443)
    // OpenSearch domain already has SGs; to simplify, allow all traffic (within VPC) - in prod tighten this.
    sgLambdaToES.addIngressRule(ingestLambdaSG, ec2.Port.tcp(443), 'allow ingest lambdas to connect to OpenSearch over TLS');

    const ingestLambda = new lambda.Function(this, 'IngestLambda', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/ingest-processor'),
      memorySize: 512,
      timeout: Duration.seconds(30),
      vpc,
      securityGroups: [ingestLambdaSG],
      vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      environment: {
        OPENSEARCH_ENDPOINT: domain.domainEndpoint, // domain endpoint
        INDEX_NAME: 'materials',
        TABLE_NAME: materialsTable.tableName,
      },
      role: ingestRole,
    });

    // Add SQS event source mapping via CDK high-level construct
    // Using EventSourceMapping requires aws-lambda-event-sources package; we can create mapping programmatically
    const eventSource = new (require('aws-cdk-lib/aws-lambda-event-sources').SqsEventSource)(ingestQueue, {
      batchSize: 10,
      maxBatchingWindow: Duration.seconds(30),
      reportBatchItemFailures: true,
    });

    ingestLambda.addEventSource(eventSource);

    // -------------------------
    // Outputs
    // -------------------------
    new cdk.CfnOutput(this, 'ApiUrl', { value: api.url, description: 'API Gateway URL' });
    new cdk.CfnOutput(this, 'MaterialsTableName', { value: materialsTable.tableName });
    new cdk.CfnOutput(this, 'IngestTopicArn', { value: ingestTopic.topicArn });
    new cdk.CfnOutput(this, 'IngestQueueUrl', { value: ingestQueue.queueUrl });
    new cdk.CfnOutput(this, 'OpenSearchEndpoint', { value: domain.domainEndpoint });

    // -------------------------
    // Helpful tags
    // -------------------------
    cdk.Tags.of(this).add('project', 'mms');
  }
}
```

> NOTE: some small differences in CDK versions can change minor API surface (import paths for event sources or security group id extraction). The code above targets CDK v2. If a function like `domain.domainEndpoint` or security group import requires small edits in your CDK version, adjust accordingly.

---

## lambda/api-handler/index.js (Node.js Lambda for API)

This single Lambda handles multiple HTTP methods and routes (for interview / sample simplicity). In production you might split handlers.

```js
const AWS = require('aws-sdk');
const { v4: uuidv4 } = require('uuid');

const ddb = new AWS.DynamoDB.DocumentClient();
const sns = new AWS.SNS();

const TABLE_NAME = process.env.TABLE_NAME;
const TOPIC_ARN = process.env.TOPIC_ARN;

exports.handler = async (event) => {
  console.log('Received event', JSON.stringify(event, null, 2));

  try {
    const method = event.httpMethod;
    const path = event.resource; // e.g., /materials or /materials/{id}
    const body = event.body ? JSON.parse(event.body) : null;
    const id = event.pathParameters ? event.pathParameters.id : null;

    // Basic router
    if (method === 'POST' && path === '/materials') {
      // Create material
      const materialId = uuidv4();
      const { name, type, colors, factories, metadata } = body;

      const item = {
        PK: `MATERIAL#${materialId}`,
        SK: 'METADATA',
        materialId,
        name,
        type,
        colors: colors || [],
        factories: factories || [],
        metadata: metadata || {},
        createdAt: new Date().toISOString(),
      };

      await ddb.put({ TableName: TABLE_NAME, Item: item }).promise();

      // Publish an ingest event to SNS for async indexing
      const eventMsg = {
        eventType: 'MATERIAL_CREATED',
        materialId,
        payload: item,
      };

      await sns.publish({ TopicArn: TOPIC_ARN, Message: JSON.stringify(eventMsg) }).promise();

      return {
        statusCode: 201,
        body: JSON.stringify({ materialId }),
      };
    }

    if (method === 'GET' && path === '/materials') {
      // Simple scan — for interview only; prefer queries with GSI for production
      const res = await ddb.scan({ TableName: TABLE_NAME, Limit: 100 }).promise();
      return {
        statusCode: 200,
        body: JSON.stringify(res.Items),
      };
    }

    if (method === 'GET' && path === '/materials/{id}') {
      const pk = `MATERIAL#${id}`;
      const res = await ddb.get({ TableName: TABLE_NAME, Key: { PK: pk, SK: 'METADATA' } }).promise();
      if (!res.Item) {
        return { statusCode: 404, body: JSON.stringify({ error: 'Not found' }) };
      }
      return { statusCode: 200, body: JSON.stringify(res.Item) };
    }

    if (method === 'PUT' && path === '/materials/{id}') {
      const pk = `MATERIAL#${id}`;
      const { name, type, colors, factories, metadata } = body;

      // simple update
      const updateParams = {
        TableName: TABLE_NAME,
        Key: { PK: pk, SK: 'METADATA' },
        UpdateExpression: 'SET #n = :n, #t = :t, colors = :colors, factories = :factories, metadata = :metadata, updatedAt = :u',
        ExpressionAttributeNames: { '#n': 'name', '#t': 'type' },
        ExpressionAttributeValues: {
          ':n': name,
          ':t': type,
          ':colors': colors || [],
          ':factories': factories || [],
          ':metadata': metadata || {},
          ':u': new Date().toISOString(),
        },
        ReturnValues: 'ALL_NEW',
      };

      const updated = await ddb.update(updateParams).promise();

      // publish event
      const eventMsg = {
        eventType: 'MATERIAL_UPDATED',
        materialId: id,
        payload: updated.Attributes,
      };
      await sns.publish({ TopicArn: TOPIC_ARN, Message: JSON.stringify(eventMsg) }).promise();

      return { statusCode: 200, body: JSON.stringify(updated.Attributes) };
    }

    if (method === 'DELETE' && path === '/materials/{id}') {
      const pk = `MATERIAL#${id}`;
      await ddb.delete({ TableName: TABLE_NAME, Key: { PK: pk, SK: 'METADATA' } }).promise();

      const eventMsg = { eventType: 'MATERIAL_DELETED', materialId: id };
      await sns.publish({ TopicArn: TOPIC_ARN, Message: JSON.stringify(eventMsg) }).promise();

      return { statusCode: 204, body: '' };
    }

    return { statusCode: 400, body: JSON.stringify({ error: 'Unsupported route' }) };
  } catch (err) {
    console.error('Error', err);
    return { statusCode: 500, body: JSON.stringify({ error: 'Internal error', detail: err.message }) };
  }
};
```

**Notes about the API lambda:**

* Uses DynamoDB DocumentClient.
* Creates/updates items and publishes events to SNS for async ingestion into OpenSearch.
* In a production system you'd split responsibilities, implement validation, use structured logging and tracing, and more granular IAM.

---

## lambda/ingest-processor/index.js (SQS consumer → OpenSearch indexer)

This Lambda receives SQS messages (SNS delivery to SQS), optionally enriches or fetches DynamoDB data, and indexes into OpenSearch using SigV4 signed HTTP requests.

> We'll implement a SigV4 signed request using the `aws-sdk` v2 `HttpRequest` / `Signers` approach. In Lambda Node 18+ you can use `@aws-sdk/signature-v4` (v3) — but to keep the code compact and widely compatible, I'll use `aws-sdk` built-in signer.

```js
const AWS = require('aws-sdk');
const https = require('https');
const url = require('url');

const ddb = new AWS.DynamoDB.DocumentClient();
const region = process.env.AWS_REGION || 'us-east-1';
const endpoint = process.env.OPENSEARCH_ENDPOINT; // e.g. search-mms-...amazonaws.com
const indexName = process.env.INDEX_NAME || 'materials';
const tableName = process.env.TABLE_NAME;

const endpointUrl = `https://${endpoint}`;

function buildRequest(path, method, body) {
  const parsed = new url.URL(endpointUrl + path);
  return {
    method,
    hostname: parsed.hostname,
    path: parsed.pathname + parsed.search,
    body,
    headers: {
      'Content-Type': 'application/json',
      'Host': parsed.hostname,
    },
  };
}

function sendSignedRequest(opts) {
  return new Promise((resolve, reject) => {
    const credentials = new AWS.EnvironmentCredentials('AWS');
    credentials.get((err) => {
      if (err) return reject(err);

      const signer = new AWS.Signers.V4(new AWS.HttpRequest(new AWS.Endpoint(endpointUrl), region), 'es');
      signer.request = new AWS.HttpRequest(new AWS.Endpoint(endpointUrl), region);
      signer.request.method = opts.method;
      signer.request.path = opts.path;
      signer.request.body = opts.body;
      signer.request.headers = opts.headers;

      signer.addAuthorization(credentials, new Date());

      const requestOptions = {
        hostname: opts.hostname,
        path: opts.path,
        method: opts.method,
        headers: signer.request.headers,
      };

      const req = https.request(requestOptions, (res) => {
        let body = '';
        res.on('data', (chunk) => (body += chunk));
        res.on('end', () => {
          resolve({ statusCode: res.statusCode, body });
        });
      });
      req.on('error', reject);
      if (opts.body) req.write(opts.body);
      req.end();
    });
  });
}

exports.handler = async (event) => {
  console.log('Ingest event', JSON.stringify(event, null, 2));
  // event.Records for SQS messages
  for (const record of event.Records) {
    try {
      // SNS -> SQS message format wraps SNS Message as string in record.body
      const payload = JSON.parse(record.body);
      const snsMessage = payload; // if subscribed raw; if SNS wraps, adjust accordingly
      let messageObj;
      // Handle SNS envelope
      if (snsMessage.Message) {
        messageObj = JSON.parse(snsMessage.Message);
      } else {
        messageObj = snsMessage;
      }

      const { eventType, materialId, payload: materialPayload } = messageObj;

      // If you need more data, fetch from DynamoDB
      let doc = materialPayload;
      if (!doc || Object.keys(doc).length === 0) {
        // fetch TTL: METADATA
        const pk = `MATERIAL#${materialId}`;
        const res = await ddb.get({ TableName: tableName, Key: { PK: pk, SK: 'METADATA' } }).promise();
        doc = res.Item || {};
      }

      if (eventType === 'MATERIAL_DELETED') {
        // delete document from OpenSearch
        const path = `/${indexName}/_doc/${materialId}`;
        const opts = buildRequest(path, 'DELETE', null);
        opts.host = opts.hostname;
        opts.hostname = opts.hostname;
        opts.path = `${path}`;
        opts.headers = { Host: new URL(endpointUrl).host };
        await sendSignedRequest(opts);
        console.log('Deleted doc', materialId);
      } else {
        // index or update document
        const path = `/${indexName}/_doc/${materialId}`;
        const body = JSON.stringify(doc);
        const opts = buildRequest(path, 'PUT', body);
        opts.path = path;
        opts.hostname = opts.hostname;
        opts.headers = { 'Content-Type': 'application/json', Host: new URL(endpointUrl).host };
        opts.body = body;
        const resp = await sendSignedRequest(opts);

        if (resp.statusCode >= 200 && resp.statusCode < 300) {
          console.log('Indexed doc', materialId);
        } else {
          console.error('OpenSearch indexing error', resp.statusCode, resp.body);
          throw new Error('Indexing failed');
        }
      }
    } catch (err) {
      console.error('Failed to process record', err);
      // Let Lambda fail this record — with reportBatchItemFailures true SQS & Lambda will handle it
      throw err;
    }
  }
};
```

**Notes on signer code:**

* Signing OpenSearch requests requires SigV4. The simplified approach above uses `aws-sdk` v2 signer (`AWS.Signers.V4`) and `AWS.HttpRequest` — it's compact for demo purposes. For production prefer `@aws-sdk/signature-v4` and `@opensearch-project/opensearch` with AWS SigV4 plugin.
* If the OpenSearch domain requires Cognito or basic auth, adapt accordingly.

---

## Deploy & run (step-by-step)

1. Install dependencies

   ```bash
   npm install
   npm run build
   ```

2. Bootstrap your AWS environment (one-time)

   ```bash
   npx cdk bootstrap aws://ACCOUNT_ID/REGION
   ```

   Replace ACCOUNT_ID and REGION or rely on your environment variables.

3. Synthesize & deploy

   ```bash
   npx cdk synth
   npx cdk deploy --require-approval never
   ```

   The deployment outputs will include `ApiUrl` (API Gateway base URL), `OpenSearchEndpoint`, `MaterialsTableName`, etc.

4. Test the API (create a material)

   ```bash
   API_URL=$(npx cdk output --stack MmsStack ApiUrl) # or copy from cdk deploy output
   curl -X POST "$API_URL/materials" -H "Content-Type: application/json" -d '{
     "name":"Premium Leather",
     "type":"leather",
     "colors":["brown","black"],
     "factories":[{"id":"F1","name":"Factory A"}]
   }'
   ```

5. Confirm the pipeline:

   * The API Lambda writes to DynamoDB and publishes to SNS.
   * SNS delivers to SQS; the ingest Lambda consumes messages and indexes into OpenSearch.
   * Check OpenSearch endpoint using `curl` (beware private VPC endpoint — likely only accessible from inside VPC).
   * Inspect CloudWatch logs for `IngestLambda` and `ApiLambda`.

6. Redrive / DLQ:

   * If ingest fails, messages will go to the DLQ; use the console or CLI to inspect and redrive.

---

## Important production improvements & considerations

* **Auth & API Security**

  * Protect API with Cognito Authorizer or IAM authorizer, not open.
  * Add request validation and schema checks before writing to DynamoDB.

* **OpenSearch security**

  * Enable fine-grained access control or use Cognito with role mapping.
  * Restrict access using security groups and private subnets only.
  * Consider using OpenSearch Service fine-grained RBAC and an API Gateway + Lambda that signs requests.

* **Indexing performance**

  * Bulk index multiple documents in one request for throughput.
  * Add retry/backoff logic for failed indexing to avoid data loss.

* **Observability**

  * Add X-Ray tracing in Lambdas and propagate trace IDs.
  * Add CloudWatch metrics and alarms for failed messages, queue depth, and OpenSearch indexing errors.

* **IAM least privilege**

  * Narrow all IAM policies to the minimum required resources (table ARNs, topic ARNs, queue ARNs, domain ARN).
  * Avoid wildcard resources where possible.

* **Error handling**

  * For partial failures in batch processing, use `reportBatchItemFailures` (we configured it) and return appropriate result structure.

* **Schema & Single Table Design**

  * I used a simple single-table approach for materials. In interviews be prepared to discuss alternatives (separate tables, GSI designs, access patterns).

* **Cost**

  * OpenSearch costs can be significant — for interviews choose the smallest instance types or use Amazon OpenSearch Serverless if available/appropriate.

---

# Detailed Explaination of above code: 
---


Nice — below is a detailed, line-by-line explanation of that `bin/app.ts` entrypoint for an AWS CDK TypeScript app, plus context about how CDK uses it, best practices, runtime behavior, and common pitfalls.

# Explanation — `bin/app.ts` (full breakdown)

```ts
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { MmsStack } from '../lib/mms-stack';

const app = new cdk.App();

new MmsStack(app, 'MmsStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

## `import 'source-map-support/register';`

* This imports and runs the `source-map-support` module at startup.
* Purpose: when TypeScript code throws an error at runtime (after `ts-node` or compiled), `source-map-support` maps stack traces back to original `.ts` source lines instead of compiled `.js` lines. That makes debugging much easier.
* Behavior: no symbol is bound to a variable; importing the module for its side-effects is common pattern.

## `import * as cdk from 'aws-cdk-lib';`

* Imports the CDK v2 library into the `cdk` namespace.
* `aws-cdk-lib` contains constructs, `App`, `Stack`, `Tags`, helper functions, and all L2/L1 construct classes.
* You use this object to create the CDK application (`cdk.App()`) and to reference constants like `cdk.Environment`, `cdk.Duration`, `cdk.Tags`, etc.

## `import { MmsStack } from '../lib/mms-stack';`

* Imports your custom stack class `MmsStack` from your project code (the stack that defines all resources).
* `MmsStack` should extend `cdk.Stack` and contain the infrastructure definitions (DynamoDB, Lambdas, API Gateway, VPC, OpenSearch, etc.).
* Keeping your stack implementation in `lib/` and the entrypoint in `bin/` is a common CDK project layout.

## `const app = new cdk.App();`

* Creates a CDK **application** object representing the root of the construct tree.
* The `App` acts as the container for one or more `Stack` instances.
* When you run `cdk synth` or `cdk deploy`, CDK executes this file, builds the construct tree, and synthesizes CloudFormation templates for each stack attached to the `App`.
* Important behaviors:

  * **Synthesis time** vs **deploy time**: any code that runs here is executed at *synth-time*; avoid runtime calls that depend on unavailable resources unless intentional (use context or lookups).
  * You can create multiple stacks (e.g., `new MmsStack(app, 'MmsStack'); new OtherStack(app, 'OtherStack')`) and control their `env` per stack.

## `new MmsStack(app, 'MmsStack', { env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION, }, });`

* This line *instantiates* your `MmsStack` and attaches it to the `app` construct tree.
* Parameters explained:

  1. `app` — the parent scope; puts the stack under the application.
  2. `'MmsStack'` — the logical (construct) id for the Stack. CDK uses this id as part of resource logical IDs and the synthesized CloudFormation stack name (unless you override `stackName` in props).
  3. The third argument is `StackProps` — here you pass an `env` property that pins the stack to a specific AWS account and region.

     * `env.account` and `env.region` are being read from environment variables `CDK_DEFAULT_ACCOUNT` and `CDK_DEFAULT_REGION`.
     * CDK sets those environment variables automatically if you run `cdk deploy` with an AWS CLI profile that resolves an account and region, or when `cdk bootstrap` has been run and your CLI is configured. They’re also populated by `aws-cdk`/`cdk` when executing in CI with environment variables.
     * Example values: `account: '123456789012'`, `region: 'us-east-1'`.

### Why specifying `env` matters

* **Environment-specific stacks** (with `env` set) allow CDK to do account/region-specific lookups (like `Vpc.fromLookup`) and to create stable ARNs/tokens. They produce a CloudFormation template that’s tied to that account/region.
* If you omit `env`, your stack is **environment-agnostic** (a "cloud-agnostic" synthesized template). That limits some features: CDK cannot perform certain context lookups (like imported VPCs by attributes) during synthesis.
* Best practice: for simple apps use environment-agnostic stacks during development; for real deployments set `env` explicitly (or rely on `CDK_DEFAULT_ACCOUNT`/`CDK_DEFAULT_REGION` provided by your execution context).

## How CDK executes this file

* When you run `npx cdk synth` or `npx cdk deploy`, the CDK CLI runs the `app` entrypoint (per `cdk.json`), which executes this TS file.
* The construct tree is built, then CDK synthesizes one CloudFormation template per stack.
* `cdk deploy` then uses CloudFormation to apply the generated templates.

# Practical notes, tips & gotchas

### 1. How `CDK_DEFAULT_ACCOUNT` and `CDK_DEFAULT_REGION` are set

* If you run `cdk deploy` with AWS credentials configured (from `aws configure` or environment vars like `AWS_PROFILE`), the CDK CLI resolves account & region and sets those environment variables for the synth process.
* In CI, set `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` and `AWS_REGION` or use IAM role on the runner.

### 2. When to omit `env` vs pin it

* **Omit `env`**: good for libraries or when you want the synthesized template to be deployable to any account/region. But you lose lookup abilities (VPC lookup, availability zone discovery).
* **Pin `env`**: good for app stacks intended for a specific account/region or when you perform lookups at synth time.

### 3. Multiple stacks & stages

* You can instantiate multiple stacks in `bin/app.ts`. For example, separate `NetworkStack`, `DataStack`, `AppStack`. This helps boundaries, reusability, and different lifecycle management.
* For multi-stage deployment flows (dev/stage/prod), create a small script that instantiates stacks with different props or use CDK Pipelines.

### 4. Synth-time vs deploy-time side effects

* Code in `bin/app.ts` runs during synthesis. Avoid calling external APIs that you expect to change frequently unless using `cdk.context` or explicit context values — these calls will affect determinism.
* If you need dynamic values from AWS at synth time (like VPC lookups), CDK performs those lookups and stores them in `cdk.context.json`. Be aware this is cached.

### 5. Stack names and logical IDs

* The second argument `'MmsStack'` becomes part of CloudFormation stack name and resource logical IDs. Changing this id later causes replacement of many resources (logical ID changes) — treat the id as stable.

### 6. Debugging & source maps

* `source-map-support` helps make error stack traces readable (maps compiled JS back to original TS). Useful when developing with `ts-node` or after `tsc`.

### 7. Bootstrapping requirement

* If your stack uses assets (Lambda code, Docker images), you must run `cdk bootstrap` in the target environment before `cdk deploy`. Bootstrapping creates the S3 bucket, ECR repo, and roles CDK needs to publish assets.

### 8. Permissions for deployment

* `cdk deploy` requires your deploy principal (CLI user/role) to have permissions to create/update CloudFormation stacks and to create the resources defined in the stack. Use least privilege in production; for local dev you may use broader deployer permissions.

# Examples & common variations

### Use a specific account/region inline

```ts
new MmsStack(app, 'MmsStack', {
  env: { account: '123456789012', region: 'us-east-1' },
});
```

### Environment-agnostic (no env)

```ts
new MmsStack(app, 'MmsStack'); // more portable; good for libraries
```

### Multiple stacks

```ts
new NetworkStack(app, 'NetworkStack', { env: { account, region }});
new DataStack(app, 'DataStack', { env: { account, region }});
new AppStack(app, 'AppStack', { env: { account, region }});
```

# Quick checklist before you run this file

* `npm install` dependencies (`aws-cdk-lib`, `constructs`, `ts-node`, etc.)
* Run `npx cdk bootstrap` in your target account/region if your stack uses assets
* Ensure AWS credentials are configured (environment variables, AWS_PROFILE, or CI role)
* Run `npx cdk synth` to see the generated templates
* Run `npx cdk deploy` to deploy

---

