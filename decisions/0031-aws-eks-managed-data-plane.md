# 31. AWS: EKS for the workloads, managed data plane, CloudFront in front

- Status: Accepted
- Date: 2026-10-21
- Related: [ADR-0001](0001-repo-and-deployment-strategy.md) (microservices on
  Kubernetes), [ADR-0020](0020-local-k8s-k3d-helm.md) (k3d + Helm, which stays
  as the local mode), [ADR-0021](0021-prometheus-grafana-in-chart.md).
- Requirements: **RNF-09, RNF-10, RNF-11** (E90 §3.2.4).
- Source: the infrastructure interview (E90 §3.1.2.3, Annex C.1) and a follow-up
  conversation with the same interviewee.

## Context

The system runs only on a laptop: images are loaded into k3d
(`imagePullPolicy: Never`), datastore passwords are literals in `values.yaml`,
and WebAuthn is pinned to `localhost`. The final delivery needs it reachable on
the internet, both to demonstrate it and to run the functional and usability
tests that chapter 3 still has to report.

The interview offered two paths. **Path 1**: managed data plane, stateless
application workloads on a serverless container platform (ECS/Fargate), a
private registry, CI, infrastructure as code. **Path 2**: managed Kubernetes,
charts, GitOps, node autoscaling, which he described as where a growing product
ends up and as "very similar to what you have today".

Options considered:

| | ECS Fargate | EKS | One VM with k3s |
|---|---|---|---|
| Images | unchanged | unchanged | unchanged |
| Helm charts | rewritten as task definitions | reused with an AWS values file | reused |
| HPA, Prometheus, Grafana, init containers | rebuilt on ECS/CloudWatch | unchanged | unchanged |
| RNF-11 ("the same package goes to a managed orchestrator") | needs rewording | met literally | not a managed orchestrator |
| Portability to another provider | low | high (Terraform modules change, charts do not) | high |
| Hourly cost while running (estimate) | ≈ US$0.18 | ≈ US$0.22 | lowest |

The difference in cost is small at this scale, and the environment is switched
on only for rehearsals, tests and the defense, so the hourly figure is what
matters (≈ US$10/month at ~40 h of use). The VM option was ruled out: it does
not exercise a managed orchestrator, which is what RNF-11 promises.

## Decision

**EKS for application workloads; RDS and ElastiCache for the state; CloudFront
for HTTPS.** Path 2's orchestrator with path 1's managed data plane.

### Network

- One VPC across two availability zones, with a **public** and a **private**
  subnet in each.
- **Private subnets**: EKS nodes, RDS, ElastiCache. Nothing in them is reachable
  from the internet, regardless of security groups.
- **Public subnets**: the ALB and one NAT gateway (for nodes to pull images and
  reach AWS APIs).
- Security groups: datastores accept 5432/6379 only from the node security
  group; the ALB accepts only the AWS-managed CloudFront origin-facing prefix
  list.
- Pod-to-pod and pod-to-datastore traffic stays inside the VPC (VPC CNI: pods
  get VPC addresses).

### Compute

- EKS with one managed node group of **Graviton (arm64)** instances
  (`t4g.medium` × 2 to start). Images are built multi-arch (arm64 + amd64) so the
  same tag runs on Apple-silicon laptops, k3d and EKS. Falling back to `t3` is a
  one-line change if a dependency ever lacks an ARM build.
- Charts from `rba-infra/helm` with a `values-aws.yaml` overlay that disables
  the in-cluster Postgres and Redis and points at the managed endpoints.
- AWS Load Balancer Controller provisions the ALB from Ingress objects;
  metrics-server keeps the HPA working as it does locally.
- Prometheus and Grafana stay in the chart (ADR-0021), unchanged.

### State

- **RDS PostgreSQL**, single instance, still one database per service (ADR-0010
  semantics: services own databases, infra owns the server).
- **ElastiCache for Valkey**, single node. Valkey is Redis-protocol compatible;
  `redis-py` and `ProfileState` are unchanged. Chosen over Redis for licensing
  and price, as the interviewee recommended.
- **RabbitMQ stays in the cluster.** The outbox in Postgres is the durable record
  of every decision; the broker is transport. Accepted risk: if the broker pod
  dies with messages still queued, those profile/audit updates are lost. That is
  stated in the informe rather than paid for with Amazon MQ.
- After this, the only stateful workload in the cluster is RabbitMQ (plus
  Prometheus' scratch storage).

### Edge: CloudFront instead of a domain

WebAuthn requires a secure context, and ACM cannot issue a certificate for an
`*.elb.amazonaws.com` name, so the ALB URL alone cannot serve passkeys.
CloudFront gives each distribution an HTTPS `*.cloudfront.net` name for free.

- One distribution per public surface: the RokuSec IdP (login + console), Roku
  Bank, Kaze Labs. The IdP distribution name is the WebAuthn relying-party ID.
- Distributions forward the viewer `Host` header; the Ingress routes by those
  (stable) host names.
- Distributions live in the permanent stack, so their names never change. On
  every `up`, the origin is re-pointed to the new ALB (a foundation re-apply,
  ~5 min). While the runtime is down they answer 502.
- Buying a domain later only adds aliases and an ACM certificate; nothing else
  moves.

### Infrastructure as code: two OpenTofu stacks

Written for **OpenTofu** (`tofu`), the open-source fork of Terraform: same
language and providers, and the one Homebrew still ships after Terraform's
licence change. "Terraform" below means the language, not the binary.

| Stack | Lifetime | Contents | Cost while idle |
|---|---|---|---|
| `foundation` | permanent | VPC + subnets + IGW, ECR repositories, CloudFront distributions, GitHub OIDC role, attacker Lambdas, AWS Budgets alert, S3 state bucket | ≈ US$1/month |
| `runtime` | created and destroyed per session | NAT gateway, EKS + node group, RDS, ElastiCache, controllers, Helm releases, seed job | none once destroyed |

`make up` / `make down` wrap the two applies. Terraform state lives in S3 with
native locking. Region: **us-east-1**, matching the prices quoted in chapter 4;
TLS terminates at the CloudFront edge nearest the viewer, so the region adds
round-trip time only on the origin leg.

### Secrets

No Secrets Manager. Terraform generates every password (`random_password`) and
writes them into Kubernetes Secrets. They exist in Terraform state (encrypted
S3) and in the cluster, never in git. The `rba`/`admin` literals remain for k3d
only.

### CI/CD

GitHub Actions in each service repo, from one reusable workflow: test → build
multi-arch → push to ECR, authenticated with **OIDC** (no long-lived AWS keys).
Deploy is `helm upgrade` against the running cluster when one exists. GitOps
(Argo CD / Flux) is not adopted: with an environment that is destroyed after
each session there is nothing for a reconciler to reconcile between sessions.

## Consequences

**Good.**

- RNF-11 holds literally: the chart that runs on k3d is the chart that runs on
  EKS, plus one values file.
- RNF-10 is now true at infrastructure level, not only in code: pods hold no
  state that matters, so replicas can be replaced freely.
- The local k3d mode (ADR-0020) keeps working; the two modes differ only in
  values.
- The cost of an idle month is about a dollar.

**Costs and risks.**

- `make up` takes ~20 minutes (EKS control plane is the slow part). Rehearsals
  need to start early.
- RDS is destroyed with the runtime, so **passkeys are re-registered at the start
  of each session**. Demo data is re-seeded by a job on every `up`.
- The EKS control plane (US$0.10/h) is the largest single line while running.
- `*.cloudfront.net` names look less like a product than a domain would.
- Multi-arch builds lengthen CI.

## Alternatives rejected

- **ECS Fargate.** Cheaper by a few cents per hour and the interviewee's first
  recommendation for a small team, but it discards the charts, HPA and the
  Prometheus/Grafana set-up, and makes RNF-11 untrue.
- **Datastores inside the cluster** (CloudNativePG, a Valkey operator). Free in
  money, expensive in operation, and the interview advised against it for a
  team of one.
- **Public subnets for the nodes** to save the NAT gateway. Saves ≈ US$2 for 40
  hours of use and leaves the nodes one security-group mistake away from the
  internet.
- **Amazon MQ for RabbitMQ.** ≈ US$20–30/month for a guarantee the outbox already
  provides where it matters.
- **Secrets Manager.** Rotation and audit are real features, but nothing in this
  deployment lives long enough to need rotation.
