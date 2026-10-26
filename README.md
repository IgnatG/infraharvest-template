# infraharvest-template

A GitHub template repository that sets up a new repository to bring an existing AWS account, Azure subscription or Google Cloud project under Terraform with [infraharvest](https://github.com/IgnatG/infraharvest)'s [GitHub Action](https://github.com/IgnatG/infraharvest/blob/main/action/README.md). The workflows only read the cloud and write files; nothing is applied until you apply it.

## What you get

| File | What it does |
|---|---|
| [`.github/workflows/import.yml`](.github/workflows/import.yml) | Every Monday at 06:00 UTC, and when you run it: imports the services you name into Terraform roots under `infraharvest-out/`, and opens or updates a pull request on `infraharvest/import` with the configuration, the import report as its body. Later runs add only what is new to the roots the repository has. |
| [`.github/workflows/drift.yml`](.github/workflows/drift.yml) | Every day at 05:00 UTC, and when you run it: checks the applied roots for changes made in the cloud since, and opens or updates a pull request on `infraharvest/drift` with the drift report and the decisions file, `infraharvest-out/drift.yaml`. With [several environments](#several-environments), one job and one pull request per environment. |
| [`infraharvest.yaml`](infraharvest.yaml) | infraharvest's configuration: the state backend of the generated roots, and environments. |
| [`.github/workflows/template-test.yml`](.github/workflows/template-test.yml) | Tests the template. It runs only in `IgnatG/infraharvest-template`; in your repository every job is skipped, and you can delete it. |

Instead of a pull request, the import can [commit straight to a branch](#committing-straight-to-a-branch).

> [!IMPORTANT]
> infraharvest has no release yet. The Action installs the release that the `INFRAHARVEST_VERSION` variable names and verifies its signature, so the workflows can't run until a release exists. The workflows pin the Action to a commit on infraharvest's `main` (`IgnatG/infraharvest/action@<sha> # main`). Once a release exists, change that pin in `import.yml` and `drift.yml` to the commit of the release tag.

## Setup

### 1. Create the repository

Click **Use this template** on GitHub, or:

```sh
gh repo create <owner>/<name> --template IgnatG/infraharvest-template --private --clone
```

### 2. Pick the cloud

The `CLOUD` variable (step 4) selects which cloud the workflows sign in to and import: `aws`, `azure` or `google`. One repository imports one cloud. For several clouds, make one repository per cloud.

### 3. Create the read-only identity, trusted by this repository

The workflows sign in with OIDC, using the job's GitHub ID token, so no cloud secret is stored. The identity needs read access only. The Terraform configurations in [IgnatG/infraharvest/permissions](https://github.com/IgnatG/infraharvest/tree/main/permissions) grant it: [`aws`](https://github.com/IgnatG/infraharvest/tree/main/permissions/aws), [`azure`](https://github.com/IgnatG/infraharvest/tree/main/permissions/azure) and [`google`](https://github.com/IgnatG/infraharvest/tree/main/permissions/google). Drift also reads the roots' state, which the read-only templates don't allow. The drift templates grant the same read access plus read access to the state alone: [`aws-drift`](https://github.com/IgnatG/infraharvest/tree/main/permissions/aws-drift), [`azure-drift`](https://github.com/IgnatG/infraharvest/tree/main/permissions/azure-drift) and [`google-drift`](https://github.com/IgnatG/infraharvest/tree/main/permissions/google-drift). [Permissions](https://ignatg.github.io/infraharvest/permissions/) describes what each one grants.

Scheduled runs present the token of the default branch, whose subject is `repo:<owner>/<name>:ref:refs/heads/main`. Trust that subject, or the whole repository. If you trust only that subject, a run started by hand on another branch can't sign in.

**AWS.** `permissions/aws` (the import role, `infraharvest-readonly`) and `permissions/aws-drift` (the drift role, `infraharvest-drift`) trust GitHub Actions through OIDC when you give them this repository's subject. The account needs GitHub's OIDC identity provider once; `create_github_oidc_provider=true` creates it in the first configuration you apply, and the other looks it up:

```sh
cd permissions/aws
terraform init
terraform apply \
  -var 'github_oidc_subjects=["repo:<owner>/<name>:ref:refs/heads/main"]' \
  -var create_github_oidc_provider=true   # only if the account has no GitHub OIDC provider yet

cd ../aws-drift
terraform init
terraform apply -var 'github_oidc_subjects=["repo:<owner>/<name>:ref:refs/heads/main"]'
```

Each prints its `role_arn`: set them as `AWS_ROLE_ARN` and `AWS_DRIFT_ROLE_ARN` (step 4). Sessions last an hour by default. A drift run must keep valid credentials until it ends. If an import or drift run takes longer than an hour, raise `max_session_duration` on the role and add `role-duration-seconds` to the workflow's sign-in step.

**Azure.** Create an app registration, or a user-assigned managed identity, with a federated credential for this repository:

```sh
az ad app federated-credential create --id <application (client) ID> --parameters '{
  "name": "infraharvest-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<name>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

Then apply `permissions/azure` with `principal_id` set to the object ID of its service principal, which assigns Reader. For drift, apply `permissions/azure-drift` with `assign_reader=false`, which assigns Storage Blob Data Reader on the state's container. The workflows sign in by setting `ARM_USE_OIDC` and the `ARM_*` IDs, which infraharvest, Terraform and the azurerm backend all read, so the Azure CLI isn't used.

**Google Cloud.** Create a workload identity pool and provider for GitHub, limited to this repository, and let it impersonate a service account:

```sh
gcloud iam workload-identity-pools create github --project=<project> --location=global
gcloud iam workload-identity-pools providers create-oidc infraharvest --project=<project> \
  --location=global --workload-identity-pool=github \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping=google.subject=assertion.sub,attribute.repository=assertion.repository \
  --attribute-condition="assertion.repository == '<owner>/<name>'"
gcloud iam service-accounts create infraharvest --project=<project>
gcloud iam service-accounts add-iam-policy-binding infraharvest@<project>.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/projects/<project number>/locations/global/workloadIdentityPools/github/attribute.repository/<owner>/<name>"
```

Then apply `permissions/google` with `member=serviceAccount:infraharvest@<project>.iam.gserviceaccount.com`. For drift, apply `permissions/google-drift` with `grant_read_only=false`. The project needs the Cloud Asset API (`cloudasset.googleapis.com`) enabled.

For drift, also set a `backend` in [`infraharvest.yaml`](infraharvest.yaml) before the first import. Without a backend, the roots keep local state, which isn't committed, so drift has no state to compare against.

### 4. Set the repository variables

Under **Settings → Secrets and variables → Actions → Variables**, or with `gh variable set <NAME> --body <value>`:

| Variable | For | Value |
|---|---|---|
| `INFRAHARVEST_VERSION` | all | The infraharvest release to install, such as `v0.1.0`. No release exists yet (see above). |
| `CLOUD` | all | `aws`, `azure` or `google` |
| `SERVICES` | import | Services to import, comma-separated. For AWS, such as `vpc,subnet,sg,s3,iam`. For Azure, `resource_graph`. For Google Cloud, `cloud_asset` or `*`. `infraharvest services <provider>` lists them. |
| `REGIONS` | AWS, Google Cloud | Comma-separated. For AWS, such as `eu-west-2,us-east-1`; by default, the region the sign-in uses. For Google Cloud, `global` plus your regions, such as `global,europe-west2`; by default, `global`. Azure takes none: leave it unset. |
| `AWS_ROLE_ARN` | AWS | The import role's ARN |
| `AWS_DRIFT_ROLE_ARN` | AWS, drift | The drift role's ARN. If unset, drift uses `AWS_ROLE_ARN`. |
| `AWS_REGION` | AWS, optional | The region to sign in to. Default `us-east-1`. |
| `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | Azure | The identity's client ID, its tenant, and the subscription to import |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Google Cloud | `projects/<project number>/locations/global/workloadIdentityPools/github/providers/infraharvest` |
| `GCP_SERVICE_ACCOUNT` | Google Cloud | `infraharvest@<project>.iam.gserviceaccount.com` |
| `GCP_PROJECT` | Google Cloud | The project ID to import, passed as `--projects` |
| `MANAGED_STATE` | optional | Leave it empty at first. Set it to `backend` once the roots are applied and their `imports.tf` deleted (see [Next](#next)). |
| `INFRAHARVEST_APP_CLIENT_ID` | optional | A GitHub App's client ID (step 6) |

### 5. Let workflows open pull requests

Under **Settings → Actions → General → Workflow permissions**, turn on **Allow GitHub Actions to create and approve pull requests**. New repositories have it off, and with the default `github.token`, opening the pull request fails without it. If the repository belongs to an organisation, the organisation must allow it too.

### 6. Optional: a GitHub App, so that checks run on the pull requests

GitHub starts no workflows for a pull request that the default `github.token` opens. So your checks, such as a `terraform plan` or tflint, never run on the import and drift pull requests, and required checks leave those pull requests unmergeable (see [Checks on the pull request](https://github.com/IgnatG/infraharvest/blob/main/action/README.md#checks-on-the-pull-request)). To have checks run:

1. Create a GitHub App with **Contents** and **Pull requests** read and write permissions, and install it on this repository.
2. Set the variable `INFRAHARVEST_APP_CLIENT_ID` to its client ID.
3. Set the secret `INFRAHARVEST_APP_PRIVATE_KEY` to its private key.

The workflows then open the pull requests with the app's token. The token lasts an hour, so an import that runs longer than that can't push with it.

### 7. Run the import

Under **Actions → import → Run workflow**, or:

```sh
gh workflow run import.yml
```

The job summary shows the import report. When the run finishes, a pull request titled "Import existing resources with infraharvest" is waiting.

## Next

1. **Review** the pull request. The body is the import report. Each root under `infraharvest-out/` has a `README.md` that lists what it imports and the steps left, such as secret variables to set.
2. **Apply.** For each root, run `terraform init`, then `terraform plan`, which imports every resource and changes nothing, then `terraform apply`, which records the resources in the state. Merge the pull request.
3. **Delete each root's `imports.tf`** in a follow-up commit, since the resources are now in the state. Then set the `MANAGED_STATE` variable to `backend`, so that the weekly import reads the state to find what the roots already manage. On AWS, `AWS_ROLE_ARN` then needs state read access too: point it at the drift role.
4. **Drift** now runs daily. Its pull request accepts nothing. It records each change in `infraharvest-out/drift.yaml` with `decision: undecided`. Merge that pull request, then set each entry's `decision` to `accept` or `keep`. To write the accepted values into the code, run with `drift: accept` and `drift-decisions: infraharvest-out/drift.yaml`, or with rules in `drift-accept` (see the comment in `drift.yml`, and [Drift](https://github.com/IgnatG/infraharvest/blob/main/action/README.md#drift)).
5. **Your apply workflow**, if you have one, should use the concurrency group `terraform-apply`, which `import.yml` and `drift.yml` use too. Drift and an import with `MANAGED_STATE` read the state without locking it, so they shouldn't run while an apply writes it.

**Not ready to import?** Run an [audit](https://ignatg.github.io/infraharvest/audit/) first, which is the low-risk option. With `infraharvest discover`, it lists what in the account isn't under Terraform yet, using the same read-only access, and it writes nothing you need to keep.

## Committing straight to a branch

To have the import commit its output and push it to a branch, with no pull request, change the Action's inputs in `import.yml`:

```yaml
          # instead of pull-request and pull-request-token:
          commit: "true"
          commit-branch: infraharvest/imported
```

`pull-requests: write` can then go from `permissions`. The push never forces, so the second run is rejected if the branch has commits that the checkout doesn't. Either check out that branch (`ref: infraharvest/imported` on `actions/checkout`, once the branch exists), or push to the branch the job checks out by leaving `commit-branch` out. Branch protection still applies: a push to `main` that requires pull requests is rejected, unless the token's app may bypass the rule. See [Committing straight to a branch](https://github.com/IgnatG/infraharvest/blob/main/action/README.md#committing-straight-to-a-branch).

## Several environments

With `environments:` in [`infraharvest.yaml`](infraharvest.yaml), each environment gets its own roots. The roots of each stack then merge into one root that the environments share, with what differs in `vars/<env>/` (see [Merging environments](https://ignatg.github.io/infraharvest/configuration/#merging-environments)).

- **Import** stays one job, because merging needs every environment in one run. Its credentials must reach every environment. On AWS, put `accounts` and `assume-role` under `providers.aws` in `infraharvest.yaml`, and use a role that can assume the role in each account.
- **Drift** checks a shared root once per environment, each time with that environment's credentials. With the job's credentials, the other environments' resources would look deleted, so the Action refuses shared roots unless `environment` is set. Replace the job in `drift.yml` with a matrix that has one entry per environment and its own role and decisions file. Each entry opens its own pull request, on `infraharvest/drift-<env>`. On AWS:

```yaml
jobs:
  drift:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - environment: dev
            role: arn:aws:iam::111111111111:role/infraharvest-drift
          - environment: prod
            role: arn:aws:iam::222222222222:role/infraharvest-drift
    steps:
      # the checkout and the app token as before, then:
      - uses: aws-actions/configure-aws-credentials@e1253824e5c10ff9df46874f81ed3ec929e19cfd # v6.3.0
        with:
          role-to-assume: ${{ matrix.role }}
          aws-region: ${{ vars.AWS_REGION || 'us-east-1' }}
      - uses: IgnatG/infraharvest/action@<the same commit> # main
        with:
          version: ${{ vars.INFRAHARVEST_VERSION }}
          import: "false"
          drift: report
          environment: ${{ matrix.environment }}
          drift-decisions: decisions/drift-${{ matrix.environment }}.yaml
          pull-request: "true"
          pull-request-token: ${{ steps.app-token.outputs.token || github.token }}
```

If one account, subscription or project holds every environment, split by tags, keep the single job and set `environment: all`. See [Roots environments share](https://github.com/IgnatG/infraharvest/blob/main/action/README.md#roots-environments-share).

## Keeping it up to date

Dependabot (`.github/dependabot.yml`) opens a weekly pull request for actions that have a new release. Every action is pinned to a full commit SHA, with the release in a comment. Update the infraharvest Action pin and `INFRAHARVEST_VERSION` yourself.

## License

[MIT No Attribution](LICENSE) (MIT-0). You can copy, change and use what this template gives you without keeping the copyright notice, so a repository made from it carries no license obligation. infraharvest itself is licensed separately.
