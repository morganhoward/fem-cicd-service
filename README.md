# fem-cicd-service — Enterprise stage

This branch holds the end-of-segment-15 state of the Frontend Masters workshop "Cloud CI/CD with GitHub Actions". The pipeline now meets the security and operational bar a regulated organization expects: deploys are gated by a `production` environment with required reviewers and a wait timer, AWS authentication uses OIDC instead of long-lived access keys, CloudFront with Origin Access Control fronts a private S3 bucket, every third-party action is SHA-pinned, workflow permissions are deny-all by default, and concurrency controls serialize production deploys.

## What this branch shows

The pipeline has the same shape as Stable — a composite action at `.github/actions/build-astro/action.yml`, a reusable workflow at `.github/workflows/_build.yml`, a `ci.yml` PR gate, and a `deploy.yml` deployer — but every workflow file is hardened. Third-party actions are SHA-pinned with a trailing version comment (per OWASP CICD-SEC-3). Each workflow declares `permissions: {}` at the top level to deny everything by default and re-grants the minimum scope per job (CICD-SEC-1). `ci.yml` uses a PR-keyed `concurrency:` block with `cancel-in-progress: true` so superseded PR runs are cancelled, and `deploy.yml` uses a `production-deploy` concurrency group with `cancel-in-progress: false` so production deploys queue rather than race. The deploy job targets a `production` environment, and AWS authentication uses `aws-actions/configure-aws-credentials` with `role-to-assume` against an OIDC-trusted IAM role (CICD-SEC-2 / CICD-SEC-6) — no long-lived AWS keys live in repository secrets.

## AWS prerequisites

Create these by hand in the AWS console before pushing. Use the placeholders `<example-bucket>`, `<DISTRIBUTION_ID>`, `<ACCOUNT_ID>`, `<owner>/<repo>`, and `<role-name>` exactly as written — substitute real values locally and never commit them.

- One S3 bucket. Public access is fully blocked. Static-website hosting is disabled. The bucket policy grants `s3:GetObject` only to the CloudFront distribution's Origin Access Control service principal, conditioned on the distribution ARN. The bucket is no longer publicly readable.
- One CloudFront distribution fronting `<example-bucket>` via Origin Access Control. The distribution's ID is `<DISTRIBUTION_ID>` and is referenced by the `cloudfront:CreateInvalidation` step in `deploy.yml`.
- One GitHub Actions OIDC provider in AWS IAM at `arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com`. Pre-create this — creating it live during the workshop requires `iam:CreateOpenIDConnectProvider`, which most demo accounts will not grant on the fly.
- One IAM role (NOT an IAM user) whose trust policy targets the OIDC provider above. The trust policy's `sub` condition binds to `repo:<owner>/<repo>:environment:production` so only workflow runs from the `production` environment in this repo can assume the role (TDD §6.4). The role's permission policy grants `s3:PutObject`, `s3:DeleteObject`, and `s3:ListBucket` on `<example-bucket>` plus `cloudfront:CreateInvalidation` on the distribution. No wildcards.
- The IAM user from POC and Stable is deleted. The long-lived access keys it held are gone.

## GitHub prerequisites

Configure these on the repository hosting this branch before the first push.

- The repository must be public.
- A `production` environment exists in repository settings with required reviewers (the demo account self-approves for the live workshop) and a wait timer set to 1 minute. A more realistic production wait is 30 minutes; the short timer keeps the demo moving.
- A repository **variable** `AWS_ROLE_TO_ASSUME` set to the IAM role's ARN (`arn:aws:iam::<ACCOUNT_ID>:role/<role-name>`). This is a variable, not a secret — the role ARN is not sensitive on its own; the OIDC trust policy is what protects assumption of the role.
- No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` repository secrets. They were removed in segment 13 along with the IAM user.
- A branch protection rule on `main` carried forward from Stable: require a pull request and require the `build` status check from `ci.yml` to pass.
- Replace the `<example-bucket>` and `<DISTRIBUTION_ID>` placeholders in the workflow files with the real values before pushing. The committed `<40-char-sha>` placeholders on third-party action references are intentional teaching artifacts — the instructor pins real 40-character commit SHAs at workshop time.

## What was deliberately solved

This stage exists to invert the deliberate "wrong choices" carried by POC and Stable. Each item below is the corrected end-state and maps back to the OWASP CI/CD Top 10.

- Long-lived AWS credentials in repository secrets are replaced by OIDC and short-lived role assumption (CICD-SEC-2, CICD-SEC-6).
- Mutable `@v4`-style action tags are replaced by SHA pins with a version comment (CICD-SEC-3). A retagged release can no longer silently change what runs in CI.
- Auto-deploy on every push to `main` is replaced by an environment gate with required reviewers and a wait timer.
- Unbounded concurrent runs are replaced by `concurrency:` blocks: PR-keyed cancel for `ci.yml`, queued and non-cancelling for `deploy.yml` so production deploys serialize.
- Public-read S3 is replaced by a private bucket fronted by CloudFront with Origin Access Control (CICD-SEC-1, least privilege at the storage layer).
- Inherited from Stable and retained: the PR-validation gate, npm caching, the composite action, and the reusable workflow.

Self-hosted runners are discussed in segment 15 as the next operational lever for an enterprise pipeline. They are intentionally not built in this branch.

## Linkouts

- Master outline: `docs/instructor/OUTLINE.md`.
- Enterprise stage step-by-step guide: `docs/instructor/ENTERPRISE.md`.
- Architecture TDD: `docs/tdd/fem-cicd-workshop-architecture.md`.
- Previous stages: run `git checkout poc` or `git checkout stable`.
- Workshop URL: https://frontendmasters.com/workshops/github-actions/.