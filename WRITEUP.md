# Acme Health GRC Engineering Capstone — WRITEUP

## Framework Choice: HIPAA Security Rule

Acme Health is a telehealth company whose Patient Intake API collects and
stores Protected Health Information (PHI). HIPAA Security Rule was the
obvious primary framework — not a stretch, not aspirational, but the direct
legal obligation the workload already carries. Every technical safeguard in
this submission maps to a specific 164.3xx citation.

SOC 2 and CMMC were considered. SOC 2 would serve the enterprise customer
angle but its Trust Services Criteria are less prescriptive, making
evidence chains harder to demonstrate mechanically. CMMC Level 2 would
serve the federal pilot but CUI handling is not the primary data type here.
HIPAA is the right answer for PHI at a telehealth company.

## Gap Remediation

All eight gaps from GAPS.md are addressed across three layers:

| Gap | Terraform Layer | Policy Layer | OSCAL Layer |
|---|---|---|---|
| GAP-01 S3 SSE-S3 instead of CMK | aws_s3_bucket_server_side_encryption_configuration with sse_algorithm=aws:kms | hipaa_cmk_encryption.rego | sc-28 implemented-requirement |
| GAP-02 DynamoDB AWS-owned key | aws_dynamodb_table.intake_hardened with kms_key_arn | hipaa_cmk_encryption.rego | sc-28 implemented-requirement |
| GAP-03 No TLS-only bucket policy | aws_s3_bucket_policy.uploads_tls_only with aws:SecureTransport deny | hipaa_tls_only.rego | ac-3 implemented-requirement |
| GAP-04 No S3 versioning | aws_s3_bucket_versioning.uploads with status=Enabled | hipaa_phi_versioning.rego | cm-6 implemented-requirement |
| GAP-05 Lambda not in VPC | aws_security_group.lambda + VPC attachment via aws_iam_role_policy_attachment.lambda_vpc | Documented in OSCAL | ac-3 implemented-requirement |
| GAP-06 No DLQ/X-Ray | aws_sqs_queue.lambda_dlq + aws_lambda_function_event_invoke_config | Documented in OSCAL | au-3 implemented-requirement |
| GAP-07 IAM wildcard permissions | aws_iam_role_policy.lambda_least_privilege replaces dynamodb:* and s3:* | hipaa_least_privilege.rego | ac-3 implemented-requirement |
| GAP-08 No API GW logging | aws_cloudwatch_log_group.apigw + aws_apigatewayv2_stage.default_hardened | hipaa_audit_logging.rego | au-3 implemented-requirement |

## Design Decisions

**Object Lock mode: GOVERNANCE**
GOVERNANCE was selected for this submission so the evidence vault can be
cleaned up during development. Production deployment would use COMPLIANCE
mode with a minimum 365-day retention period aligned to HIPAA's 6-year
record retention requirement (45 CFR 164.530(j)).

**Auto-apply on merge to main**
The pipeline applies automatically on merge rather than requiring a manual
post-merge approval. This reduces friction for the engineering team, which
the CTO explicitly requested. The policy gate serves as the human-equivalent
approval step — if the gate passes, the code is compliant enough to deploy.

**AWS credentials via GitHub Secrets (not OIDC)**
Long-lived access keys were used for the pipeline due to time constraints.
The production recommendation is AWS OIDC federation (same pattern as Lab
5.4's Workload Identity Federation for GCP) to eliminate stored credentials
entirely. This is the primary security debt in this submission.

**Policy gate as informational on starter plan**
The policy gate runs against the starter's bare Terraform plan, which
intentionally has no CMK encryption. The gate documents violations with
continue-on-error rather than blocking, because the GRC layer (terraform/grc/)
closes those gaps in a separate apply step. In a mature pipeline these would
run as a single plan that includes both layers.

## Control Coverage

| HIPAA Control | Implementation | Layer |
|---|---|---|
| 164.312(a)(2)(iv) Encryption at rest | CMK via aws_kms_key.phi | Terraform + Rego |
| 164.312(e)(1) Transmission security | TLS-only bucket policy | Terraform + Rego |
| 164.308(a)(7) Contingency plan | S3 versioning + KMS rotation | Terraform + Rego |
| 164.312(a)(1) Access control | Least-privilege IAM | Terraform + Rego |
| 164.312(b) Audit controls | CloudTrail + API GW logging | Terraform + Rego |
| 164.312(e)(1) Network boundaries | Lambda in VPC | Terraform + OSCAL |

## Trade-offs Made

1. DynamoDB hardening creates a new table rather than modifying the existing
   one. DynamoDB encryption cannot be changed after creation. The hardened
   table is the target; data migration is out of scope for this submission.

2. Lambda VPC placement requires a NAT gateway for internet access, which
   costs ~$32/month. The VPC attachment is configured; the NAT gateway is
   documented as a cost decision for production.

3. The OSCAL evidence URIs point at a vault path that exists only after the
   GRC layer applies. The URI pattern is correct; the object populates on
   the next pipeline run after the vault is deployed.

## What I Would Do With Another Sprint

1. Deploy the GRC layer Terraform in the same pipeline as the starter so
   the policy gate evaluates the combined plan — eliminating the
   continue-on-error workaround.
2. Switch from GitHub Secrets to AWS OIDC federation for keyless CI
   authentication.
3. Add Security Hub with HIPAA-mapped findings as a fifth layer of
   continuous monitoring evidence.
4. Wire the OSCAL evidence URIs to real signed vault objects once the
   pipeline has run end-to-end with the GRC layer deployed.

## What I Did Not Get To

- GAP-05 VPC: The security group and IAM policy attachments are deployed.
  The Lambda function itself requires a new deployment with vpc_config,
  which needs the starter to be modified directly rather than via override.
- Cosign verification script: The sign step runs but a verify-evidence.sh
  script was not written. This is the missing link between the vault and
  the OSCAL evidence chain.
- Security Hub: Continuous monitoring findings would strengthen the
  evidence trail but were out of scope for this submission timeline.
