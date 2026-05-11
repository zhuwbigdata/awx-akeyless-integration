# Akeyless-to-AWX Credential Sync

Syncs AWS credentials from Akeyless into AWX as standard "Amazon Web Services" credentials. Runbooks use the resulting credential with zero code changes.

## How it works

1. A scheduled AWX job runs the sync playbook inside the `akeyless-awx-ee` execution environment
2. The playbook authenticates to Akeyless using the existing Akeyless custom credential (cert auth or k8s auth)
3. Fetches the AWS secret (static, dynamic, or rotated)
4. Creates/updates an AWX credential of type "Amazon Web Services"
5. Any job template referencing this credential gets fresh AWS creds automatically

## Prerequisites

- AWX 24.6.1+ with the Akeyless custom credential type registered
- Akeyless credential created in AWX (cert auth or k8s auth)
- `akeyless-awx-ee` execution environment registered in AWX
- Secrets in Akeyless containing AWS credential data

## Quick start

### 1. Push this repo to your Git server

```bash
git remote add origin <your-git-url>
git push -u origin main
```

### 2. Run the setup playbook

```bash
export CONTROLLER_HOST="https://your-awx.example.com"
export CONTROLLER_USERNAME="admin"
export CONTROLLER_PASSWORD="your-password"

ansible-playbook playbooks/setup_awx_resources.yml
```

This creates the AWX project, AAP credential, job template, and 15-minute sync schedule.

### 3. Configure the secret mapping

Set extra_vars on the job template:

| Variable | Default | Description |
|----------|---------|-------------|
| `AKEYLESS_AWS_SECRET_NAME` | `/devops/test-user-password-as-generic` | Akeyless secret path |
| `AKEYLESS_SECRET_TYPE` | `static` | `static`, `dynamic`, or `rotated` |
| `AWS_CREDENTIAL_NAME` | `aws-creds-from-akeyless` | Name for the AWX credential |
| `AWS_CREDENTIAL_ORG` | `Default` | AWX organization |
| `AWS_ACCESS_KEY_FIELD` | `access_key` | JSON key for access key in static secret |
| `AWS_SECRET_KEY_FIELD` | `secret_key` | JSON key for secret key in static secret |
| `AWS_TOKEN_FIELD` | `security_token` | JSON key for STS token (optional) |

### 4. Launch the sync job

Either wait for the schedule or launch manually in AWX.

## Secret format

### Static secrets

Store as a JSON object in Akeyless:

```json
{
  "access_key": "AKIAIOSFODNN7EXAMPLE",
  "secret_key": "wJalrXUtnFEMI/K7MDENG",
  "security_token": ""
}
```

### Dynamic secrets

AWS dynamic secrets return temporary credentials automatically. The playbook handles the response format (`access_key_id`, `secret_access_key`, `security_token`).

## Architecture

```
AWX Schedule (15 min)
    |
    v
Job Template: "Sync AWS Credentials from Akeyless"
    |- Credential: akeyless_cert_auth_credential (AKEYLESS_* env vars)
    |- Credential: AWX Self-Referential (CONTROLLER_* env vars)
    |- EE: akeyless-awx-ee
    |
    v
Playbook: sync_aws_credential.yml
    |- Authenticate to Akeyless
    |- Fetch AWS secret
    |- Create/update AWX "Amazon Web Services" credential
    |
    v
AWS Credential: "aws-creds-from-akeyless"
    |
    v
Customer job templates use this credential as usual
```
