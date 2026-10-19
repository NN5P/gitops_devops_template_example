# gitops_devops_template_example

Reusable GitHub Actions workflows shared by every app repo. An app repo only keeps small caller
workflows; the build and deploy logic lives here.

Flow: app repo builds an image to ECR, then bumps the image tag in the GitOps repo
([NN5P/gitops_argocd_example](https://github.com/NN5P/gitops_argocd_example)), and ArgoCD deploys it.

| Workflow | Does |
|---|---|
| `docker-build.yml` | Build the caller's Dockerfile, push `<registry>/<ecr-repository>:<sha>` to ECR |
| `gitops-update.yml` | Set `.image.tag` in `apps/<app-name>/values-<environment>.yaml` in the GitOps repo, or copy it from another environment (`promote-from`) |

## docker-build.yml

| Input | Default | |
|---|---|---|
| `ecr-repository` | required | ECR repository name |
| `aws-region` | `""` | Required when `push: true` |
| `aws-role-arn` | `""` | IAM role for GitHub OIDC. Empty = use the access key secrets |
| `dockerfile` | `Dockerfile` | Path from the repo root, e.g. `api/Dockerfile` |
| `context` | Dockerfile's folder | Build context |
| `platforms` | `linux/amd64,linux/arm64` | Image architectures. arm64 is built under QEMU (slower) |
| `push` | `true` | `false` = build only (pull requests, no AWS access) |

Secrets (only without `aws-role-arn`): `aws-access-key-id`, `aws-secret-access-key`.
Outputs: `image`, `tag`.

The job runs with the caller's permissions: grant `id-token: write` when using OIDC.

## gitops-update.yml

| Input | Default | |
|---|---|---|
| `app-name` | required | Chart folder under `apps/` in the GitOps repo |
| `environment` | required | `dev` / `prod` -> `apps/<app-name>/values-<environment>.yaml` |
| `image-tag` | `""` | Tag to set |
| `promote-from` | `""` | Copy the tag from `apps/<app-name>/values-<promote-from>.yaml` instead |
| `gitops-repo` | `NN5P/gitops_argocd_example` | |
| `gitops-branch` | `main` | |

Secret: `gitops-token`, a fine-grained PAT with Contents read/write on the GitOps repo.
It only rewrites the `tag:` line, does nothing if the tag is unchanged, and rebases and retries
if another app pushed at the same time.

## Using it in a new app repo

Copy `.github/workflows/{dev,prod,pr}.yml` from
[NN5P/gitops_app_example](https://github.com/NN5P/gitops_app_example) and change `app-name`,
`ecr-repository` and (if needed) `dockerfile`. Then set in the app repo:

| Name | Kind | Value |
|---|---|---|
| `GITOPS_TOKEN` | secret | PAT with Contents read/write on the GitOps repo |
| `AWS_REGION` | variable | ECR region |
| `AWS_ROLE_ARN` | variable | OIDC role with ECR push rights (keep it a variable: a secret masks the account ID and drops the `image` output) |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | secrets | only without `AWS_ROLE_ARN` |

The app also needs a chart in the GitOps repo (copy `_template/`) and an ECR repository.

If this repo is private, allow access under Settings -> Actions -> General -> Access.

## Versioning

Callers use `@main`, so a change here reaches every app on its next run. Once stable, tag a
release (`git tag v1 && git push origin v1`) and point callers at `@v1`.
