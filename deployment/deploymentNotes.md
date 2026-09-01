# Deployment Process — Deep Dive Roadmap

We'll go from fundamentals → CI/CD → deployment strategies → environments & config → rollback & verification → architecture-specific patterns → production discipline.

*This is the cross-cutting, stack-agnostic deployment reference for the workspace — the Angular, React, Node.js, Java, Python, AWS, and Docker/Kubernetes notes each cover their own stack's production-engineering specifics; this file focuses on the end-to-end deployment process itself, applicable regardless of which stack it's moving.*

---

## 1. Deployment Fundamentals

**Definition:** deployment is the process of taking a specific version of software and making it run in a target environment — the mechanical act of getting built code from a repository onto the infrastructure that actually serves real (or test) traffic.

**Deploy vs release — Definition:** these are related but genuinely distinct concepts, and conflating them is a common source of confusion. **Deploying** software means it is running somewhere; **releasing** it means it is actually available to (some or all) end users. Decoupling the two — deploying new code to production while keeping it hidden behind a feature flag (section 10) or only reachable by internal traffic — lets a team verify a deployment is healthy *before* committing to a release, and is the foundation of several safer deployment strategies (canary, dark launches, section 10).

**The software delivery lifecycle overview:**

```
Code committed → Build → Test (CI, section 5) → Package (artifact, section 4)
    → Deploy to staging → Verify → Deploy to production → Release (if decoupled)
    → Monitor (section 15) → (Rollback if needed, section 14)
```

**Artifacts vs source code — Definition:** source code is the human-authored, version-controlled input; an **artifact** is the built, packaged output produced from it (a Docker image, a compiled JAR, a bundled JS application, a Python wheel) — deployment moves and runs **artifacts**, never source code directly, which is precisely why the build step (section 4) exists as a distinct, essential phase between committing code and deploying it.

**Idempotent deployments — Definition:** a deployment process that produces the same end result no matter how many times it's run, or whether it's run from a clean slate or re-applied on top of a partially-completed previous attempt — essential for reliability, since a deployment that fails partway through and needs to be retried (or re-triggered by an automated pipeline) must not corrupt the target environment or produce a different result the second time; infrastructure-as-code tools (section 9) and container-based deployments are naturally more idempotent than older "run a sequence of imperative shell commands on a live server" approaches.

**Why deployment process discipline matters** — an application can be architecturally excellent and still fail in production due entirely to a fragile, manual, or poorly-understood deployment process — deployment is frequently the highest-risk moment in a software system's entire lifecycle (the point where theoretical correctness meets real infrastructure and real traffic), which is exactly why the rest of this roadmap treats it as a first-class engineering discipline, not an afterthought bolted on once the "real" application work is done.

---

## 2. Environments & Environment Parity

**Definition:** an environment is a distinct, isolated deployment target — a separate running copy of the infrastructure a system depends on (servers, databases, configuration) — used to progressively validate a change before it reaches real users.

**Dev, staging, QA, production — Definition:**
- **Development (dev)** — where active development happens, often a local machine or a shared, frequently-changing environment; lowest stakes, expected to be unstable.
- **Staging** — a environment intended to closely mirror production, used for final validation before a real release — the last checkpoint before real users are affected.
- **QA** — a dedicated environment for manual or automated quality-assurance testing, sometimes merged with staging at smaller organizations, kept separate at larger ones specifically so active testing doesn't collide with final pre-release validation.
- **Production (prod)** — the environment real users actually interact with; highest stakes, and the ultimate target every other environment exists to de-risk reaching safely.

**Environment parity ("dev/prod parity") — Definition:** the principle (part of the well-known "12-factor app" methodology) that non-production environments should resemble production as closely as practical — same or equivalent database engine, same OS/runtime versions, same general architecture — since the entire point of a staging environment is to catch problems *before* production; a staging environment that differs meaningfully from production (a different database, mocked-out dependencies that don't exist in the real environment) fails at exactly the job it exists to do, letting bugs slip through that only manifest against production's real configuration.

**Environment-specific configuration (recap)** — see the Node.js/Java/Python/Angular/React notes' respective production sections; the same application code runs across every environment, with only externalized configuration (section 7) — API URLs, feature flags, resource sizing — differing between them, never environment-specific code branches baked into the application itself.

**Ephemeral environments — Definition:** short-lived, automatically-provisioned environments created for a specific purpose (commonly, one per open pull request) and torn down once no longer needed — lets a reviewer or the PR author interact with a fully-running version of *that specific change* before it merges, catching integration issues earlier than a shared, long-lived staging environment (where multiple in-progress changes are mixed together) would, at the cost of the infrastructure automation needed to provision/deprovision environments on demand reliably.

**Data in non-production environments — Definition:** using genuine production data directly in lower environments creates real privacy/compliance risk (the same PII/regulated-data concerns as the AWS/Java security notes and the FDE notes' section 7); the standard mitigations are **data masking** (replacing sensitive fields with realistic-but-fake values while preserving data shape/relationships), **synthetic data generation** (fabricated data resembling production's statistical characteristics without ever containing real user information), or simply **seeding** a known, small, safe dataset specifically designed to exercise the application's features.

---

## 3. Version Control & Branching Strategies

**Definition:** a branching strategy is a team's agreed convention for how code changes flow through version control (branches, merges) on their way toward release — the strategy chosen directly shapes how continuous (or batched) a team's deployment process can realistically be.

**Trunk-based development — Definition:** developers commit small, frequent changes directly to a single shared branch (`main`/`trunk`), avoiding long-lived feature branches entirely — pairs naturally with continuous integration/deployment (sections 5–6), since there's no lengthy, divergent branch to eventually reconcile; incomplete features are hidden behind feature flags (section 10) rather than isolated on a separate branch, keeping `main` always in a deployable state.

**Git Flow — Definition:** a more structured model with dedicated long-lived `develop` and `main` branches, plus temporary `feature/`, `release/`, and `hotfix/` branches following defined rules for when each merges where — provides clear structure for teams with scheduled, batched releases (section 16's "release trains"), but its longer-lived branches and more elaborate merge choreography sit in real tension with a fast, continuous deployment cadence.

**GitHub Flow — Definition:** a lighter-weight model between the two extremes — short-lived feature branches, each opened as a pull request and merged directly into `main` once reviewed/approved, with `main` always deployable — the common default for teams practicing continuous delivery (section 6) without going as far as pure trunk-based development's direct-to-main commits.

**Feature branches & pull requests — Definition:** isolating a change on its own branch until it's reviewed and merged — the PR itself serves as both a code-review checkpoint and, typically, the trigger point for CI (section 5) to validate the change before it's allowed to merge.

**Tagging releases — Definition:** attaching an immutable, named marker (a Git tag, e.g. `v2.4.1`) to a specific commit representing exactly what was deployed as a given release — gives a permanent, unambiguous reference point for "what code was actually running in production at this point in time," essential for debugging production incidents after the fact and for rollback (section 14).

**Semantic versioning (SemVer) — Definition:** a versioning convention, `MAJOR.MINOR.PATCH` (e.g. `2.4.1`) — incrementing **MAJOR** signals a breaking change, **MINOR** signals a backward-compatible new feature, **PATCH** signals a backward-compatible bug fix — lets consumers of a versioned artifact (a library, an API, an internal service other teams depend on) reason about the risk of upgrading to a new version purely from its version number, without reading a full changelog.

**Branch protection & required checks — Definition:** repository-level rules preventing direct pushes to protected branches (`main`) and requiring specific conditions — passing CI, a minimum number of approving reviews — before a pull request can be merged, enforcing a team's quality bar mechanically rather than relying purely on developer discipline.

**Choosing a branching strategy** — trunk-based/GitHub Flow for teams practicing genuine continuous delivery/deployment with a strong automated test suite; Git Flow (or a variant) for teams with scheduled release cycles, multiple parallel supported versions, or a lower-trust automated testing setup that genuinely benefits from a longer-lived, more heavily-vetted release branch — the right choice follows from the team's actual deployment cadence and risk tolerance, not an assumption that one model is universally correct.

---

## 4. Build Process & Artifacts

**Definition:** the build process transforms source code into a deployable artifact — compiling, bundling, and packaging it into the specific form the target environment actually runs — a distinct, essential step between "code that exists" and "code that can be deployed" (section 1).

**Reproducible builds — Definition:** a build process that produces byte-identical (or functionally identical) output given the same source input and the same build environment/dependency versions, every time — essential for trust in the deployment process: without reproducibility, "it worked when we built it before" provides no real guarantee about a rebuild today, and diagnosing a production issue becomes far harder if you can't be certain the artifact you're inspecting locally matches what's actually running.

**Build caching — Definition:** reusing the results of previous, unchanged build steps rather than redoing them from scratch on every build — the same principle as Docker's layer caching (Docker/Kubernetes notes' section 2) or a compiler's incremental-build support — a major lever for keeping CI pipeline (section 5) feedback loops fast as a codebase grows.

**Artifact types — Definition:** the specific packaged form varies by stack — a compiled **JAR** (Java, Java Backend notes); a **Docker image** (any containerized stack, Docker/Kubernetes notes); a **bundled static JS application** (Angular/React notes' production builds — hashed JS/CSS files ready to serve from a CDN); a Python **wheel** or a fully-built virtual environment (Python Backend notes) — the deployment process itself is largely artifact-type-agnostic in its overall shape, but the concrete mechanics of building and deploying each differ.

**Artifact versioning & immutability — Definition:** every built artifact should be tagged with a unique, immutable identifier (a semantic version, or — increasingly the more robust default — the exact git commit SHA it was built from) and, once published, **never modified in place** — deploying `myapp:1.2.0` should always deploy the exact same bits, every time, at every environment; overwriting an existing tag's contents (or relying on a floating tag like `latest`, already flagged as an anti-pattern in the Docker/Kubernetes notes' section 2) breaks reproducibility and makes rollback (section 14) unreliable.

**Artifact repositories/registries — Definition:** a dedicated store for built artifacts, separate from source control — a container registry (Docker Hub, Amazon ECR, GitHub Container Registry — AWS notes' section 9) for images, a package registry (npm, PyPI, Maven Central, or a private equivalent) for libraries/packages — the build pipeline publishes here, and the deployment step pulls from here, keeping "what got built" and "what's deployed" cleanly traceable back to a specific, retrievable artifact.

---

## 5. Continuous Integration (CI)

**Definition:** Continuous Integration is the practice of automatically building and testing every code change (typically on every commit or pull request) against the shared codebase — catching integration problems and regressions immediately, rather than discovering them much later when multiple developers' separately-developed changes are finally combined.

**What CI actually guarantees — Definition:** CI's core promise is narrow but important: that a given change **compiles/builds successfully and passes the existing automated test suite** — it does *not* guarantee the change is bug-free, well-designed, or actually solves the intended problem (those require code review and the test suite's own quality, respectively) — understanding this scope precisely avoids over-trusting a green CI run as a guarantee of correctness it was never designed to provide.

**The CI pipeline — Definition:** a typical CI pipeline runs, in order: **lint** (static code-quality/style checks — fails fast, cheaply, before spending time on slower steps), **test** (the automated test suite — unit, then typically integration tests, per the testing sections of this workspace's language-specific notes), **build** (produce the actual artifact, section 4) — structured so cheaper, faster checks run first and can fail the pipeline quickly, without wasting time on slower steps for a change that was already going to fail an early, simple check.

```yaml
# example CI pipeline (GitHub Actions-style)
jobs:
  ci:
    steps:
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

**Fast feedback loops — Definition:** the time between pushing a change and knowing whether it passed CI directly determines developer productivity and how willing a team is to make small, frequent commits (trunk-based development, section 3) rather than large, infrequent, riskier ones — a CI pipeline that takes 30+ minutes measurably discourages the small-commit discipline that makes continuous delivery actually work well; keeping pipelines fast (via build caching, parallelized test execution, running only affected tests where feasible) is itself an ongoing engineering investment, not a one-time setup task.

**Flaky tests & how they erode CI trust — Definition:** a **flaky test** passes and fails intermittently for the same, unchanged code — usually due to timing assumptions, shared/uncleaned test state, or genuine non-determinism in what's being tested — flaky tests are corrosive specifically because they train developers to reflexively re-run a failed CI job "to see if it passes this time" rather than actually investigating failures, which eventually causes a **genuine** regression to also be waved through as "probably just flaky" — treating flaky tests as a priority bug to fix (or, if truly unfixable in the short term, quarantining them out of the required-check gate explicitly) protects the entire team's trust in CI as a meaningful signal.

**CI as a gate before merge (recap)** — see section 3's branch protection; requiring CI to pass before a pull request can merge is what turns CI from "a thing that runs and reports status" into an actually-enforced quality bar, rather than a check that's easy to ignore.

---

## 6. Continuous Delivery vs Continuous Deployment (CD)

**Definition:** both extend CI (section 5) further down the pipeline toward actually getting a change into production, but differ in exactly one, consequential way: whether a human explicitly approves each production deployment, or whether it happens fully automatically.

**Continuous Delivery — Definition:** every change that passes the full pipeline (CI, plus any further automated testing/staging validation) is **automatically confirmed to be deployable at any time**, but the actual production deployment is triggered by an explicit, deliberate human action (a button click, an approval step) — the team maintains the discipline and automation to deploy at any moment, without necessarily doing so on every single change.

**Continuous Deployment — Definition:** goes one step further — **every** change that passes the full automated pipeline is deployed to production **automatically**, with no human approval gate at all — requires substantially higher confidence in the automated test suite's coverage and the deployment process's safety nets (fast rollback, section 14; deployment strategies that limit blast radius, section 10), since there is no human checkpoint left to catch a problem before it reaches real users.

**Choosing between them** — Continuous Deployment demands a genuinely mature, comprehensive automated test suite and robust rollback/canary safety nets before it's actually a *safe* choice, not just a technically-possible one; Continuous Delivery is the more common, still highly effective default for teams wanting fast, low-friction releases while retaining an explicit human decision point for exactly when a given change actually goes live — neither is "more advanced" than the other in an unconditional sense; the right choice depends on the team's actual test-suite maturity and the acceptable risk profile for this specific system.

**Pipeline stages — Definition:** a typical CD pipeline extends the CI pipeline (section 5) with further stages: **build** → **test** → **deploy to staging** (section 2) → **further validation** (automated end-to-end tests against staging, possibly manual QA) → **deploy to production** (automatically, for Continuous Deployment, or gated behind approval, for Continuous Delivery).

**Approval gates — Definition:** an explicit checkpoint in a CD pipeline requiring a specific person or team to approve before the pipeline proceeds to the next (typically production) stage — the concrete mechanism implementing Continuous Delivery's human-decision-point distinction from Continuous Deployment, and also commonly used for other genuinely high-stakes steps (a database migration, a change affecting a regulated system) even within an otherwise largely-automated pipeline.

---

## 7. Configuration & Secrets Management

**Definition:** configuration is any value that varies between environments or deployments without requiring a code change — externalizing it fully from application code (rather than hardcoding environment-specific values) is what allows the *same* build artifact (section 4) to be deployed unmodified across dev, staging, and production.

**Externalized configuration (recap, 12-factor)** — see this workspace's Angular/React/Node.js/Java/Python production-engineering sections, all converging on the same "12-factor app" principle: configuration (database URLs, API keys, feature flags) lives outside the codebase, injected at runtime/deploy-time via environment variables or a dedicated config service — never as environment-specific code branches baked directly into the application.

**Environment variables vs config files vs config services — Definition:** **environment variables** are the simplest, most universally-supported mechanism (every language/platform reads them), well suited to simple key-value configuration; **config files** (a `.env` file, a YAML config) group related settings together more readably, at the cost of needing to be securely delivered to each environment rather than being a platform-native mechanism; a **config service** (e.g. AWS Parameter Store, discussed in the AWS notes' section 14) centralizes configuration across many services/environments, supports versioning and audit history, and can push live config changes without a redeploy — the right choice scales with how much configuration a system has and how many services need to share it consistently.

**Secrets managers (recap)** — see the AWS notes' section 14 and the Java/Node.js/Python notes' respective security sections; a dedicated secrets store (AWS Secrets Manager, HashiCorp Vault) rather than plain environment variables for anything genuinely sensitive (database passwords, API keys, signing secrets) — providing access-controlled retrieval, audit logging, and (critically) automatic **rotation**.

**Secret rotation — Definition:** periodically replacing a secret's value (a database password, an API key) on a schedule, without requiring a full application redeploy to pick up the new value — limits how long a leaked or compromised secret remains valid/useful to an attacker, and is a capability meaningfully easier to get right with a dedicated secrets manager than with secrets baked into environment variables that require a redeploy to change.

**Avoiding secrets in source control & build logs — Definition:** a secret committed to git history remains recoverable indefinitely, even if later "removed" in a subsequent commit (unless the entire repository history is rewritten and force-pushed — a genuinely disruptive, last-resort remediation); equally, secrets can leak through **build/CI logs** if a pipeline step accidentally prints an environment variable's value — both are common, entirely avoidable real-world security incidents, prevented by never hardcoding secrets in code (always injected via the mechanisms above) and configuring CI systems to automatically mask known secret values in logged output.

---

## 8. Containerization & Image Management

**Definition:** packaging an application and its runtime dependencies into a container image is, for most modern stacks, the standard way to produce a consistent, portable deployment artifact (section 4) — see the Docker/Kubernetes notes for the full underlying mechanics; this section focuses specifically on the deployment-process concerns around image management.

**Building deployable images (recap)** — see the Docker/Kubernetes notes' section 2 in full; multi-stage builds producing a slim final image, with dependencies and build tooling excluded from what actually ships.

**Image tagging strategy for deployments — Definition:** the same immutable-artifact principle from section 4, applied specifically to container images — tag every built image with something traceable and unique (the git commit SHA, or a semantic version, section 3) rather than relying on `latest`, so any deployed image can be traced back unambiguously to the exact source code it was built from, and so redeploying a previous version (rollback, section 14) is simply "deploy the image tagged with that earlier commit/version" rather than an ambiguous or impossible operation.

**Container registries (recap)** — see the AWS notes' section 9 (ECR) and Docker/Kubernetes notes' section 2 (Docker Hub, GHCR); the artifact repository (section 4) specifically for container images, which the deployment pipeline pushes newly-built images to and the target environment (a Kubernetes cluster, an ECS service) pulls from.

**Image scanning in the pipeline — Definition:** automatically scanning a built image for known vulnerabilities (in its base OS packages and installed dependencies) as a pipeline step, before it's allowed to be deployed — the container-image-specific instance of the broader dependency-scanning practice covered in section 17, catching a vulnerable base image or dependency before it reaches production rather than discovering it later via a separate security audit.

**Multi-arch builds (brief) — Definition:** building a container image that supports multiple CPU architectures (e.g. both x86-64 and ARM64) from one build process, so the same image tag works correctly regardless of the underlying host architecture — increasingly relevant as ARM-based infrastructure (AWS Graviton, Apple Silicon-based CI runners) has become more common alongside traditional x86 infrastructure.

---

## 9. Infrastructure as Code

**Definition:** Infrastructure as Code (IaC) means defining and provisioning infrastructure through versioned, declarative configuration files rather than manual console clicks or ad hoc scripts — see the AWS notes' section 10 for the full treatment (CloudFormation, CDK, Terraform); this section focuses on how IaC fits into the deployment process specifically.

**Provisioning vs configuration vs deployment (distinguishing the three) — Definition:** these are three related but genuinely distinct concerns, often conflated. **Provisioning** creates the underlying infrastructure itself (a server, a database instance, a Kubernetes cluster) — typically IaC's job. **Configuration** sets up that infrastructure's runtime settings (installing packages, setting OS-level config) — sometimes IaC's job, sometimes a separate configuration-management tool's (Ansible, Chef). **Deployment** (this file's overall subject) puts a specific application version onto already-provisioned, already-configured infrastructure — a deployment pipeline typically assumes provisioning/configuration already happened (infrequently) and focuses on the much-more-frequent act of shipping new application code onto infrastructure that already exists.

**State management in IaC — Definition:** IaC tools track the current, real state of provisioned infrastructure (Terraform's state file being the canonical example) so they can compute the difference between "what's declared in code" and "what actually exists" and apply only the necessary changes — this state itself needs careful management (stored securely, with locking to prevent two people applying conflicting changes simultaneously), since state corruption or drift (the AWS notes' section 10 covers drift detection) can cause IaC tooling to make incorrect, potentially destructive changes.

**Immutable infrastructure — Definition:** rather than modifying existing running servers in place (installing an update, patching a config file on a live machine), immutable infrastructure **replaces** them entirely with newly-provisioned instances built from an updated image/template — eliminates an entire class of "it works on this server but not that one because of some undocumented manual change made months ago" configuration-drift problems, and pairs naturally with container-based deployment (section 8) and the deployment strategies in section 10, all of which are built around replacing rather than mutating running instances.

**GitOps — Definition:** a deployment model where a Git repository is the single source of truth for a system's *desired* infrastructure/application state, and an automated controller continuously reconciles the live environment to match whatever's declared in that repository — deployments happen by merging a Git change, not by a human or CI job directly issuing `kubectl apply`/cloud-provider commands against the live environment — see the Docker/Kubernetes notes' section 19 (ArgoCD, Flux) for the concrete Kubernetes-specific implementation of this pattern, which is increasingly the standard deployment model for container-orchestrated systems specifically.

---

## 10. Deployment Strategies

**Definition:** a deployment strategy is the specific technique used to shift live traffic from an old version of a running application to a new one — chosen based on how much risk (of the new version having a problem) the team is willing to expose real users to, and how quickly that risk needs to be caught and reversed if something goes wrong.

**Recreate — Definition:** the simplest strategy — stop the old version entirely, then start the new version — causes a real, if brief, period of downtime while the old version is down and the new one is starting up; acceptable only for systems where brief downtime during a deploy is genuinely tolerable (internal tools, low-traffic systems, or systems with a defined maintenance window).

**Rolling deployment — Definition:** instances of the old version are replaced with the new version gradually, in batches, rather than all at once — at any given moment during the rollout, some instances run the old version and some run the new one, serving traffic simultaneously — avoids Recreate's downtime, at the cost of requiring the application to genuinely tolerate two versions running concurrently for the rollout's duration (a real constraint on database schema changes, section 11) — the default deployment behavior for a Kubernetes `Deployment` object (Docker/Kubernetes notes' section 9).

**Blue/green deployment — Definition:** two complete, independent environments exist — "blue" (currently live) and "green" (the new version, deployed fully but not yet receiving live traffic) — once the green environment is verified healthy, traffic is switched over to it all at once (typically via a load balancer or DNS change), with blue kept running briefly afterward as an instant rollback target (section 14) — provides very fast, clean rollback (just switch traffic back) at the cost of running two full production-sized environments simultaneously during the switch.

```
Blue (live, v1) ← all traffic
Green (new, v2) ← deployed, health-checked, NOT yet receiving traffic

  ↓ traffic switch (instant)

Blue (v1) ← kept briefly as rollback target
Green (v2) ← all traffic
```

**Canary deployment — Definition:** the new version is deployed and given only a small percentage of real traffic initially (e.g. 5%), while the old version continues serving the rest — that small slice is monitored closely for errors/regressions, and traffic is gradually shifted further to the new version only if it proves healthy, eventually reaching 100% — limits the **blast radius** of a bad deployment to a small fraction of real users, catching problems with genuine production traffic/data before they affect everyone, at the cost of needing solid metrics/monitoring (section 15) to actually judge canary health and more complex traffic-splitting infrastructure than a simple all-or-nothing switch.

**Shadow / dark launch deployment — Definition:** the new version receives a **copy** of real production traffic (mirrored, not redirected) and processes it fully, but its responses are never actually returned to real users — the old version continues to be what users actually see — lets a team validate a new version's behavior against genuine production traffic patterns with **zero** user-facing risk at all, at the cost of real complexity in safely mirroring traffic (especially for anything with side effects — writes, external API calls — which typically need to be carefully stubbed/isolated in the shadow path to avoid duplicate real-world effects).

**Feature flags as a deployment decoupling mechanism (recap)** — see section 1's deploy-vs-release distinction; a feature flag lets new code be deployed to production (fully built and running) while remaining inactive/hidden until explicitly toggled on — decouples the deployment strategies above (which control *how* new code physically reaches production) from the separate decision of *when* a feature is actually exposed to users, and enables fine-grained release control (enable for internal users only, then a percentage of real users, entirely independent of the underlying deployment mechanism).

**Choosing a strategy** — Recreate for low-stakes systems where brief downtime is fine; Rolling as the solid default for most containerized/orchestrated systems; Blue/green when instant, clean rollback matters most and running duplicate infrastructure briefly is affordable; Canary when validating against real production traffic/data before full exposure matters most and the system has the monitoring maturity to judge canary health reliably; Shadow launches for the highest-risk changes where even a small fraction of real user-facing risk (canary's residual exposure) is unacceptable.

---

## 11. Database Migrations During Deployment

**Definition:** deploying a change to an application's code and deploying a change to its database schema are two distinct operations that must be carefully sequenced together, since — unlike application code, which a deployment strategy can cleanly version and roll back — a database's schema is a single, shared, stateful resource that both old and new application code versions may need to interact with simultaneously during a rolling/canary rollout (section 10).

**Schema migrations & deployment ordering — Definition:** the order in which a schema migration and the corresponding application code deployment happen matters critically — deploying application code that expects a new column *before* that column actually exists in the database will fail; deploying a schema migration that *removes* a column the still-running old application code depends on will also fail — the section below (expand/contract) exists specifically to make this ordering safe under a rolling deployment, where old and new code run concurrently for a period.

**Backward-compatible migrations (expand/contract pattern) — Definition:** the standard technique for making a schema change safe during a rolling deployment (section 10), where old and new application code run simultaneously for a period — split a single conceptual schema change into multiple, independently-safe deployment steps:
1. **Expand** — add the new column/table (additive only; old code, unaware of it, continues working unaffected).
2. Deploy new application code that can read/write **both** the old and new schema shape.
3. Backfill/migrate existing data into the new shape.
4. Deploy application code that uses **only** the new shape.
5. **Contract** — remove the old, now-unused column/table, only once no running code depends on it anymore.

This spreads a schema change safely across several independent deployments rather than attempting one risky, all-at-once "change the schema and the code together" step that a rolling deployment's concurrent old/new code would make unsafe.

**Zero-downtime schema changes (recap)** — the expand/contract pattern above is precisely how zero-downtime deployment (section 12) is achieved for schema changes specifically — the same underlying principle (never make old and new simultaneously-running code incompatible with the current schema state) generalizes across every database technology covered in this workspace's SQL and MongoDB notes.

**Migration tooling (recap)** — see this workspace's stack-specific notes: Flyway/Liquibase (Java Backend notes' section 11), Alembic (Python Backend notes' section 14), Django's built-in migrations (Python Backend notes' section 13), and the SQL notes' general migration-strategy section — each automates applying versioned migration scripts in order and tracking which have already been applied to a given database, but the expand/contract *sequencing discipline* above is a deployment-process concern layered on top of whichever specific tool is used, not something the tool alone guarantees.

**Data migrations vs schema migrations — Definition:** a **schema migration** changes the database's structure (adding a column, an index); a **data migration** changes/moves the actual data within that structure (backfilling a new column's values for existing rows, transforming data into a new format) — data migrations on a large table can take significant time and lock/load the database meaningfully, so they're often run as a separate, carefully-throttled, resumable background process rather than a blocking step within the deployment pipeline itself.

---

## 12. Zero-Downtime Deployment

**Definition:** a deployment process that completes without any period during which the system is unable to serve real user requests — achieved through the combination of a non-Recreate deployment strategy (section 10), graceful connection handling, and (when relevant) backward-compatible schema changes (section 11) — genuinely requires every layer of the system to cooperate, not just the deployment strategy alone.

**What zero-downtime actually requires end-to-end** — a rolling/blue-green/canary deployment strategy (section 10) alone is necessary but not sufficient; it must be paired with: graceful shutdown of outgoing instances (below), health-check-gated traffic routing (below) so new instances only receive traffic once actually ready, and (if applicable) schema changes that remain compatible with both old and new code running concurrently (section 11) — missing any one of these can still produce user-facing errors during a deploy, even with an otherwise-correct deployment strategy in place.

**Load balancer draining & connection handling — Definition:** when an instance is being taken out of rotation (as part of a rolling deployment, section 10), the load balancer should stop routing **new** requests to it while allowing any **already in-flight** requests to that instance to complete normally before it's actually terminated — abruptly killing an instance with active connections still being served causes exactly the kind of dropped-request errors zero-downtime deployment is meant to eliminate.

**Graceful shutdown (recap)** — see the Node.js/AWS/Docker-Kubernetes notes' respective sections; on receiving a termination signal (`SIGTERM`), an application should stop accepting new connections but finish any in-flight requests before actually exiting — the application-level half of the load-balancer-draining coordination above; both sides (load balancer routing and application shutdown behavior) need to cooperate correctly for connection draining to actually work end-to-end.

**Health checks & readiness gates (recap)** — see the AWS/Docker-Kubernetes notes' respective sections (liveness vs readiness probes); a newly-deployed instance should only begin receiving real traffic once its **readiness** check passes (confirming it's actually fully initialized and able to serve requests correctly) — routing traffic to an instance that's still starting up, before it's actually ready, is a common, avoidable source of errors during an otherwise well-executed rolling deployment.

**In-flight request handling during a deploy** — the combination of the three points above: a request that's already being processed when a deployment begins should complete successfully against whichever instance/version it started on, never abruptly dropped mid-flight purely because a deployment happened to be in progress at that moment — the concrete, user-facing guarantee "zero-downtime deployment" is actually promising.

---

## 13. Cloud & Platform-Specific Deployment Patterns

**Deploying to VMs/EC2 (recap)** — see the AWS notes' section 3; deployment onto virtual machines typically involves either an immutable-infrastructure approach (build a new AMI/image with the new code baked in, section 9, and replace instances via an Auto Scaling Group rolling update) or an in-place approach (a deployment script/agent pushes new code onto already-running instances) — the immutable approach is generally preferred today for the same configuration-drift-avoidance reasons covered in section 9.

**Deploying to Kubernetes (recap + deployment-specific detail)** — see the Docker/Kubernetes notes' sections 9 and 19 in full; a `Deployment` object's rolling update (section 10 of this file) is configured via `maxSurge`/`maxUnavailable`, controlling exactly how aggressively old pods are replaced with new ones during a rollout — `kubectl rollout status`/`kubectl rollout undo` provide direct visibility into and control over an in-progress or completed rollout, and are the concrete mechanism behind this file's rollback discussion (section 14) in a Kubernetes context specifically.

**Deploying serverless (Lambda, Cloud Functions) — Definition:** deploying a new function version is typically near-instantaneous (no servers to roll out across, section 8's containerization concerns and section 12's connection-draining concerns don't apply the same way) — but a genuinely safe serverless deployment still uses gradual traffic shifting between function **versions/aliases** (AWS Lambda's alias-based traffic-shifting being the direct analog of canary deployment, section 10, for a serverless target) rather than an instant, all-at-once cutover for anything beyond a low-stakes function.

**Deploying static sites/SPAs (CDN-based) — Definition:** for a purely static frontend build (Angular/React notes' production builds), "deployment" means uploading the new, hashed-filename build artifacts to object storage (AWS S3) and invalidating/updating the CDN (CloudFront) serving them (AWS notes' sections 4 and 11) — the hashed-filename convention (already covered in this workspace's frontend production-engineering sections) means old and new versions' asset files can coexist in storage simultaneously without collision, making this one of the simplest deployment targets to make safely zero-downtime.

**Platform-as-a-Service deployment (brief) — Definition:** platforms like Heroku (or similar "push your code, we handle the infrastructure" services) abstract away most of sections 8–10's underlying mechanics behind a simple `git push`-triggered deploy — trading direct control over the deployment process for significantly reduced operational overhead, a reasonable tradeoff for smaller teams/projects where that operational simplicity outweighs the value of fine-grained control.

**Multi-region deployment (recap)** — see the AWS notes' section 17 and System Design notes' section 12 for the full disaster-recovery/availability rationale; deploying the same application version consistently across multiple geographic regions adds real coordination complexity to every concept in this file (a rollout now needs a defined per-region sequencing strategy, not just a per-instance one) in exchange for lower latency to geographically distributed users and resilience against a single region's outage.

---

## 14. Rollback & Disaster Recovery

**Definition:** rollback is the process of reverting a deployment that's causing problems back to a previously-known-good version — a deployment process's rollback capability should be designed and tested **before** it's ever actually needed under real incident pressure, not improvised in the moment.

**Rollback strategies per deployment strategy — Definition:** each deployment strategy (section 10) has a correspondingly different rollback mechanism: **Recreate/Rolling** — redeploy the previous artifact version (the same deployment mechanism run in reverse, immediate but not instant); **Blue/green** — switch traffic back to the still-running "blue" environment (fastest possible rollback, since the old version never actually stopped running); **Canary** — simply stop shifting traffic further and route the small canary slice back to the stable version (cheapest rollback of all, since only a small fraction of traffic was ever exposed in the first place).

**Fast rollback as a design requirement, not an afterthought — Definition:** the ability to roll back quickly and reliably should be a first-class design goal of the deployment process itself, established *before* a real incident, not a capability discovered to be missing (or too slow, or too risky) for the first time during one — section 4's insistence on immutable, uniquely-tagged artifacts exists specifically so "roll back to the previous version" is always a well-defined, mechanically simple operation ("redeploy artifact X"), rather than an ambiguous, error-prone manual reconstruction under pressure.

**Database rollback complications — Definition:** rolling back *application code* is comparatively straightforward (redeploy the old artifact); rolling back a *database schema/data* change is fundamentally harder, since data written under the new schema may not have a clean, lossless representation under the old one — this is precisely why the expand/contract migration pattern (section 11) matters so much: by keeping the old and new schema shapes compatible for a transition period, it also keeps a *rollback* of the application code safe during that same window, without requiring an equally risky reverse data migration to match.

**Runbooks for failed deployments — Definition:** a written, specific, step-by-step procedure for what to do when a deployment is identified as problematic — who has authority to trigger a rollback, exactly which commands/pipeline actions to run, how to verify the rollback actually succeeded, who needs to be notified — prepared in advance so a real incident is executed against a calm, pre-thought-through plan rather than improvised under the time pressure and stress a live production issue creates.

**Disaster recovery (recap)** — see the AWS notes' section 17 and System Design notes' section 12; RTO (Recovery Time Objective — how fast service must be restored) and RPO (Recovery Point Objective — how much data loss is acceptable) as the two numbers that determine which disaster-recovery strategy is actually appropriate for a given system — rollback (this section) addresses recovering from a *bad deployment* specifically; broader disaster recovery addresses a wider range of failure scenarios (an entire region outage, catastrophic data loss) that rollback alone doesn't cover.

---

## 15. Post-Deploy Verification, Monitoring & Alerting

**Definition:** deploying new code and *knowing it's actually working correctly* are two different things — a deployment process isn't complete until it includes some form of verification that the newly-deployed version is genuinely healthy, not merely "running without an immediately-crashing error."

**Smoke tests after deploy — Definition:** a small, fast set of tests run immediately after a deployment completes, checking that the most critical functionality genuinely works against the real, just-deployed environment (not a broad regression suite — that already ran during CI, section 5 — but a final, narrow "did the deployment itself actually work" sanity check) — commonly automated as an explicit pipeline stage that can trigger an automatic rollback (section 14) if it fails.

**Automated post-deploy verification — Definition:** beyond simple smoke tests, some pipelines automatically compare key metrics (error rate, latency) between the newly-deployed version and its immediate predecessor for a defined observation window immediately after deployment, automatically halting a rollout or triggering rollback if the new version's metrics are meaningfully worse — the automated, codified version of what a canary deployment's (section 10) manual health-monitoring step is doing, removing the need for a human to be actively watching a dashboard at the exact moment of every single deployment.

**Monitoring what actually matters post-deploy (recap)** — see the System Design notes' section 14 and AWS/Docker-Kubernetes notes' respective observability sections; error rate, latency percentiles, and (crucially) actual business/task-success metrics — not just "is the process still running," which can be true even while the application is silently failing to do its actual job correctly.

**Alerting thresholds & on-call (recap)** — see the System Design notes' section 14's alerting-noise discussion; alerts tied to deployments specifically should distinguish a genuine regression introduced by *this* deployment from normal, expected variance — an alerting system too sensitive around every deployment trains the on-call team to reflexively dismiss deployment-triggered alerts, defeating their purpose exactly the way flaky tests erode trust in CI (section 5).

**Deployment markers in observability tools — Definition:** annotating monitoring dashboards/graphs with the exact timestamp of each deployment — makes it immediately visually obvious, when investigating a metric change, whether it correlates with a specific deployment (strongly suggesting that deployment as the cause) or occurred independently of any deployment (pointing investigation elsewhere) — a small, cheap addition to an observability setup with outsized value for fast incident diagnosis.

---

## 16. Release Management & Change Management

**Definition:** release management is the broader discipline of planning, coordinating, and communicating *what* gets released *when* — distinct from (though built on top of) the purely technical deployment mechanics covered in the rest of this file.

**Release notes & changelogs — Definition:** a human-readable summary of what changed in a given release — who it's written for (end users, internal stakeholders, other engineering teams depending on a service) shapes what level of detail and framing is appropriate; automatically generating a changelog from commit messages/PR titles (feasible when a team follows a consistent commit-message convention) reduces the manual overhead of keeping this documentation current.

**Coordinating releases across teams/services — Definition:** in a multi-service (microservices, System Design notes' section 9) or multi-team system, one team's release can depend on, or be depended on by, another's — requiring explicit coordination (a shared release calendar, defined API versioning/backward-compatibility contracts, section 3's semantic versioning) so that services can actually be deployed independently without breaking each other — the deployment-process analog of the same inter-service coupling concerns covered in the System Design and Java/Node.js microservices sections.

**Release trains vs on-demand releases — Definition:** a **release train** bundles a batch of changes together on a fixed schedule (e.g. every two weeks, everything that's ready ships together, anything not ready waits for the next train) — provides predictability and a natural batching of change-management overhead, at the cost of coupling an individual change's release timing to the schedule rather than shipping the moment it's ready; **on-demand releases** (the natural fit for continuous delivery/deployment, section 6) ship each change independently, as soon as it's ready — the right choice tracks closely with section 6's CD-model choice and section 3's branching-strategy choice, since all three are really facets of the same underlying "how continuous is our delivery cadence" decision.

**Change approval processes (when they help, when they're overhead) — Definition:** a formal review/sign-off step required before a change can be deployed — genuinely valuable for changes carrying real regulatory/compliance requirements or unusually high blast-radius risk; can become pure friction/overhead — without commensurate risk reduction — when applied uniformly to every change regardless of actual risk, which is precisely the tension the "when do they help vs when are they overhead" framing is meant to surface: match the process's weight to the change's actual risk, not a one-size-fits-all policy.

**Communicating deployments to stakeholders — Definition:** proactively informing relevant stakeholders (support teams who'll field user questions, other engineering teams whose systems interact with the one being changed, sometimes end users directly for user-visible changes) about upcoming and completed deployments — reduces the "why did this suddenly change and nobody told us" friction that erodes trust between teams and between an organization and its users, especially for changes with any real visible impact.

---

## 17. Security in the Deployment Pipeline

**Definition:** the deployment pipeline itself — the CI/CD system, its credentials, and everything it touches — is a genuine, high-value attack surface, since it typically has broad, automated write access to production infrastructure; securing it deserves the same deliberate rigor as securing the application it deploys.

**Pipeline as an attack surface — Definition:** a CI/CD system that can deploy to production is, by necessity, holding credentials capable of modifying production — which makes compromising the pipeline itself (rather than the application directly) an attractive target: a successful attack on CI/CD tooling, or on a dependency it pulls in during a build, can result in **malicious code being deployed through entirely legitimate, trusted channels** — this is the deployment-process-specific instance of supply-chain security, and real-world supply-chain attacks have specifically targeted build/CI systems for exactly this reason.

**Least-privilege CI/CD credentials (recap)** — see the AWS/Java notes' IAM/security sections; the credentials a CI/CD pipeline uses to deploy should be scoped to exactly the permissions that specific pipeline actually needs (deploy to this one service, not blanket admin access to the entire cloud account) — the same least-privilege principle applied throughout this workspace's security-focused sections, here protecting against the consequences of the pipeline itself being compromised.

**Signed artifacts & supply chain security (recap)** — see the Java notes' Docker section and Docker/Kubernetes notes' section 17; cryptographically signing build artifacts (container images especially) so a deployment target can verify an artifact genuinely came from your trusted build pipeline and hasn't been tampered with in transit or substituted for a malicious one — an increasingly standard practice (via tools like Sigstore/Cosign) for anything beyond a low-stakes internal system.

**Dependency scanning in the pipeline (recap)** — see section 8's image-scanning discussion and the Node.js/Python/Java notes' respective security sections (`npm audit`, similar tooling); automatically scanning both application dependencies and (section 8) container base images for known vulnerabilities as a required pipeline step, failing the build/blocking deployment if a sufficiently severe vulnerability is found, rather than relying on a separate, periodic, easily-forgotten manual security audit.

**Audit logging of deployments — Definition:** recording who deployed what, when, and (ideally) why — for accountability, for incident investigation ("was this issue introduced by a specific deployment, and by whom"), and frequently for genuine compliance/regulatory requirements — modern CI/CD platforms and cloud providers typically provide this natively (CloudTrail, AWS notes' section 13, being the direct AWS-specific example), but it's worth explicitly verifying it's actually enabled and retained for long enough to be useful when actually needed, rather than assuming it exists by default.

---

## 18. Deployment Patterns by Architecture

**Monolith deployment — Definition:** a single deployable unit containing the entire application — deployment is comparatively simple (one artifact, one deployment pipeline, section 10's strategies apply directly and cleanly) but an entire application must be redeployed for even a small, isolated change, and a failure in one part of the deployment affects the whole system at once — the tradeoff already covered conceptually in the Node.js and System Design notes' architecture sections, viewed here specifically through the deployment-process lens.

**Microservices deployment (independent deployability) — Definition:** the defining deployment-process benefit of a microservices architecture (System Design notes' section 9) is that each service can be deployed **independently**, on its own schedule, without requiring the entire system to redeploy together — but this independence only actually holds if services maintain backward compatibility with each other across independent deployments (section 16's coordination discussion), since an assumption that "everything always deploys together" quietly creeping back in defeats the whole point of having split into microservices in the first place.

**Static frontend/SPA deployment (recap)** — see section 13; typically the simplest deployment target in this whole roadmap (upload hashed static files, update CDN), since there's no running server process to roll out across, no connection-draining concern, and old/new asset versions can coexist trivially in storage without conflict.

**Serverless deployment (recap)** — see section 13; near-instant artifact deployment, but genuine production safety still benefits from gradual, alias-based traffic shifting (the serverless-specific analog of canary deployment, section 10) for anything beyond the lowest-stakes function.

**Mobile app deployment (brief) — Definition:** deploying a native mobile application differs fundamentally from every other pattern in this file — releases go through an **app store review process** (Apple App Store, Google Play) that's entirely outside the development team's direct control, introducing real, sometimes multi-day latency between "build is ready" and "users can actually receive it," and, unlike a web deployment, the *previous* version keeps running on users' devices indefinitely until they individually choose to update — meaning multiple app versions can be genuinely live in the real world simultaneously for a long time, which server-side APIs the app depends on must be designed to tolerate (maintaining backward compatibility across several previous app versions, not just the one currently "live") far more durably than a typical web backend needs to.

---

## 19. Interview Preparation & Common Pitfalls

**Common deployment-process interview questions** — explain the difference between continuous delivery and continuous deployment, and how you'd decide which is appropriate for a given team (section 6); walk through how you'd safely deploy a backward-incompatible database schema change without downtime (section 11's expand/contract pattern); compare blue/green and canary deployment strategies and when you'd choose each (section 10); what makes a deployment pipeline itself a security risk, and how do you mitigate it (section 17); how would you design a rollback strategy for a system before it's ever actually needed (section 14).

**Designing a deployment pipeline (worked approach)** — clarify the system's actual deployment cadence needs and risk tolerance first (a high-traffic consumer product's requirements differ enormously from an internal admin tool's, section 10's strategy choice follows directly from this); identify the artifact type and build process (section 4); decide the branching strategy and CI/CD model together, since they're tightly coupled (sections 3 and 6); design the deployment strategy and its corresponding rollback plan **together**, never as an afterthought (sections 10 and 14); and explicitly plan for how database changes will be sequenced safely relative to code deployments (section 11) — narrating this reasoning, not just the final pipeline diagram, is what a strong answer actually demonstrates.

**Common pitfalls & anti-patterns:**
- Treating deployment as an afterthought bolted on after the "real" application work is done, rather than a first-class part of system design from the start.
- Deploying database schema changes and application code changes as one inseparable, all-at-once step, breaking under a rolling deployment's concurrent old/new code (section 11).
- No tested rollback plan, discovering — under real incident pressure — that rolling back is slow, risky, or not actually well-defined (section 14).
- Relying on a floating `latest` tag instead of immutable, uniquely-versioned artifacts, making "what exactly is running in production right now" genuinely ambiguous (sections 4, 8).
- A deployment pipeline with broad, unscoped credentials — turning a pipeline compromise into a full production compromise (section 17).
- No automated post-deploy verification, relying purely on a human happening to notice something's wrong (section 15).
- Applying heavyweight change-approval process uniformly to every change regardless of actual risk, creating pure friction without corresponding safety benefit (section 16).

**Glossary of deployment terms** — quick cross-reference back to where each is defined in depth: **Deploy vs release** (§1), **Environment parity** (§2), **Trunk-based development** (§3), **Reproducible builds** (§4), **CI** (§5), **Continuous Delivery/Deployment** (§6), **Secrets rotation** (§7), **Immutable infrastructure** (§9), **GitOps** (§9), **Blue/green, Canary, Rolling deployment** (§10), **Expand/contract migration** (§11), **Zero-downtime deployment** (§12), **RTO/RPO** (§14), **Deployment markers** (§15), **Release train** (§16), **Supply chain security** (§17).
