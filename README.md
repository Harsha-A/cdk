# cdk
aws cdk prep notes


Here you go — a **focused SDE-3 level AWS CDK interview cheat-sheet with 25 deep, high-signal technical questions and concise, high-quality answers.**
This is specifically for **AWS CDK**, not general AWS, and is crafted for senior-level interviews.

---

# **AWS CDK — SDE-3 Deep Technical Interview Cheat-Sheet (25 Q&A)**

---

## **1. What is the difference between L1, L2, and L3 constructs in CDK?**

**Answer:**

* **L1 (CFN Resources):** Auto-generated from CloudFormation spec; 1:1 mapping; verbose; all properties exposed.
* **L2 (High-level abstractions):** Opinionated “default-safe” constructs with sensible defaults (e.g., `sqs.Queue`).
* **L3 (Patterns):** Complex multi-resource patterns implementing best practices (e.g., `aws-solutions-constructs` like API GW → Lambda → DynamoDB).
* **SDE-3 expectation:** choose L1 when you need full control or bleeding-edge features; L2 for 90% use cases; L3 for faster iteration.

---

## **2. How does CDK synthesize infrastructure? What exactly is generated?**

**Answer:**
CDK synthesizes your code into a **CloudFormation template (JSON)** using the **construct tree**. Every construct generates a CFN resource in the synthesized output.

* Uses **jsii** to support multiple languages.
* Output = `cdk.out/` containing templates and assets.
* No infrastructure is created until **CloudFormation deploys** it.

---

## **3. Explain the CDK construct tree and scope resolution.**

**Answer:**
Every construct has:

* **scope** (parent)
* **id** (unique within that parent)
* **children**

Construct tree forms a hierarchical structure. Logical IDs in CloudFormation derive from this tree.
Understanding this is critical for correct resource isolation, environment scoping, and cross-stack references.

---

## **4. What are CDK Aspects and when would you use them?**

**Answer:**
**Aspects** allow you to apply an operation across the entire construct tree (e.g., enforce tagging, security rules).
Example use cases:

* Enforce encryption on all S3 buckets.
* Add standard monitoring to all Lambda functions.
* Apply organization-wide governance.

---

## **5. How do you write reusable constructs in CDK?**

**Answer:**
Create a class extending `Construct`, encapsulate related resources, expose outputs through properties, and accept configuration props.
This allows:

* Parameterized reuse
* Testing
* Versioning via construct libraries

---

## **6. How does CDK handle deployments to multiple accounts/environments?**

**Answer:**
Use **Environments** (Account + Region):

```ts
env: { account: '123456789012', region: 'us-east-1' }
```

For cross-account:

* Use **CDK Bootstrap** (stores assets in bootstrap bucket)
* Use **ARN-based IAM trust policies**
* Use **SSM parameters or exports** for cross-account references
  CF doesn't allow direct cross-account references → must use SSM or manual wiring.

---

## **7. What is CDK Bootstrapping and why is it required?**

**Answer:**
Bootstrapping provisions:

* S3 bucket for assets
* ECR repository for Docker images
* IAM roles for deployment

Required for stacks using:

* Lambda from asset
* ECS images
* File assets
* `cdk deploy` with permissions

---

## **8. What is a CDK Context? What problems does it solve?**

**Answer:**
Context stores environment-specific values used during synthesis (e.g., VPC lookups, availability zones).
Prevents nondeterministic synthesis results and reduces API calls.
Stored in `cdk.context.json`.

---

## **9. What are Tokens in CDK?**

**Answer:**
Tokens represent values that are not known until deploy-time (e.g., ARNs, IDs).
Lazy evaluation placeholders → CloudFormation resolves them.
Example: `bucket.bucketArn`, `lambda.functionArn`.

---

## **10. How do you avoid circular dependencies in CDK?**

**Answer:**

* Use **SSM parameters or Lambda environment variables** instead of direct references.
* Avoid resources referencing each other’s attributes.
* Split constructs into multiple stacks with clear boundaries.

---

## **11. How do you unit test CDK code?**

**Answer:**
Use `assertions` module:

* `Template.fromStack()`
* `template.hasResourceProperties()`
* Snapshot testing for templates
* Tests validate IAM policies, resource properties, tags.

---

## **12. How does CDK handle assets like Lambda code or Docker images?**

**Answer:**
During `cdk synth`:

* Source code is packaged into `.zip`
* Uploaded to bootstrap S3 bucket or ECR
* CloudFormation template references those locations
  Assets are tracked in `.cdk.staging` and `cdk.out`.

---

## **13. Explain the difference between `CfnParameter`, `SSM parameter`, and CDK context.**

**Answer:**

* **CfnParameter:** dynamic *deploy-time* parameter (not recommended for most infra).
* **SSM Parameter:** runtime/config parameter stored in SSM. Good for cross-env consistency.
* **CDK Context:** *synth-time* configuration for environment lookups.

---

## **14. How do you override low-level CloudFormation properties in CDK?**

**Answer:**
Use the `.node.defaultChild` or L1 constructs.
Example:

```ts
(myBucket.node.defaultChild as s3.CfnBucket).property = ...
```

---

## **15. How do you manage secrets in CDK?**

**Answer:**

* Use **Secrets Manager** (`aws_secretsmanager.Secret`).
* For Lambda env vars, never store plaintext.
* Use AWS-managed KMS keys or CMKs.
* Optionally use CDK `Secret.fromSecretArn()` for imports.

---

## **16. How does CDK handle cross-stack references?**

**Answer:**
CDK creates:

* **CFN exports** in producer stack
* **CFN imports** in consumer stack

Limitations:

* Only works in same region/account
* Changing output names requires a replacement stack

Alternative: **SSM Parameter Store** based wiring.

---

## **17. How do you handle large Lambda layers or Docker Lambda in CDK?**

**Answer:**
Use:

* `lambda.Code.fromDockerBuild()` for container Lambda
* `lambda.LayerVersion` for layers

CDK will upload the docker image to ECR automatically via the asset system.

---

## **18. What are escape hatches in CDK?**

**Answer:**
Methods to directly manipulate underlying CFN resources when CDK doesn’t expose something:

* Use L1 constructs directly
* Use `CfnResource` overrides
* Modify `defaultChild` of L2 constructs

Used for edge cases.

---

## **19. How do you enforce security best practices using CDK?**

**Answer:**

* Use Aspects (apply encryption rules)
* Enable server-side encryption by default
* Enforce TLS/HTTPS policies (e.g., API Gateway)
* Use IAM scoped grants (`grantReadWrite`, etc.)
* Use AWS Config rules via CDK

---

## **20. How does CDK differ from Terraform?**

**Answer:**

* CDK uses **CloudFormation** → rollback safety, drift detection, stack dependency graph.
* Terraform uses its own engine.
* CDK provides **real programming languages** and abstractions (L2/L3).
* CDK is strongly tied to AWS ecosystem.

---

## **21. How do you properly version and distribute custom constructs?**

**Answer:**
Package constructs as **npm packages (TS)** or **PyPI (Python)** using **jsii**.
Version via SemVer; publish to internal registry; document APIs.
Allows infra standardization across teams.

---

## **22. What is the difference between `cdk deploy`, `cdk synth`, and `cdk diff`?**

**Answer:**

* **synth:** produce CF templates
* **diff:** show changes vs deployed stack
* **deploy:** apply changes → deploy stack via CloudFormation

---

## **23. How do you enforce deterministic builds in CDK?**

**Answer:**

* Lock context (`cdk.context.json`)
* Freeze dependency versions
* Avoid API lookups in synth unless cached
* Ensure consistent asset hashing

---

## **24. Why is it risky to use CloudFormation parameters in CDK?**

**Answer:**
They break:

* Deterministic synthesis
* Asset resolution
* Cross-stack references
* CI automation
  CDK philosophy → “inputs should be known at synth-time.”

---

## **25. How do you debug a failed CDK deployment?**

**Answer:**
Steps:

1. Check **CloudFormation Events** for failed resource.
2. Review synthesized template (`cdk.out`).
3. Validate IAM permissions (common root cause).
4. Check asset uploads (S3/ECR).
5. Run `cdk diff` to ensure expected template.
6. If needed, redeploy with `--rollback` disabled for inspection.

---
## AWS CDK CLI Commands Reference

The AWS CDK CLI provides comprehensive commands for managing infrastructure as code. Here are all the essential commands:[1]

### Project Initialization & Setup

**cdk init** - Creates a new CDK project from a template in the current directory, supporting multiple programming languages like TypeScript, Python, Java, and C#.[5][1]

**cdk bootstrap** - Prepares an AWS environment for CDK deployments by deploying the CDKToolkit stack, which includes resources like S3 buckets for asset storage and IAM roles for deployment permissions.[7][1]

### Stack Management

**cdk list (ls)** - Lists all CDK stacks and their dependencies from your CDK application, with options to show environment details and stack dependencies.[3][1][7]

**cdk synth** - Synthesizes CDK stacks into AWS CloudFormation templates without deploying them, useful for reviewing generated templates before deployment.[2][1]

**cdk deploy** - Deploys one or more CDK stacks into your AWS environment, creating or updating resources through CloudFormation.[1][7]

**cdk destroy** - Deletes one or more CDK stacks from your AWS environment, removing all associated resources.[7][1]

**cdk diff** - Performs a comparison to see infrastructure changes between your current CDK code and the deployed stack, showing what would change before deployment.[1][7]

### Resource Management

**cdk import** - Uses AWS CloudFormation resource imports to bring existing AWS resources into a CDK stack, enabling management of pre-existing infrastructure.[7][1]

**cdk migrate** - Migrates AWS resources, CloudFormation stacks, and CloudFormation templates into a new CDK project, facilitating transition from other IaC approaches.[1][7]

**cdk rollback** - Rolls back a failed deployment to the previous stable state.[7]

**cdk refactor** - Preserves deployed resources when refactoring code in your CDK application, allowing resource movement between stacks or within the same stack.[1][7]

### Development & Monitoring

**cdk watch** - Monitors a CDK app for deployable and hotswappable changes, automatically deploying updates during development.[7]

**cdk drift** - Detects configuration drift for resources you define, manage, and deploy using CDK, identifying differences between expected and actual state.[1][7]

**cdk doctor** - Inspects and displays useful information about your local CDK project and development environment for troubleshooting.[7][1]

### Configuration & Context

**cdk context** - Manages cached context values for your CDK application, storing and retrieving environment-specific information.[1]

**cdk flags** - Views and modifies feature flag configurations for the CDK CLI, controlling experimental or optional features.[1]

### Maintenance & Information

**cdk gc** - Garbage collects assets associated with the bootstrapped stack, cleaning up unused deployment artifacts.[7]

**cdk metadata** - Displays metadata associated with a CDK stack, providing insights into stack composition.[1]

**cdk notices** - Displays relevant notices for your CDK application, including security advisories and deprecation warnings.[7][1]

**cdk acknowledge (ack)** - Acknowledges a notice by issue number and hides it from displaying again.[7][1]

**cdk docs (doc)** - Opens CDK documentation in your browser for quick reference.[1]

**cdk cli-telemetry** - Enables or disables CLI telemetry collection for usage analytics.[7]

### Global Options

All commands support global options including `--app` (specify app command), `--profile` (AWS profile), `--region` (target region), `--verbose` (debug logging), `--help` (command help), and `--output` (synthesis output directory).
