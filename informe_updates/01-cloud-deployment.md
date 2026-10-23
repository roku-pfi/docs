# Cloud deployment (ADR-0031)

- **Changed:** the system runs on AWS, not only locally: EKS for the services,
  RDS Postgres, ElastiCache Valkey, CloudFront for HTTPS, one VPC with private
  subnets. Two OpenTofu stacks: `foundation` (permanent) and `runtime`
  (created by `make aws-up`, destroyed by `make aws-down`, ~US$0.27/h).
- **Review:** any architecture/deployment diagram or text that shows only
  docker compose / k3d; cost section if there is one.
