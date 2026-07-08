# GitHub Actions OIDC Lab

This lab builds the smallest safe baseline for studying identity propagation from GitHub Actions into cloud providers. The workflows are manual-only and perform read-only identity checks.

## Workflows

- `.github/workflows/oidc-claims.yml` decodes selected GitHub OIDC claims without printing the raw JWT.
- `.github/workflows/aws-oidc.yml` exchanges the GitHub OIDC token for AWS role credentials and runs `aws sts get-caller-identity`.
- `.github/workflows/azure-oidc.yml` exchanges the GitHub OIDC token through `azure/login` and runs `az account show`.
- `.github/workflows/gcp-oidc.yml` exchanges the GitHub OIDC token through Workload Identity Federation and runs `gcloud auth list`.

## GitHub Repository Variables

Configure these as repository variables, not secrets, because they are identifiers rather than private credentials.

AWS:

- `AWS_ROLE_ARN`
- `AWS_REGION`

Azure:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

GCP:

- `GCP_WORKLOAD_IDENTITY_PROVIDER`
- `GCP_SERVICE_ACCOUNT`

## Minimum Cloud-Side Trust Constraints

Start with the narrowest possible trust binding:

- Bind only this repository.
- Bind only a specific branch or protected environment when possible.
- Set the expected audience.
- Grant read-only cloud permissions for the first run.

The first research artifact should be a table of which trust policies accept or reject each GitHub OIDC `sub` shape. After the baseline succeeds, clone each policy into safer and weaker variants to measure which identity propagation edges become reachable.

## Claims to Record

For each run, record:

- `iss`
- `aud`
- `sub`
- `repository`
- `repository_owner`
- `ref`
- `workflow`
- `job_workflow_ref`
- `actor`
- `event_name`

These fields are the bridge between GitHub-side trust and cloud-side trust. They will become the detector evidence for identity propagation risk chains.
