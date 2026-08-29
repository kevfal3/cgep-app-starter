# Acme Health Patient Intake API — GRC Engineering Capstone

Fork of cgep-app-starter. Primary framework: HIPAA Security Rule.

## Grader Verification

### 1. Policy Suite
Run: opa test -v policies/
Expected: PASS 10/10

### 2. OSCAL Validation
Run from oscal/ directory:
python -m trestle validate -f component-definitions/acme-health-hipaa/component-definition.json
python -m trestle validate -f profiles/hipaa-minimum/profile.json
Expected: VALID for both

### 3. Pipeline Evidence
See .github/workflows/grc-gate.yml for the five-step pipeline.
Red runs (policy gate blocked): Actions runs 1-5
Green run (all steps passed): Actions run 6+

### 4. Gap Remediation
See WRITEUP.md for full gap remediation table mapping all eight
GAPS.md items to Terraform resources and Rego policies.

## Repo Structure
- terraform/ - Starter workload plus terraform/grc/ GRC baseline
- policies/ - 5 HIPAA Rego policies plus tests
- .github/workflows/ - grc-gate.yml pipeline
- oscal/ - Component definition and profile
- WRITEUP.md - Design decisions and trade-offs
