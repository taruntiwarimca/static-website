# Static Website on Amazon S3 — deployed by GitHub Actions or GitLab CI

```
site/                       ← your website (index.html, error.html, styles.css)
infra/cloudformation.yml    ← S3 website bucket + IAM roles for CI (OIDC, no access keys)
.github/workflows/deploy.yml← GitHub Actions pipeline
.gitlab-ci.yml              ← GitLab CI pipeline
```

Push to `main` → pipeline assumes an AWS role via OIDC → `aws s3 sync site/` → site is live at the S3 website endpoint.

---

## 1. Prerequisites
- An AWS account and the AWS CLI configured locally (`aws configure`) with admin-level rights for the one-time setup.
- A GitHub repo **or** GitLab project containing these files.

## 2. Create the bucket and CI role (one time)

Pick a globally unique bucket name and region, then deploy the template.

**GitHub:**
```bash
aws cloudformation deploy \
  --stack-name static-site \
  --template-file infra/cloudformation.yml \
  --capabilities CAPABILITY_IAM \
  --region ap-south-1 \
  --parameter-overrides \
      BucketName=my-static-site-12345 \
      GitHubRepo=your-user/your-repo
```

**GitLab:**
```bash
aws cloudformation deploy \
  --stack-name static-site \
  --template-file infra/cloudformation.yml \
  --capabilities CAPABILITY_IAM \
  --region ap-south-1 \
  --parameter-overrides \
      BucketName=my-static-site-12345 \
      GitLabProject=your-group/your-project
```

You can pass both `GitHubRepo` and `GitLabProject` to set up both.
If the account already has the OIDC provider (`token.actions.githubusercontent.com` or `gitlab.com`) from an earlier project, add `CreateGitHubOIDCProvider=false` / `CreateGitLabOIDCProvider=false`, otherwise the stack fails with "already exists".

Read the outputs:
```bash
aws cloudformation describe-stacks --stack-name static-site --region ap-south-1 \
  --query "Stacks[0].Outputs" --output table
```
Note `WebsiteURL` and `GitHubRoleArn` / `GitLabRoleArn`.

### What the template configures
| Setting | Value |
|---|---|
| Static website hosting | Enabled, index `index.html`, error `error.html` |
| Block Public Access | ACLs blocked; public *policy* allowed (needed for website hosting) |
| Bucket policy | Public `s3:GetObject` on objects only |
| Object ownership | Bucket owner enforced (ACLs disabled) |
| Versioning | Enabled (easy rollback) |
| CI role | Trusts only your repo **and only the `main` branch**; can List/Get/Put/Delete in this bucket only |

## 3. Configure the pipeline

### GitHub Actions
Repo → **Settings → Secrets and variables → Actions → Variables** → add:

| Variable | Example |
|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::123456789012:role/static-site-GitHubDeployRole-…` |
| `AWS_REGION` | `ap-south-1` |
| `S3_BUCKET` | `my-static-site-12345` |

The workflow uses an environment called `production`; GitHub creates it automatically on first run (you can add required reviewers to it later).

### GitLab CI
Project → **Settings → CI/CD → Variables** → add the same three variables (`AWS_ROLE_ARN`, `AWS_REGION`, `S3_BUCKET`). They don't need to be masked — none are secrets. If you mark them *Protected*, make sure `main` is a protected branch.

> Self-managed GitLab: replace every `gitlab.com` in `infra/cloudformation.yml` and the `aud:` in `.gitlab-ci.yml` with your GitLab host, and update the provider thumbprint.

## 4. Deploy
```bash
git add . && git commit -m "Initial static site" && git push origin main
```
Watch the run under **Actions** (GitHub) or **Build → Pipelines** (GitLab). Then open the `WebsiteURL`, e.g.
`http://my-static-site-12345.s3-website.ap-south-1.amazonaws.com`

You can also trigger manually: **Run workflow** (GitHub) or **Run pipeline** on `main` (GitLab).

## 5. Verify
```bash
curl -I http://my-static-site-12345.s3-website.ap-south-1.amazonaws.com          # 200
curl -I http://my-static-site-12345.s3-website.ap-south-1.amazonaws.com/nope      # 404 → error.html
aws s3 ls s3://my-static-site-12345/
```

## How the deploy step works
- `aws s3 sync --delete` uploads changed files and removes files you deleted from `site/`.
- Two passes: CSS/JS/images get `Cache-Control: max-age=86400`; HTML gets `no-cache` so content changes show immediately.
- No long-lived AWS keys are stored anywhere — each run gets 1-hour credentials from STS.

## Troubleshooting
| Symptom | Fix |
|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Repo/project name or branch doesn't match the role trust policy. Re-deploy the stack with the exact `owner/repo` (case-sensitive) and branch. |
| `Could not load credentials` (GitHub) | Workflow lacks `permissions: id-token: write`, or `AWS_ROLE_ARN` variable is empty. |
| `403 Forbidden` in the browser | Bucket policy missing or Block Public Access re-enabled on the account level (S3 → *Block Public Access settings for this account*). |
| Stack fails: provider already exists | Set `CreateGitHubOIDCProvider=false` / `CreateGitLabOIDCProvider=false`. |
| Old page still showing | Hard-refresh; HTML is `no-cache` but your browser may hold the old copy from before the first deploy. |

## Optional next steps
- **HTTPS + custom domain:** put CloudFront in front (with Origin Access Control), add an ACM certificate in `us-east-1` and a Route 53 alias; then you can re-enable full Block Public Access on the bucket. Add `aws cloudfront create-invalidation --paths "/*"` to the pipeline.
- **Build step:** for React/Vue/Hugo, add `npm ci && npm run build` before the sync and point it at `dist/` or `public/`.

## Tear down
```bash
aws s3 rm s3://my-static-site-12345 --recursive
aws s3api delete-objects --bucket my-static-site-12345 \
  --delete "$(aws s3api list-object-versions --bucket my-static-site-12345 \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
aws cloudformation delete-stack --stack-name static-site
```
(Versioned buckets also have delete markers; empty the bucket in the S3 console with "Empty" if the command above leaves any.)
