# ADR-0009: Record visits with GET /api/visit

Status: Accepted.

## Context
- Recording a visit changes state, which would normally be a POST.
- CloudFront origin groups fail over to a second origin only for GET, HEAD and OPTIONS.

## Decision
- `GET /api/visit` records the visit and returns counters plus history.
- Responses carry `Cache-Control: no-store`; CloudFront does not cache the default behaviour.
- The `lab_vid` cookie (HttpOnly, Secure, SameSite=Lax) identifies the visitor.
- The default CloudFront behaviour allows only GET, HEAD and OPTIONS.
- The edge module accepts `dr_alb_dns_name` and builds an origin group with failover on 500, 502, 503, 504.

## Consequences
- Gap: no DR cluster, replica database or failover drill exists. The failover path was never exercised.
- A GET with side effects: prefetchers, crawlers and any page that requests the URL inflate the counters.
- The WAF rate limit is the only mitigation for inflated counts.
- Unsafe methods cannot reach the API through CloudFront at all.

## Alternatives considered
- POST for the visit: semantically correct, but no origin failover.
- Client-side beacon to a separate endpoint: not needed for this scope.
