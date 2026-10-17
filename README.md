# Acme Health Patient Intake API — GRC Engineering Capstone

Fork of cgep-app-starter. Primary framework: HIPAA Security Rule.

## Grader Verification

Prerequisites: [OPA](https://www.openpolicyagent.org/docs/latest/#running-opa)
and [compliance-trestle](https://pypi.org/project/compliance-trestle/)
(`pip install compliance-trestle`, Python 3.10+ recommended).

### 1. Policy Suite

```sh
opa test -v policies/
```

Expected: `PASS: 10/10`

### 2. OSCAL Validation

```sh
cd oscal
python -m trestle validate -f component-definitions/acme-health-hipaa/component-definition.json
python -m trestle validate -f profiles/hipaa-minimum/profile.json
```

Expected: `VALID` for both.

### 3. Pipeline Evidence

See `.github/workflows/grc-gate.yml` for the five-step pipeline
(Plan → Policy Check → Apply → Sign → Upload).

- Red runs (policy gate failures): Actions runs 1–5
- Green run (all steps passed): Actions run 6+

Apply, Sign and Upload run only when the workflow is started manually
from the Actions tab on `main`. Pushes and pull requests run Plan and
Policy Check only.

### 4. Gap Remediation

See `WRITEUP.md` for the gap remediation table mapping all eight
`GAPS.md` items to Terraform resources, Rego policies and OSCAL controls.

## Repo Structure

- `terraform/` — starter workload, plus `terraform/grc/` GRC baseline
- `policies/` — 5 HIPAA Rego policies plus tests
- `.github/workflows/` — `grc-gate.yml` pipeline
- `oscal/` — component definition and profile
- `WRITEUP.md` — design decisions and trade-offs
