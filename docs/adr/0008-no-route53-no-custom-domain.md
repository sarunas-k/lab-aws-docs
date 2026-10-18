# ADR-0008: No Route 53 and no custom domain

Status: Accepted.

## Context
- A domain costs money each year and a hosted zone has a monthly fee; prices were not checked here.
- The project is a portfolio lab with a fixed budget.

## Decision
- Access is the CloudFront `*.cloudfront.net` URL.
- No hosted zone, no ACM certificates, no DNS records.

## Consequences
- The URL is not memorable and changes if the distribution is recreated (edge is a destroyable stack).
- The origin hop stays HTTP (ADR-0007).
- No DNS-based failover. The DR option relies on CloudFront origin groups instead (ADR-0009).
- Adding a domain later would need: a Route 53 zone or external DNS, a CloudFront alias with an ACM certificate in us-east-1, and an ACM certificate on the ALB with `https-only` origin protocol. None of this is written.
- The SCP region allow-list already treats Route 53 as a global service, so it would not block this.

## Alternatives considered
- Domain with Route 53 and ACM: better UX and end-to-end TLS, rejected on cost and scope.
