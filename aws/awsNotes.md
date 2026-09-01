# AWS — Deep Dive Roadmap

We'll go from fundamentals → core services → networking → serverless → containers → IaC → security → cost → production → interview problems.

---

## 1. AWS Fundamentals

**Definition:** Amazon Web Services (AWS) is a cloud computing platform providing on-demand compute, storage, networking, and higher-level managed services over the internet, billed on a pay-as-you-go basis instead of requiring upfront hardware purchase.

**Regions & Availability Zones — Definition:** a **Region** is a physically separate geographic area (e.g. `us-east-1`) containing multiple, isolated data centers; an **Availability Zone (AZ)** is one of those data centers (or clusters of them) within a region, engineered to be isolated from failures in other AZs but connected by low-latency links — the basis for high availability (spread resources across AZs) and disaster recovery (spread across regions).

**Global vs regional vs zonal services — Definition:** a **global** service has one instance for the whole AWS account, spanning all regions (IAM, Route 53, CloudFront); a **regional** service is deployed independently per region (most services — S3 buckets, Lambda functions); a **zonal** resource lives in one specific AZ (an EC2 instance, an EBS volume).

**The AWS shared responsibility model — Definition:** AWS is responsible for the security **of** the cloud (physical data centers, hardware, host OS, network infrastructure); the customer is responsible for security **in** the cloud (data, IAM configuration, OS patching on EC2, network configuration) — the exact dividing line shifts depending on the service (e.g. Lambda shifts more responsibility to AWS than EC2 does).

**AWS Organizations — Definition:** a service for centrally managing multiple AWS accounts — consolidated billing, service control policies (SCPs) that set permission guardrails across all member accounts, and organizational units (OUs) for grouping accounts by team/environment.

**AWS CLI & SDKs — Definition:** the CLI is a command-line tool for interacting with AWS APIs from a terminal/scripts; SDKs (`boto3` for Python, `aws-sdk`/`@aws-sdk/client-*` for JS, etc.) provide the same API access from application code.

```bash
aws configure                       # set access key, secret, region
aws s3 ls                            # list buckets
aws ec2 describe-instances --region us-east-1
```

**Pricing models — Definition:**
- **On-demand** — pay per hour/second used, no commitment, most expensive per unit.
- **Reserved Instances / Savings Plans** — commit to 1 or 3 years of usage for a significant discount (up to ~72%).
- **Spot Instances** — bid on AWS's unused capacity at up to ~90% discount, but the instance can be reclaimed with short notice — suited to fault-tolerant, interruptible workloads (batch jobs, CI runners).

**AWS Well-Architected Framework — Definition:** AWS's set of six design pillars for evaluating cloud architectures — **Operational Excellence**, **Security**, **Reliability**, **Performance Efficiency**, **Cost Optimization**, and **Sustainability** — used as a structured review checklist for architecture decisions (referenced throughout this roadmap; see section 17 for applied patterns).

---

## 2. IAM (Identity & Access Management)

**Definition:** IAM is AWS's global service for controlling **who** (identity) can do **what** (permissions) to **which** resources — every single API call to any AWS service is authenticated and authorized through IAM.

**Users, groups, roles — Definition:**
- A **user** represents a person or application with long-term credentials (password, access keys).
- A **group** is a collection of users sharing the same set of permissions, for easier management.
- A **role** is an identity *without* long-term credentials, assumed temporarily by a user, an AWS service (e.g. an EC2 instance), or another AWS account — the preferred way to grant permissions to services, since it avoids embedding long-lived credentials anywhere.

**Policies — Definition:** JSON documents that define permissions. **Identity-based** policies attach to a user/group/role; **resource-based** policies attach directly to a resource (e.g. an S3 bucket policy), specifying who may access *that resource*, regardless of their own identity-based permissions.

**Policy structure:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": { "IpAddress": { "aws:SourceIp": "203.0.113.0/24" } }
    }
  ]
}
```

- **Effect** — `Allow` or `Deny` (an explicit `Deny` always overrides any `Allow`, from any policy).
- **Action** — the API operation(s) the statement applies to.
- **Resource** — the ARN(s) (Amazon Resource Names) the statement applies to.
- **Condition** — optional extra constraints (source IP, time of day, MFA presence).

**Principle of least privilege — Definition:** granting only the exact permissions an identity needs to perform its task, and nothing more — the central IAM design discipline, since overly broad permissions (`"Action": "*"`) turn any single compromised credential into a much larger blast radius.

**IAM roles for services — Definition:** e.g. an **EC2 instance role** or **Lambda execution role** — a role attached to a compute resource so the code running on it can call other AWS APIs (e.g. read from S3) using temporary, automatically-rotated credentials, without any access key ever being stored on the instance/function.

**Assume role & cross-account access — Definition:** `sts:AssumeRole` lets an identity in one account (or a role) temporarily obtain the permissions of a role in another account (or the same account), returning short-lived credentials via AWS STS (Security Token Service) — the standard mechanism for cross-account access and for federating external identity providers into AWS.

**Access keys vs temporary credentials** — access keys (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`) are long-lived and must be manually rotated/revoked if leaked; temporary credentials issued via STS/roles automatically expire (minutes to hours), making them the strongly preferred choice for anything running on AWS infrastructure itself.

**MFA (Multi-Factor Authentication)** — requiring a second factor (an authenticator app code, a hardware key) in addition to a password, especially for privileged/root-account access — an IAM best practice, often enforced via policy `Condition` on sensitive actions.

**IAM best practices:** never use the root account for daily work; enable MFA on root; grant permissions via roles/groups, not individual user policies; rotate access keys regularly; use least privilege; prefer temporary credentials over long-lived keys wherever possible.

---

## 3. Compute — EC2

**Definition:** EC2 (Elastic Compute Cloud) provides resizable virtual machines ("instances") in the cloud — the foundational IaaS compute service, giving full control over the OS and installed software, unlike higher-level managed compute (Lambda, ECS).

**Instance types & families — Definition:** EC2 instances come in families optimized for different workloads — `t`/`m` (general purpose), `c` (compute-optimized), `r`/`x` (memory-optimized), `i`/`d` (storage-optimized), `p`/`g` (GPU/accelerated computing) — each available in multiple sizes (`t3.micro`, `m5.large`, etc.).

**AMI (Amazon Machine Image) — Definition:** a template containing the OS, application server, and any pre-installed software used to launch an EC2 instance — either an AWS-provided base image, a community/Marketplace image, or a custom "golden image" snapshot of a previously-configured instance.

**Instance lifecycle — Definition:** `pending` → `running` → (`stopping` → `stopped`, or `shutting-down` → `terminated`). A **stopped** instance retains its EBS-backed storage and can be restarted (billed only for storage, not compute, while stopped); a **terminated** instance and its root volume (by default) are permanently deleted.

**Security Groups vs NACLs — Definition:** a **Security Group** is a **stateful** virtual firewall attached to instances/ENIs — if you allow inbound traffic on a port, the corresponding outbound response is automatically allowed, and rules are allow-only (no explicit deny). A **NACL** (Network ACL) is a **stateless** firewall at the subnet level — inbound and outbound rules must both be configured explicitly, and it supports both allow and deny rules, evaluated in numbered order. (Full networking context in section 5.)

**Key pairs & SSH access — Definition:** an EC2 key pair consists of a public key (stored by AWS, injected into the instance at launch) and a private key (downloaded once, kept by you) — used to SSH into a Linux instance instead of a password, since password auth is disabled by default on standard AMIs.

**EBS (Elastic Block Store) — Definition:** persistent, network-attached block storage volumes for EC2 instances — data survives instance stop/restart (unlike **instance store**, which is physically attached to the host and lost on stop/termination) — analogous to a virtual hard drive.

```bash
aws ec2 create-volume --size 20 --availability-zone us-east-1a --volume-type gp3
aws ec2 attach-volume --volume-id vol-123 --instance-id i-456 --device /dev/sdf
```

**Auto Scaling Groups (ASG) — Definition:** a group of EC2 instances managed together, automatically launching new instances (scaling out) or terminating them (scaling in) based on demand metrics (CPU utilization, request count) or a schedule, keeping the fleet size within configured min/max bounds and replacing unhealthy instances automatically.

**Elastic Load Balancers — Definition:** distribute incoming traffic across multiple targets (EC2 instances, containers, IPs) for availability and scalability.
- **ALB (Application Load Balancer)** — Layer 7 (HTTP/HTTPS), supports path/host-based routing, ideal for web applications and microservices.
- **NLB (Network Load Balancer)** — Layer 4 (TCP/UDP), ultra-low latency, handles millions of requests/sec, preserves client IP.
- **CLB (Classic Load Balancer)** — the legacy, largely deprecated load balancer type.

**Launch templates — Definition:** a reusable, versioned specification (AMI, instance type, security groups, user data script) for launching EC2 instances consistently — used by Auto Scaling Groups to know exactly what a new instance should look like.

---

## 4. Storage — S3

**Definition:** S3 (Simple Storage Service) is AWS's object storage service — stores files ("objects") of any type/size inside "buckets," accessed over HTTP(S), designed for effectively unlimited scale and 99.999999999% ("11 nines") durability.

**Buckets & objects — Definition:** a **bucket** is a globally-uniquely-named container for objects (with a region assigned at creation); an **object** is a file plus metadata, identified within a bucket by a **key** (its full "path"-like name) — S3 has no true nested folder structure; the `/` in a key is just a convention consoles/tools render as folders.

**Storage classes — Definition:** different cost/retrieval-time tiers for the same object storage: **Standard** (frequent access, millisecond retrieval), **Standard-IA** (Infrequent Access, cheaper storage + retrieval fee), **One Zone-IA** (cheaper still, single AZ — less durable against AZ loss), **Glacier**/**Glacier Deep Archive** (archival, retrieval takes minutes to hours, lowest storage cost) — chosen based on how often and how quickly data needs to be accessed.

**Versioning — Definition:** when enabled on a bucket, S3 keeps every version of an object ever written (including on overwrite or delete, which creates a "delete marker" rather than truly erasing data) — protects against accidental overwrite/deletion, at the cost of storing every historical version.

**Lifecycle policies — Definition:** rules that automatically transition objects between storage classes (e.g. Standard → IA after 30 days → Glacier after 90 days) or expire (delete) them after a set time, without manual intervention.

**Bucket policies vs ACLs — Definition:** a **bucket policy** is a resource-based IAM policy (JSON) attached to the bucket, controlling who can access it and how — the modern, recommended approach; **ACLs** are a legacy, more limited access-control mechanism generally discouraged in favor of bucket policies + IAM.

**Presigned URLs — Definition:** a temporary, time-limited URL generated (using the credentials of an IAM identity with access) that grants whoever holds it access to a specific S3 object — the standard way to let a client upload/download a private object directly without making the object or bucket public.

```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const url = await getSignedUrl(client, new PutObjectCommand({ Bucket: 'my-bucket', Key: 'upload.jpg' }), { expiresIn: 3600 });
```

**Static website hosting** — S3 can serve a bucket's contents directly as a static website (HTML/CSS/JS), typically fronted by CloudFront for HTTPS and caching (section 11).

**S3 event notifications — Definition:** S3 can automatically trigger a Lambda function, SQS queue, or SNS topic whenever an object is created/deleted in a bucket — the basis for many event-driven, serverless data-processing pipelines (e.g. resize an image the moment it's uploaded).

**Multipart uploads — Definition:** splitting a large object upload into multiple independently-uploaded parts (in parallel, with retry-per-part), then having S3 assemble them — required for objects over 5GB, and beneficial for reliability/throughput on any large upload.

**Consistency model** — S3 now provides **strong read-after-write consistency** for all operations (a `GET` immediately after a successful `PUT` always returns the latest data) — a change from S3's older, weaker eventual-consistency model.

---

## 5. Networking — VPC

**Definition:** a VPC (Virtual Private Cloud) is a logically isolated, private network within AWS where you launch resources (EC2, RDS, Lambda-in-VPC, etc.), with full control over IP addressing, subnets, and routing — the networking foundation almost every other AWS service sits on top of.

**Subnets — Definition:** a subnet is a range of IP addresses within a VPC, tied to a single Availability Zone. A **public subnet** has a route to an Internet Gateway (resources can have public IPs and be reached from the internet); a **private subnet** has no direct internet route (used for databases, internal services).

**Route tables — Definition:** a set of rules ("routes") determining where network traffic from a subnet is directed — e.g. "traffic to `0.0.0.0/0` goes to the Internet Gateway" makes a subnet effectively public.

**Internet Gateway — Definition:** a VPC component that allows two-way communication between resources in the VPC (with public IPs) and the public internet — one per VPC, attached, then referenced by a public subnet's route table.

**NAT Gateway vs NAT Instance — Definition:** both let resources in a **private** subnet initiate outbound connections to the internet (e.g. to download updates) while remaining unreachable from the internet inbound. A **NAT Gateway** is a fully-managed AWS service (recommended — no maintenance, scales automatically); a **NAT Instance** is a self-managed EC2 instance running NAT software (legacy, more control but more operational burden).

**Security Groups (stateful) vs NACLs (stateless)** — see section 3; in practice, Security Groups are the primary, day-to-day firewall control (attached to instances/resources), while NACLs are a coarser, subnet-wide secondary layer of defense.

**VPC Peering — Definition:** a private network connection between two VPCs (same or different accounts/regions) that lets resources in each communicate using private IP addresses as if on the same network — non-transitive (peering A↔B and B↔C does *not* let A reach C).

**VPC Endpoints — Definition:** a private connection from a VPC directly to a supported AWS service, without traffic ever traversing the public internet. A **Gateway Endpoint** (S3, DynamoDB only) is added directly as a route table target, free of charge; an **Interface Endpoint** (most other services) provisions an ENI with a private IP inside your VPC, billed hourly.

**CIDR blocks & subnetting — Definition:** CIDR notation (`10.0.0.0/16`) specifies an IP address range and its size via the prefix length — a VPC is assigned a CIDR block, then divided into smaller CIDR ranges for individual subnets; correct subnet sizing up front avoids painful re-addressing later.

**Elastic IPs — Definition:** a static, public IPv4 address you allocate to your account and attach to a resource (typically a NAT Gateway or an EC2 instance) — unlike an auto-assigned public IP, it persists across stop/start and can be remapped to a different instance.

---

## 6. Serverless — Lambda

**Definition:** AWS Lambda is a **Function-as-a-Service (FaaS)** compute service that runs code in response to events without provisioning or managing servers — you supply a function, AWS handles the underlying compute, scaling (including down to zero), and patching.

**Function handlers — Definition:** the entry-point function Lambda invokes for each event, receiving the event payload and a context object.

```js
export const handler = async (event, context) => {
  const body = JSON.parse(event.body);
  return { statusCode: 200, body: JSON.stringify({ received: body }) };
};
```

**Triggers/event sources — Definition:** the AWS services or events that invoke a Lambda function — API Gateway (HTTP requests), S3 (object created), DynamoDB Streams (table changes), SQS (new message), EventBridge (scheduled/pattern-matched events), and many more.

**Cold starts — Definition:** the latency incurred the first time a new Lambda execution environment is initialized (downloading code, starting the runtime, running any top-level init code) before the handler can run — subsequent invocations reuse a "warm" environment and skip this cost, until AWS eventually recycles it.

**Execution environment & runtime — Definition:** each Lambda invocation runs inside a sandboxed micro-VM (Firecracker) with a specific language runtime (Node.js, Python, Java, etc., or a custom runtime/container image) — AWS may reuse an environment for consecutive invocations (avoiding a cold start) or spin up new ones in parallel under concurrent load.

**Memory/timeout configuration — Definition:** memory (128MB–10GB) is configured per function, and CPU is allocated *proportionally* to memory — increasing memory can therefore also speed up CPU-bound functions; timeout caps how long a single invocation may run (max 15 minutes).

**Environment variables** — key-value configuration injected into the function's environment at invoke time, editable without a code change/redeploy — sensitive values should reference Secrets Manager/Parameter Store rather than being stored in plaintext (section 14).

**Layers — Definition:** a way to package shared code/dependencies (a common library, a native binary) separately from the function itself, attached to multiple functions — keeps deployment packages smaller and dependencies reusable/versioned independently.

**Concurrency — Definition:** the number of simultaneous executions of a function.
- **Reserved concurrency** — caps (and guarantees) the maximum simultaneous executions for a function, protecting downstream systems (like a database) from being overwhelmed.
- **Provisioned concurrency** — keeps a specified number of execution environments pre-initialized and warm at all times, eliminating cold starts for that portion of traffic, at a continuous cost even when idle.

**Lambda + API Gateway** — the standard pattern for building a serverless HTTP API: API Gateway receives the HTTP request and invokes a Lambda function per route (or a single function handling all routes via internal routing).

**Lambda + S3/DynamoDB triggers** — event-driven processing: e.g. a Lambda automatically resizes an image the moment it's uploaded to S3, or reacts to a row change captured by a DynamoDB Stream.

**Step Functions — Definition:** a serverless orchestration service that coordinates multiple Lambda functions (and other AWS services) into a visual state machine — handling sequencing, branching, parallel execution, retries, and error handling declaratively, instead of that logic being hand-written inside one large Lambda.

---

## 7. Databases

### RDS (managed relational databases)

**Definition:** RDS is a managed service for relational databases (PostgreSQL, MySQL, MariaDB, SQL Server, Oracle) — AWS handles provisioning, patching, and backups, while you retain a standard relational schema and SQL access.

**Multi-AZ vs Read Replicas — Definition:**
- **Multi-AZ** — RDS maintains a synchronously-replicated standby instance in a different AZ, purely for **availability**: if the primary fails, RDS automatically fails over to the standby (the standby is not readable directly).
- **Read Replicas** — asynchronously-replicated, independently **readable** copies (potentially in different regions), used to scale out read traffic — not automatically used for failover, though one can be manually promoted.

**Automated backups & snapshots — Definition:** RDS automatically takes daily backups plus continuous transaction logs, enabling point-in-time restore within the retention window; **manual snapshots** are user-triggered, full backups retained until explicitly deleted (useful before a risky change).

### DynamoDB (managed NoSQL)

**Definition:** DynamoDB is a fully-managed, serverless key-value/document NoSQL database, designed for single-digit-millisecond latency at virtually any scale, with no server management or manual sharding required.

**Partition keys & sort keys — Definition:** the **partition key** determines which physical partition an item is stored on (DynamoDB hashes it) — it's the primary lever for how evenly data and traffic spread across the underlying infrastructure; an optional **sort key** orders items sharing the same partition key, enabling efficient range queries within a partition (together they form a "composite primary key").

**Read/write capacity — Definition:**
- **Provisioned** — you specify a fixed read/write throughput capacity in advance (can enable auto scaling within a range); cheaper for predictable, steady workloads.
- **On-demand** — DynamoDB scales capacity automatically per actual traffic, billed per request; simpler, better for unpredictable/spiky workloads.

**GSI / LSI — Definition:** a **Global Secondary Index** has its own partition (and optional sort) key, independent of the base table's, enabling queries on different attributes — effectively a separate, automatically-maintained index table. A **Local Secondary Index** shares the base table's partition key but has a different sort key, must be created at table-creation time, and is queried within a single partition.

**DynamoDB Streams — Definition:** an ordered, time-based log of item-level changes (insert/update/delete) in a table, consumable by Lambda (or other consumers) — the mechanism behind reacting to data changes in near real time.

### ElastiCache

**Definition:** a managed in-memory data store service (Redis or Memcached) used for caching, session storage, or as a fast key-value store in front of a slower primary database — same role as self-hosted Redis in the Node.js notes' caching section, but fully managed.

### Aurora (brief)

**Definition:** AWS's proprietary, cloud-native relational database engine, MySQL/PostgreSQL-compatible at the wire protocol level, offering higher throughput and built-in high availability compared to standard RDS engines, at a higher cost.

**When to choose RDS vs DynamoDB** — RDS/Aurora for relational data with complex queries, joins, and transactions across normalized tables; DynamoDB for massive scale, predictable access patterns keyed by a known partition key, and workloads that don't need ad-hoc relational queries.

---

## 8. API Layer

**API Gateway — Definition:** a fully-managed service for creating, publishing, and securing APIs that front backend compute (Lambda, EC2, any HTTP endpoint) — handles routing, throttling, authorization, and request/response transformation at the edge, before a request ever reaches your backend code.

**REST APIs vs HTTP APIs — Definition:** API Gateway offers two API types — the older, feature-rich **REST API** (request validation, API keys, usage plans, request/response transformation) and the newer, lighter-weight, cheaper, lower-latency **HTTP API** (fewer features, but sufficient for most Lambda-proxy use cases).

**Resources, methods, stages — Definition:** a **resource** is a URL path segment; a **method** is an HTTP verb bound to a resource, mapped to a backend integration (e.g. a specific Lambda function); a **stage** (e.g. `dev`, `prod`) is a named, independently-deployable snapshot of the API's configuration, each with its own URL and (optionally) its own throttling/variables.

**Authorizers — Definition:** the mechanism that authenticates/authorizes each incoming request before it reaches the backend — **IAM** authorizers (SigV4-signed requests from other AWS identities), **Cognito** authorizers (validate a Cognito user pool JWT), or **Lambda authorizers** (a custom function that inspects the request and returns an allow/deny policy, for arbitrary auth schemes).

**Throttling & usage plans — Definition:** rate limits (requests/second) and burst limits applied per API, per stage, or per API key, protecting the backend from being overwhelmed and enabling tiered access (e.g. free vs paid API consumers via **usage plans** + **API keys**).

**ALB as an entry point** — for containerized/EC2-hosted APIs (rather than Lambda), an Application Load Balancer often serves the same "front door" role API Gateway plays for serverless APIs.

**CloudFront in front of APIs** — adding a CDN layer (section 11) in front of an API can cache cacheable GET responses at edge locations and provide a stable custom domain/HTTPS termination point, reducing latency for globally-distributed clients.

---

## 9. Containers

**ECS (Elastic Container Service) — Definition:** AWS's native container orchestration service for running Docker containers at scale, without needing to run/manage Kubernetes yourself.

**Task definitions — Definition:** a JSON blueprint describing one or more containers to run together (image, CPU/memory, ports, environment variables, IAM role) — analogous to a Kubernetes Pod spec.

**Services — Definition:** an ECS construct that keeps a specified number of task instances running continuously, replacing failed tasks automatically, and (optionally) registering them with a load balancer — the long-running-application analog to a one-off task run.

**Fargate vs EC2 launch type — Definition:**
- **Fargate** — serverless container hosting: you specify CPU/memory per task, AWS provisions and manages the underlying compute entirely — no EC2 instances to patch/manage.
- **EC2 launch type** — you provision and manage a cluster of EC2 instances yourself, onto which ECS schedules tasks — more control (instance types, spot usage, GPU access) at the cost of managing the underlying servers.

**EKS (Elastic Kubernetes Service) — Definition:** AWS's managed Kubernetes control plane — for teams that want/need standard, portable Kubernetes (multi-cloud strategy, existing K8s tooling/expertise) rather than AWS-proprietary ECS.

**ECR (Elastic Container Registry) — Definition:** a managed, private Docker container image registry, integrated with IAM for access control — where container images are pushed to and pulled from by ECS/EKS/Lambda (container-image functions).

**When to choose ECS vs EKS vs Lambda** — Lambda for event-driven, typically short-lived, stateless workloads with minimal ops overhead; ECS for long-running containerized applications without needing Kubernetes-specific features/portability; EKS when Kubernetes itself (its API, ecosystem, or multi-cloud portability) is a requirement.

---

## 10. Infrastructure as Code

**Definition:** Infrastructure as Code (IaC) is the practice of defining cloud infrastructure in versioned, declarative configuration files rather than manually clicking through a console — enabling repeatable, reviewable, and automatable infrastructure changes.

**CloudFormation — Definition:** AWS's native IaC service — you author a **template** (JSON/YAML) describing desired resources, and CloudFormation creates a **stack** that provisions, updates, or deletes those resources to match, tracking their relationships and handling dependency ordering automatically.

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-app-uploads
```

**Change sets — Definition:** a preview of exactly what a CloudFormation stack update would create/modify/delete, reviewable *before* actually applying it — a safety mechanism against unintended infrastructure changes.

**AWS CDK (Cloud Development Kit) — Definition:** a higher-level IaC toolkit that lets you define infrastructure using real programming languages (TypeScript, Python, etc.) instead of raw YAML/JSON — CDK code synthesizes down to a CloudFormation template under the hood, adding type safety, loops/conditionals, and reusable abstractions (constructs).

```ts
const bucket = new s3.Bucket(this, 'MyBucket', { versioned: true });
```

**Terraform (brief)** — a popular multi-cloud IaC tool (HashiCorp) using its own declarative language (HCL), supporting AWS and other providers with a single consistent workflow — an alternative to CloudFormation/CDK for teams wanting cloud-agnostic tooling.

**SAM (Serverless Application Model) — Definition:** an extension of CloudFormation with a simplified syntax specifically for serverless resources (Lambda, API Gateway, DynamoDB), plus local testing/debugging tooling (`sam local`) — a thinner layer purpose-built for serverless apps rather than CDK's general-purpose approach.

**Drift detection — Definition:** CloudFormation's ability to detect when a stack's actual resources have been manually modified outside of CloudFormation (e.g. someone changed a setting in the console), so infrastructure doesn't silently diverge from what's declared in code.

---

## 11. Content Delivery & DNS

**CloudFront — Definition:** AWS's Content Delivery Network (CDN) — caches content at edge locations around the world, close to end users, reducing latency and offloading traffic from the origin (S3, an ALB, a custom server).

**Distributions, origins, cache behaviors — Definition:** a **distribution** is a CloudFront configuration tied to one or more **origins** (where content actually comes from); **cache behaviors** define, per path pattern, how requests are handled — which origin to use, what to cache and for how long, and what viewer protocols/headers are allowed.

**Cache invalidation — Definition:** explicitly purging cached content from edge locations before its TTL naturally expires — necessary when content changes but you can't wait for the cache to expire on its own (billed per invalidation path after a free monthly allowance).

**Route 53 — Definition:** AWS's managed, highly-available DNS service — also supports domain registration and health-check-based routing.

**Hosted zones — Definition:** a container for DNS records for a specific domain, telling Route 53 how to respond to queries for that domain and its subdomains.

**Record types — Definition:** standard DNS record types (`A` — IPv4 address, `AAAA` — IPv6, `CNAME` — alias to another domain name, `MX` — mail routing), plus Route 53's own **Alias** record type, which behaves like a CNAME but works at a zone apex and can point directly at AWS resources (an ALB, a CloudFront distribution) at no extra query cost.

**Routing policies — Definition:**
- **Weighted** — split traffic across multiple resources by assigned percentage (e.g. gradual canary rollout).
- **Latency-based** — routes each user to the region with the lowest latency for them.
- **Failover** — routes to a primary resource, automatically switching to a secondary if the primary's health check fails.
- **Geolocation** — routes based on the user's geographic location (e.g. compliance, localization).

**Health checks — Definition:** Route 53 periodically probes an endpoint (HTTP/HTTPS/TCP) and considers it unhealthy after repeated failures — feeds into failover routing to automatically stop directing traffic to a downed resource.

---

## 12. Messaging & Event-Driven Architecture

**SQS (Simple Queue Service) — Definition:** a fully-managed message queue service that decouples producers from consumers — a producer sends messages onto a queue, and one or more consumers poll and process them independently, at their own pace.

**Standard vs FIFO queues — Definition:** **Standard** queues offer at-least-once delivery, best-effort ordering, and virtually unlimited throughput; **FIFO** queues guarantee exactly-once processing and strict message ordering (per message group), at lower throughput limits — chosen based on whether order/duplication actually matters for the use case.

**Visibility timeout — Definition:** when a consumer receives a message from the queue, that message becomes temporarily invisible to other consumers for a configured duration; if the consumer doesn't delete it (signaling successful processing) within that window, the message becomes visible again for another consumer to retry.

**Dead-letter queues (DLQ) — Definition:** a separate queue that a message is automatically routed to after failing processing (visibility timeout expiring without deletion) a configured number of times — isolates poison messages from blocking the main queue and enables inspection/reprocessing later.

**SNS (Simple Notification Service) — Definition:** a fully-managed pub/sub messaging service — a producer publishes a message to a **topic**, and every subscriber (email, SMS, Lambda, SQS queue, HTTP endpoint) subscribed to that topic receives it — one-to-many, unlike SQS's queue model.

**EventBridge — Definition:** a serverless event bus for routing events between AWS services, SaaS applications, and custom applications, based on **rules** that match event patterns (JSON structure/content) and route matching events to one or more targets — more sophisticated content-based routing than SNS's simple topic-subscription model, and the modern default for event-driven architectures on AWS.

**Fan-out pattern (SNS + SQS) — Definition:** publishing one message to an SNS topic that has multiple SQS queues subscribed, so each queue (and its independent consumer) receives its own durable copy of the message — combines SNS's broadcast capability with SQS's durability/retry guarantees, which a raw SNS subscriber lacks on its own.

**Choosing SQS vs SNS vs EventBridge** — SQS when you need a durable work queue for one (or a pool of) consumer(s) to process at their own pace; SNS for simple one-to-many broadcast with minimal routing logic; EventBridge when you need content-based routing rules, third-party SaaS integrations, or a more general-purpose event bus across many event producers/consumers.

---

## 13. Monitoring & Observability

**CloudWatch — Definition:** AWS's native monitoring and observability service, collecting metrics, logs, and alarms across essentially every AWS service.

**Metrics — Definition:** time-series numerical data points (CPU utilization, request count, error rate) automatically published by most AWS services, plus custom metrics an application can publish itself.

**Logs & Log Groups — Definition:** CloudWatch Logs collects and stores log output (e.g. every Lambda function's `console.log` output, application logs shipped via an agent), organized into **Log Groups** (typically one per application/function) containing **Log Streams**.

**Alarms — Definition:** a CloudWatch construct that watches a metric against a threshold and, when breached for a configured duration, triggers an action — notifying via SNS, triggering Auto Scaling, or invoking a Lambda remediation function.

**Dashboards — Definition:** customizable visual widgets combining multiple metrics/logs into a single at-a-glance operational view.

**CloudTrail — Definition:** logs every API call made within an AWS account — who did what, when, from where — the audit trail for security investigation and compliance, distinct from CloudWatch's operational/performance focus.

**X-Ray — Definition:** a distributed tracing service that follows a single request as it travels across multiple services (API Gateway → Lambda → DynamoDB, etc.), visualizing the full call graph and per-segment latency — essential for diagnosing where time is spent in a microservices/serverless architecture.

**AWS Config — Definition:** continuously records the configuration state of AWS resources and evaluates them against defined rules (e.g. "no S3 bucket should be publicly writable"), flagging non-compliant resources — a compliance/governance tool, complementary to CloudTrail's activity log.

---

## 14. Security

**Encryption at rest & in transit — Definition:** **at rest** encryption protects stored data (on disk, in a database, in S3) so it's unreadable without the decryption key even if the underlying storage is somehow accessed directly; **in transit** encryption (TLS/HTTPS) protects data as it moves across the network from interception. Most AWS storage services support at-rest encryption with a single configuration flag.

**KMS (Key Management Service) — Definition:** a managed service for creating and controlling the cryptographic keys used to encrypt data across AWS services — handles key storage, rotation, and access-controlled usage (via IAM policy) without your application ever handling raw key material directly.

**Secrets Manager vs Parameter Store — Definition:** both store configuration/secrets outside application code. **Secrets Manager** is purpose-built for secrets — supports automatic rotation (e.g. rotating a database password on a schedule) and charges per secret. **Parameter Store** (part of Systems Manager) is a more general key-value store, free for standard parameters, without built-in rotation — Secrets Manager for anything needing rotation or higher sensitivity; Parameter Store for general configuration.

```bash
aws secretsmanager get-secret-value --secret-id prod/db/password
aws ssm get-parameter --name /myapp/api-url --with-decryption
```

**WAF (Web Application Firewall) — Definition:** filters and monitors HTTP requests to a CloudFront distribution, ALB, or API Gateway based on configurable rules (block SQL injection patterns, rate-limit an IP, allow only specific geographies) — an application-layer firewall, distinct from network-layer Security Groups/NACLs.

**Shield — Definition:** AWS's managed DDoS protection service — **Shield Standard** is automatically active (free) for all customers against common network/transport-layer attacks; **Shield Advanced** (paid) adds more sophisticated attack detection, mitigation, and cost protection for larger/higher-risk applications.

**GuardDuty — Definition:** a managed threat-detection service that continuously analyzes CloudTrail, VPC Flow Logs, and DNS logs using machine learning and threat intelligence to identify potentially malicious or unauthorized activity (e.g. an EC2 instance communicating with a known malware command-and-control server).

**Compliance basics** — AWS provides certifications (SOC 2, PCI-DSS, HIPAA eligibility) covering its side of the shared responsibility model; achieving actual compliance still requires the customer to correctly configure their own resources, access controls, and data handling.

---

## 15. Cost Optimization

**Cost Explorer & Budgets — Definition:** **Cost Explorer** visualizes historical AWS spending, broken down by service/tag/region, with forecasting; **Budgets** lets you set spending thresholds and receive alerts (or trigger automated actions) when actual or forecasted spend approaches/exceeds them.

**Right-sizing instances — Definition:** matching EC2/RDS instance types and sizes to actual observed CPU/memory/network utilization, rather than over-provisioning "just in case" — a frequent, high-impact cost-optimization action informed by CloudWatch metrics.

**Reserved Instances vs Savings Plans vs Spot** — see section 1; choosing the right commitment/discount model for each workload's predictability and interruption-tolerance is one of the largest levers on total AWS spend.

**S3 storage class tiering** — see section 4; **S3 Intelligent-Tiering** automatically moves objects between access tiers based on observed access patterns, removing the need to manually predict and configure lifecycle rules.

**Identifying idle/unused resources** — unattached EBS volumes, unused Elastic IPs (billed when not attached to a running instance), idle load balancers, and forgotten dev/test resources are common sources of avoidable spend — periodic audits (or automated tools like AWS Trusted Advisor) catch these.

**Tagging strategy for cost allocation — Definition:** consistently tagging resources (`Team: payments`, `Environment: prod`) so Cost Explorer can attribute spend to teams/projects/environments — essential for cost accountability once an account has more than a handful of resources.

---

## 16. CI/CD on AWS

**CodePipeline — Definition:** AWS's managed CI/CD orchestration service — defines a multi-stage pipeline (source → build → test → deploy) that automatically runs whenever code changes, coordinating other AWS developer tools (and third-party ones like GitHub) at each stage.

**CodeBuild — Definition:** a managed build service that compiles source code, runs tests, and produces deployable artifacts, defined via a `buildspec.yml` file — AWS's equivalent of a CI build runner (similar role to GitHub Actions/CircleCI, but AWS-native and easily plugged into CodePipeline).

```yaml
# buildspec.yml
phases:
  install: { commands: ["npm ci"] }
  build: { commands: ["npm run build", "npm test"] }
artifacts:
  files: ["dist/**/*"]
```

**CodeDeploy — Definition:** automates deploying application revisions to EC2, on-premises servers, Lambda, or ECS — handling the mechanics of a deployment strategy (see below) including automatic rollback on failure.

**Deployment strategies — Definition:**
- **Blue/green** — a completely new ("green") environment is stood up alongside the existing ("blue") one; traffic is cut over once the green environment is verified healthy, and blue is kept briefly for instant rollback.
- **Canary** — a small percentage of traffic is shifted to the new version first, gradually increased once it's confirmed healthy — limits the blast radius of a bad deploy compared to shifting all traffic at once.
- **Rolling** — instances/tasks are updated to the new version in batches, replacing the old version gradually across the fleet, without a separate parallel environment.

**Deploying to Lambda / ECS / EC2** — the specific mechanics differ (Lambda: shifting traffic between function versions/aliases; ECS: replacing tasks in a service; EC2: CodeDeploy agent installing a new revision on each instance), but the same blue/green/canary/rolling strategy concepts apply across all three.

---

## 17. Architecture Patterns

**Three-tier architecture — Definition:** the classic web application pattern — a **presentation** tier (load balancer + web/app servers or a frontend served via CloudFront/S3), an **application** tier (business logic, often EC2/ECS/Lambda), and a **data** tier (RDS/DynamoDB) — typically spread across public and private subnets within a VPC for defense in depth.

**Serverless architecture patterns** — API Gateway + Lambda + DynamoDB is the canonical fully-serverless API stack: no servers to manage, scales to zero, pay only for actual usage — well-suited to variable/unpredictable traffic and rapid iteration.

**Microservices on AWS** — independent services (each potentially ECS/EKS/Lambda-based) communicating via API Gateway (synchronous) and/or SQS/SNS/EventBridge (asynchronous), each with its own datastore — trades operational complexity for independent deployability/scalability of each service.

**Multi-region / disaster recovery strategies — Definition:** strategies for surviving a regional outage, in increasing cost/complexity order: **Backup & Restore** (cheapest, slowest recovery — just have backups in another region), **Pilot Light** (minimal always-on replica of critical core, scaled up on failover), **Warm Standby** (a scaled-down but fully functional replica, ready to scale up), **Multi-Site Active/Active** (full production capacity live in multiple regions simultaneously, most expensive, near-zero recovery time).

**High availability design** — spreading resources across multiple AZs (minimum) or regions (for DR), eliminating single points of failure (no single EC2 instance/AZ that the whole system depends on), and designing for graceful degradation when a dependency fails.

**Well-Architected pillars in practice** — a genuinely well-architected system explicitly weighs Reliability (multi-AZ, retries, DLQs) against Cost Optimization (right-sizing, Spot where appropriate) against Performance (caching, CDN, appropriate database choice) against Security (least privilege, encryption) — rather than optimizing any single pillar in isolation.

---

## 18. Production Engineering

**Environment separation (dev/staging/prod accounts) — Definition:** using **separate AWS accounts** (not just separate resources in one account) per environment, tied together via AWS Organizations — provides a hard security/billing/blast-radius boundary between environments, so a mistake or compromise in dev can't touch production resources or credentials at all.

**Secrets & config management** — see section 14; production secrets belong in Secrets Manager/Parameter Store, referenced by IAM-role-based access, never hardcoded or passed as plain environment variables checked into source control.

**Zero-downtime deployments** — achieved via the deployment strategies in section 16 (blue/green, rolling, canary) combined with health checks, so a load balancer/ASG never routes traffic to an instance that isn't ready.

**Auto scaling in production** — configuring Auto Scaling Groups (EC2) or service auto scaling (ECS/DynamoDB) with sensible target metrics and cooldown periods, tested against realistic load, rather than left at default/manual-only capacity.

**Logging & alerting strategy** — centralizing logs (CloudWatch Logs, or shipped to a third-party aggregator), with CloudWatch Alarms (or a third-party APM) wired to actually page/notify someone for the metrics that matter (error rate, latency, saturation) — not just collected and never looked at.

**Incident response basics** — a documented on-call/escalation process, runbooks for common failure modes, and CloudTrail/X-Ray/CloudWatch Logs as the primary tools for reconstructing what happened during an incident after the fact.

**Common pitfalls & anti-patterns:**
- Using the AWS root account for everyday work instead of IAM roles/users.
- Overly broad IAM policies (`"Action": "*", "Resource": "*"`) instead of least privilege.
- Public S3 buckets/RDS instances left open by default misconfiguration.
- No multi-AZ / single point of failure in "production" architecture.
- Hardcoded credentials in code or environment variables instead of Secrets Manager + IAM roles.
- No tagging strategy, making cost attribution and cleanup of unused resources difficult.
