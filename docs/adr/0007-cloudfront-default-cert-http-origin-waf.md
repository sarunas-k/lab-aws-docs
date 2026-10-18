# ADR-0007: CloudFront default certificate, HTTP origin, header-checking WAF

Status: Accepted.

## Context
- Without a domain (ADR-0008) there is no ACM certificate for the ALB's hostname.
- The ALB must still accept traffic only from our CloudFront distribution.

## Decision
- Viewers use HTTPS on `*.cloudfront.net` (default certificate, `redirect-to-https`).
- CloudFront to ALB is plain HTTP (`origin_protocol_policy = http-only`).
- Two origin controls: the ALB security group admits only the CloudFront origin-facing prefix list on port 80, and a regional WAF blocks any request without `X-Origin-Verify` equal to a Terraform-generated secret.
- A CloudFront-scope WAF adds a per-IP rate limit (2000 per 5 minutes by default) and the AWS common rule set.

## Consequences
- Limitation: the CloudFront to ALB hop is not encrypted.
- The prefix list is shared by all CloudFront customers, so the secret header is the real per-distribution control. It travels in clear on that hop.
- The secret lives in Terraform state; no rotation procedure is written.
- Your own `curl` to the ALB is rejected by design.

## Alternatives considered
- ACM certificate on the ALB: needs a domain.
- CloudFront VPC origins with an internal ALB: not evaluated in this repo.
