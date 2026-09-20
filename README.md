# ClaimChain

A working recovery workspace for small stores: evidence-backed cases, payment reconciliation, editable correspondence, PDF case packets, follow-up tasks, document recovery requirements, and stock transfers with inventory reservation and receipt confirmation.

## Live deployment

**https://d29p3muqc49obl.cloudfront.net**

Running on AWS in `ap-southeast-2`. Sign in with the demonstration
administrator account; the seeded workspace contains fictional records only.

## AWS services

Nine services, each doing work the application actually depends on.

| Service | Role in ClaimChain | Verified |
| --- | --- | --- |
| Amazon EC2 | Single instance running the Django identity service and the Express workspace service in one container. One instance is deliberate: correctness rests on SQLite's single-writer transaction model, which needs real POSIX file locking. | Yes |
| Amazon EBS | Encrypted persistent volume for the SQLite databases and original evidence files. | Yes |
| AWS IAM | EC2 instance role scoped to only the S3, Textract and Bedrock actions the application performs. No credentials exist in the image, the environment, or this repository. | Yes |
| Amazon S3 | Private evidence mirror, server-side encrypted, with each file's SHA-256 stored as object metadata. | Yes |
| Amazon Textract | `DetectDocumentText` extraction from photographed PNG/JPEG evidence, so recorded facts trace back to the original document. | Yes |
| Amazon Bedrock | Recovery-letter drafting through the Converse API. Evidence is supplied as untrusted data; model output is confined to a draft a human reviews and can never alter balances, stock or case status. | Yes |
| Amazon CloudFront | HTTPS termination and the public origin. The cache policy honours origin headers, so content-hashed assets cache while workspace data never does. | Yes |
| Amazon SES | Transactional email for account verification and password reset, through the SMTP endpoint. | Implemented; configured per deployment |
| Amazon CloudWatch | Application logs and instance metrics. | Yes |

Because `server/aws.ts` shares one `AWS_REGION` across its three clients, S3,
Textract and Bedrock must live in the same region. Bedrock model availability
therefore decides that region, and the evidence bucket follows it.

Deliberately not used: **ECS Fargate** (SQLite over EFS means file locking over
NFS), **Elastic Beanstalk** (replaces instances and would take the data volume
with them), **App Runner** (no persistent storage), and **Amazon Cognito**
(cannot express per-case row-level scoping, and authorization is checked inside
the same transaction as the mutation it guards). See
`docs/06-AWS-DEPLOYMENT-ARCHITECTURE.md` for the full reasoning.

## Run locally

Requires Node.js 24 or newer and Python 3.11 or newer with Django installed.

```sh
npm ci
# One time only if Django is not already installed:
py -m pip install -r django_auth/requirements.txt
npm run dev
```

Open http://127.0.0.1:5173 for the branded landing screen, then enter the workspace. The operational API runs on port 3001 and Django authentication runs on port 8000. The first startup runs Django migrations and creates fictional sample accounts. Changes persist in `data/claimchain.sqlite`, `data/claimchain-auth.sqlite3`, and `data/evidence/`.

Sample sign-in: `admin` / `ClaimChainDemo!2026`. The sample also includes `operator` and `auditor` accounts with the same demonstration password so all three RBAC views can be tested.

For a production-style Node build, run Django separately behind Nginx as described in the deployment documentation. Docker starts both services and proxies `/auth` through the public Express port:

```sh
npm run build
npm start
```

Open http://127.0.0.1:3001. Do not run both commands on an occupied API port. `PORT`, `HOST`, and `DATA_DIR` are configurable through `.env`. To start with empty data, set `SEED_SAMPLE=false` and use a new data directory; do not delete a workspace containing useful records.

## What works

- Create payment, document recovery and evidence-assistance cases.
- Attach real PDF/image/text files, download originals, inspect content hashes and plain text.
- Prepare and edit factual draft letters, protect revisions from stale updates, and download PDFs.
- Record partial/full receipts with duplicate protection and automatic balance/status reconciliation.
- Create and complete follow-ups; maintain an append-only operational event history.
- Complete document recovery requirements and resolve/reopen cases.
- Add stores and inventory, reserve stock, dispatch, cancel reservations, and confirm receipt exactly once.
- Search/filter/sort and paginate records; update business details and export workspace JSON.
- Use individual accounts with email verification, password recovery, scoped RBAC, session revocation, and a security audit.
- Run automatic or manually triggered fictional ingestion when no external feed is connected, with a persisted workspace-wide Continue/Halt control.
- Optionally connect S3, Textract, Bedrock, and SMTP/SES delivery.

## Honest boundaries

Prepared letters are not sent automatically. Payment entries are owner-recorded, not bank-verified. Stock dispatch and receipt are owner-confirmed, not courier integrations. Exported case packets contain a manifest and text, not embedded copies of original binary evidence. Legal drafts are editable factual templates, not jurisdiction-validated legal advice or completed government filings.

Local use needs no cloud credentials. AWS buttons appear only when their server-side settings are configured; live cloud operations still depend on account access. The application is one role-controlled workspace, not a tenant-isolated SaaS service. SQLite data is stored as a versioned atomic workspace snapshot to keep transactions consistent at this scale; scale-out requires a different repository.

The deployed demonstration runs one EC2 instance with no load balancer, no
autoscaling and no automated backup. That matches the single-writer design
rather than working around it, and it means the instance is a single point of
failure. Restoring from an EBS snapshot is a manual step. The Django identity
service runs behind `manage.py runserver`, which is a development server and
would be replaced with a WSGI server such as gunicorn before any real use.

## Verification

```sh
npm run typecheck
npm test
npm run test:auth
npm run build
npx playwright install chromium
npm run test:e2e
```

Browser tests use a separate temporary workspace/port. See `VERIFICATION.md` for actual results, screenshots and remaining limits.

The included GitHub Actions workflow runs formatting, build, API and browser checks on pushes and pull requests once the project is placed in a GitHub repository. That hosted workflow has not been executed in this local workspace.

## Project map

- `docs/README.md`: complete current-state documentation index.
- `docs/README.md`: complete documentation index for business and technical readers.
- `docs/01-EXECUTIVE-OVERVIEW.md`: purpose, outcome, product story, and current maturity.
- `docs/02-PRODUCT-AND-BUSINESS-CASE.md`: users, workflows, value, scope, and boundaries.
- `docs/03-SOLUTION-ARCHITECTURE.md`: components, data flows, and runtime topology.
- `docs/04-DOMAIN-MODEL-AND-OPERATIONAL-WORKFLOWS.md`: entities, safeguards, and state changes.
- `docs/05-API-AND-INTEGRATION-REFERENCE.md`: current HTTP contract.
- `docs/06-AWS-DEPLOYMENT-ARCHITECTURE.md`: selected AWS services and EC2 deployment runbook.
- `docs/07-SECURITY-PRIVACY-AND-RELIABILITY.md`: controls, privacy, risks, and production gate.
- `docs/08-DEVELOPER-AND-OPERATIONS-GUIDE.md`: setup, configuration, and operation.
- `docs/09-QUALITY-ASSURANCE-AND-TESTING.md`: coverage, quality checks, and verification limits.
- `docs/10-DEMO-GUIDE-AND-PRODUCT-ROADMAP.md`: presentation guide and responsible evolution.
- `docs/11-IDENTITY-ACCESS-AND-SIMULATION.md`: Django authentication, RBAC, audit, and simulation.
- `docs/12-DJANGO-IDENTITY-AND-RBAC-IMPLEMENTATION.md`: complete Django authentication and RBAC implementation guide.
- `docs/13-GLOSSARY.md`: plain-language terminology.
- `IMPLEMENTATION_PLAN.md`: full product, technical, UX, AWS, risk and test plan.
- `PRODUCT.md`: product contract.
- `DESIGN.md`: implemented interface system.
- `shared/types.ts`: shared domain types.
- `server/app.ts`: validated API and domain workflows.
- `server/store.ts`: SQLite transaction boundary and fictional initial records.
- `server/aws.ts`: real optional AWS adapters.
- `src/`: React application and interface styles.
- `tests/`: API and browser workflows.

## Demo

Open Kaveri Cafe's case, inspect the fictional invoice, prepare a reminder, edit it and download it. Record a partial receipt and inspect the balance and timeline. Export the packet. In Stock exchange, reserve rice for a second store, mark it dispatched and confirm receipt. Then finish a document-recovery checklist and resolve that case.
