# ADR-0013: VPC CNI network policy left in standard mode

Status: Accepted.

## Context
- The VPC CNI add-on is configured with `enableNetworkPolicy`.
- Strict mode default-denies every pod without a policy, including CoreDNS, the load balancer controller and Argo CD.

## Decision
- Standard mode (the default). Only the `lab` namespace is restricted: default-deny ingress and egress, a DNS rule, and per-workload allow rules.
- Pod Security `restricted` is enforced on `lab`; node IMDS hop limit is 1.

## Consequences
- Other namespaces (`kube-system`, `argocd`, `cnpg-system`, `external-secrets`) have no network policy.
- Postgres egress to S3 is allowed to all addresses outside the VPC on port 443, because policies cannot select a service by name. An S3 gateway endpoint and a CNI with FQDN policies would narrow it.
- The Kubernetes API rule uses the `kubernetes` Service ClusterIP (`kubernetesServiceIp`, default 172.20.0.1/32). Confirm it with `kubectl get svc kubernetes`.

## Alternatives considered
- Strict mode with policies for every system namespace: more work and risk of locking out the cluster.
- Calico or Cilium: not evaluated.
