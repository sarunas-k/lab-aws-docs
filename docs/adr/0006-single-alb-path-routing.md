# ADR-0006: One ALB with in-cluster path routing

Status: Accepted.

## Context
- Two workloads (`lab-web` on 8080, `lab-backend` on 8000) share one public entry point.
- Each ALB has a fixed hourly cost.

## Decision
- One internet-facing ALB `lab-<env>-alb`, created by the AWS Load Balancer Controller from two Ingresses in the IngressGroup `lab`.
- `/api` (order 10) goes to `lab-backend`; `/` (order 20) goes to `lab-web`. Target type is `ip`, listener is HTTP 80.
- The ALB uses a Terraform-managed security group. Terraform opens node ports 8000 and 8080 from it.

## Consequences
- The ALB belongs to Kubernetes, not Terraform. The `edge` stack reads it with `data "aws_lb"`, so it must be applied after Argo CD syncs, and destroyed before `cluster`. `scripts/teardown.sh` deletes Ingresses first.
- The ALB security group id (`sg-TODO`) must be copied from Terraform output into gitops values.
- Behaviour is defined in controller annotations, which are controller-specific.
- Migration path: Gateway API `HTTPRoute` resources would replace the annotations. Not implemented; controller support level was not checked.

## Alternatives considered
- One ALB per service: more cost.
- Ingress NGINX behind an NLB: an extra component to run.
