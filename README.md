# AWS Automated Access Review

> A hands-on cloud-security lab where I deployed a serverless access-review pipeline that audits AWS IAM, scans Security Hub / Access Analyzer / CloudTrail, and emails a clean executive report — auditor-ready, zero manual work.

**Built by:** [Shuayb](https://www.linkedin.com/in/shu-/) — student / self-learner exploring cloud security, IAM, and Python automation.
**Original scaffold:** [@ajy0127](https://github.com/ajy0127/aws_automated_access_review) — extended and re-documented as a learning exercise.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Code Style: Black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

---

## What this project does

Manual access reviews are the auditor-pain that won't go away — every quarter someone clicks through IAM, exports CSVs, eyeballs MFA, and writes the same summary. This project automates the entire loop:

1. A scheduled Lambda runs every 30 days (configurable)
2. It pulls findings from **IAM**, **Security Hub**, **IAM Access Analyzer**, **CloudTrail config**, and checks for **SCPs**
3. **Amazon Bedrock** (Claude) generates a plain-English executive narrative
4. A timestamped CSV + the AI summary land in S3 and in your inbox via SES

Result: monthly evidence packets ready to drop into a SOC 2 / ISO 27001 audit folder.

## Real run from my AWS lab

I deployed this in my own sandbox account and the first scheduled report flagged **7 real findings** — a useful (and slightly embarrassing) snapshot of the lab's state:

| Severity | Category | Finding |
|---|---|---|
| **High** | IAM | An IAM user with console access had **no MFA** |
| **High** | IAM | The account had **no password policy** configured |
| **High** | CloudTrail | **CloudTrail was not enabled** in the account |
| Medium | IAM | An access key was **124 days old** (CIS 1.4 says rotate every 90) |
| Medium | SCP | **No custom Service Control Policies** in the org |
| Info | Security Hub | No high/critical IAM findings — clean |
| Info | Access Analyzer | No external-access findings — clean |

Each finding includes the CIS / AWS Well-Architected control it maps to and a remediation recommendation. A redacted version of the CSV is in [`examples/my-run-redacted.csv`](examples/my-run-redacted.csv).

> All three High findings were genuine misconfigurations in my lab — fixing them was a useful exercise in turning a finding into a remediation ticket.

## Architecture

```
┌──────────────┐
│ EventBridge  │ (cron: every 30 days)
└──────┬───────┘
       ▼
┌────────────────────┐    pulls findings from    ┌──────────────────────┐
│  Lambda (Python)   │◀─────────────────────────▶│ IAM · Security Hub   │
│  modular checks    │                           │ Access Analyzer      │
└──────┬─────────────┘                           │ CloudTrail · SCPs    │
       │  raw findings                           └──────────────────────┘
       ▼
┌────────────────────┐    AI narrative           ┌──────────────────────┐
│  Amazon Bedrock    │◀─────────────────────────▶│  Claude model        │
└──────┬─────────────┘                           └──────────────────────┘
       │  CSV + summary
       ▼
┌────────────────────┐         ┌──────────────────┐
│       S3           │────────▶│  SES → email     │
└────────────────────┘         └──────────────────┘
```

**AWS services used:** Lambda · S3 · SES · EventBridge · Bedrock · Security Hub · IAM Access Analyzer · CloudTrail · CloudFormation

## Tech stack

- **Python 3.11** with modular checks (`iam_findings.py`, `securityhub_findings.py`, `access_analyzer_findings.py`, `cloudtrail_findings.py`, `scp_findings.py`)
- **boto3** — AWS SDK
- **Amazon Bedrock (Claude)** — narrative generation
- **CloudFormation** — IaC (`templates/access-review-real.yaml`)
- **pytest** + **flake8** + **black** — tests, lint, formatting
- **GitHub Actions** — CI on every push

## What I learned

Honest list of things this project taught me:

- **Mapping findings to controls.** It's one thing to detect "no MFA"; it's another to tag it with `CIS 1.2` so an auditor can use it as evidence. The `compliance` column in every finding made the report feel real.
- **AI is great at narrative, not detection.** Bedrock's executive summary saves real time on the *write-up*, but the actual checks must be deterministic Python — not LLM judgment calls.
- **SES verification is non-obvious.** First deploy "succeeded" but no email arrived because I hadn't clicked the SES verification link. The README now has a giant warning.
- **CloudFormation has two patterns for Lambda code.** Embedded (in the YAML, fast for demos) vs. external zip (production). This repo includes both — useful for understanding the trade-off.
- **Modular Lambda matters.** Splitting checks into modules (`src/lambda/modules/*.py`) made it easy to add tests and to reason about which AWS API call belongs where.
- **My lab was less secure than I thought.** Three High findings in a personal account — a humbling reminder that GRC work starts at home.

## Prerequisites

- AWS CLI configured with appropriate permissions
- Python 3.11+
- AWS services enabled in your account:
  - **AWS Security Hub**
  - **IAM Access Analyzer**
  - **Amazon SES** (with a verified email)
  - **Amazon Bedrock** (with access to a Claude model)

## Quick start

```bash
# 1. Clone
git clone https://github.com/nb67df5gpm-hash/aws-automated-access-review.git
cd aws-automated-access-review

# 2. Install deps
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# 3. Verify AWS creds
./scripts/check_aws_creds.sh --profile your-aws-profile

# 4. Deploy
./scripts/deploy.sh --email your.email@example.com --profile your-aws-profile

# 5. Click the SES verification link in your inbox  ⚠️  REQUIRED

# 6. Trigger an immediate report
./scripts/run_report.sh --profile your-aws-profile
```

Optional flags:
- `--stack-name` — CloudFormation stack name (default `aws-access-review`)
- `--region` — region (default `us-east-1`)
- `--schedule` — EventBridge schedule (default `rate(30 days)`)

## What you receive

- **Email** — executive summary (Bedrock-generated) + CSV attachment
- **S3 object** — same report stored for audit evidence
- **CSV columns** — `id, category, severity, resource_type, resource_id, description, recommendation, compliance, detection_date`

## Repo layout

```
src/
├── cli/                          # Local CLI entrypoints
└── lambda/
    ├── index.py                  # Lambda handler
    └── modules/
        ├── iam_findings.py
        ├── securityhub_findings.py
        ├── access_analyzer_findings.py
        ├── cloudtrail_findings.py
        ├── scp_findings.py
        ├── narrative.py          # Bedrock narrative generation
        ├── reporting.py          # CSV output
        └── email_utils.py        # SES delivery
templates/
├── access-review.yaml            # Embedded-Lambda variant (demo-friendly)
└── access-review-real.yaml       # Production variant (zip-deployed Lambda)
scripts/
├── deploy.sh                     # Deploy stack + Lambda zip
├── run_report.sh                 # Manually invoke a report
└── check_aws_creds.sh            # Pre-flight credential check
tests/unit/                       # Pytest unit tests
examples/
├── sample-access-report.csv      # Original tutorial sample
└── my-run-redacted.csv           # My real run (redacted)
```

## Running tests locally

```bash
python -m pytest tests/unit
```

## Cost & scale

- **~$1/month** in `us-east-1` for a typical lab account
- Tested on accounts with up to ~2000 resources / 500 IAM entities
- Lambda completes in 2–3 minutes per run

## Troubleshooting (from my own deploys)

| Symptom | Likely cause | Fix |
|---|---|---|
| Email never arrives | SES sender not verified | Click the verification link in the inbox AWS sent to your address |
| `Access denied` on Bedrock | Model access not requested | Open Bedrock console → Model access → request Anthropic Claude |
| Lambda times out | Large account | Bump `Timeout` in `templates/access-review-real.yaml` (default 5 min) |
| CSV missing CloudTrail row | CloudTrail genuinely not enabled | Enable a trail; the check correctly reports the gap |
| `No custom SCPs detected` flagged | Account is not part of an Org with SCPs | Informational; expected for standalone lab accounts |

## Credit

Built on top of the excellent scaffold by [@ajy0127](https://github.com/ajy0127/aws_automated_access_review) (part of the [GRC Portfolio Hub](https://github.com/ajy0127/grc_portfolio)). My work here is deployment, real-world testing, troubleshooting, and documentation — sharing the experience as a learning artifact.

## License

MIT — see [LICENSE](LICENSE).

---

Maintained by [Shuayb](https://www.linkedin.com/in/shu-/).
