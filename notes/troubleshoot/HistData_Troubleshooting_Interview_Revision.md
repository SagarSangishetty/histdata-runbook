# HistData deployment troubleshooting

Interview preparation and revision notes • 27 September 2026

## 1. What happened in simple terms

During the HistData dev/prod workflow setup, several independent configuration problems surfaced. Production image promotion could not read dev ECR. Later, an installation-script error left the External Secrets controller without AWS credentials. Separately, unresolved Git conflict markers were committed into Helm values, preventing Argo CD from rendering the application.

These were different failures at different stages. Fixing ECR did not fix Kubernetes secrets, and fixing External Secrets could not repair invalid YAML in Git.

**Status at the end of the recorded conversation:** the working ECR repository policy was imported into Terraform, and the user confirmed a no-change plan. The Helm-values fix and an intentional ACM certificate update were reported merged. Restarting External Secrets and refreshing Argo CD were recommended, but final outputs confirming store readiness, Secret creation, healthy pods, and a working HTTPS application were not supplied. Do not describe this session as a fully verified recovery yet.

This document records the 27 September troubleshooting session. The attached 26 September handover supplies background; its earlier stopping point does not describe the later work completed here.

## 2. Architecture and workflow to remember

| Component | Responsibility |
| --- | --- |
| `histdata-infra-terraform` | AWS infrastructure, IAM, ECR and remote Terraform state |
| `histdata-app` | Application build, dev image publication and prod image promotion |
| `histdata-k8s-deployments` | Helm chart, environment values and Argo CD configuration |
| Dev AWS account | `935776475838` |
| Prod AWS account | `708553018735` |
| AWS region | `us-east-1` |
| ECR repository | `histdata-app` in both accounts |
| External Secrets Operator (ESO) | Reads AWS Secrets Manager and creates Kubernetes Secrets |
| Argo CD | Renders Git configuration and reconciles Kubernetes resources |

The intended delivery flow was:

1. Merge application changes into `main`.
2. Dev CI builds an image and publishes it to dev ECR with the application commit SHA as its tag.
3. CI opens a GitOps PR updating dev Helm values. Once merged, Argo CD reconciles dev.
4. Test the application in dev. Additional fixes produce new images and new dev values changes.
5. Start prod promotion. The workflow selects the image tag from merged dev values and captures its digest; it does not simply choose the newest ECR tag.
6. With the prod promotion role, verify the source digest and copy that artifact into prod ECR. Verify the destination digest.
7. Update prod values through a GitOps PR. After merge, prod Argo CD reconciles the desired application configuration.

The image is built once. A digest identifies its content; a commit-SHA tag links it to source history. Selecting a tag from dev values does not itself prove that the image passed functional testing or is currently healthy in the cluster.

## 3. Incident: cross-account image promotion denied

### Symptom and evidence

The prod workflow assumed `histdata-prod-github-actions` in account `708553018735`, then received:

```text
ecr:BatchGetImage ... because no resource-based policy allows the action
```

The target was the dev repository:

```text
arn:aws:ecr:us-east-1:935776475838:repository/histdata-app
```

### Why dev needed a policy

The prod role is the caller, but dev owns the source image. Cross-account authorization requires permission on both sides:

| Side | Required permission |
| --- | --- |
| Prod role identity policy | Read the specified dev repository |
| Dev ECR repository policy | Accept read requests from the specified prod role |
| Prod role identity policy | Push to the prod repository |
| Caller identity policy | `ecr:GetAuthorizationToken` with resource `*` for registry authentication |

The three source-repository read actions used were `ecr:BatchCheckLayerAvailability`, `ecr:BatchGetImage`, and `ecr:GetDownloadUrlForLayer`.

Logging into dev ECR with prod credentials does not change the caller into a dev role. The request still originates from the prod role and requires cross-account access.

### A second failure: ECR rejected the proposed policy

Terraform returned `Invalid repository policy provided`. A policy using the prod account-root principal with an exact `aws:PrincipalArn` condition also failed.

We checked:

```bash
aws sts get-caller-identity --query Account --output text
aws iam get-role --role-name histdata-prod-github-actions \
  --query Role.Arn --output text
```

The outputs confirmed the prod account and exact IAM role ARN. IAM Access Analyzer validation returned `[]`, meaning it found no issues in its checks. That was not proof that ECR would accept the policy.

After switching to dev credentials, `get-repository-policy` confirmed no existing repository policy. Submitting the same policy directly through the CLI produced the same rejection. This established that the failure was not specific to Terraform.

### Accepted policy and reconciliation

ECR accepted a minimal policy with the verified role ARN directly as principal and no explicit `Resource` or `Condition`:

```hcl
resource "aws_ecr_repository_policy" "prod_promotion_pull" {
  repository = "histdata-app"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "AllowProdPromotionRoleToPull"
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::708553018735:role/histdata-prod-github-actions"
      }
      Action = [
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ]
    }]
  })

  depends_on = [module.ecr]
}
```

The manually attached policy was then brought under Terraform management using the dev backend/workspace:

```bash
terraform import -var-file=dev.tfvars \
  aws_ecr_repository_policy.prod_promotion_pull histdata-app
terraform plan -var-file=dev.tfvars
```

The user confirmed no changes in the plan.

**Evidence limit:** both principal representation and the `Resource` field changed in the successful test. We did not isolate which difference caused acceptance. AWS documentation includes ECR examples with `Resource: "*"`; therefore, do not claim that field is universally invalid. Final image-promotion success was not shown in this conversation.

**Interview answer:** “I traced a promotion denial to missing cross-account repository access. I verified the caller, role ARN, repository and policy, reproduced a subsequent policy rejection outside Terraform, then applied a minimal role-scoped policy that ECR accepted. I imported it into Terraform and verified there was no drift.”

## 4. Incident: GitOps promotion job skipped

The original workflow had an optional `create_gitops_pr` input, defaulting to false. The image-copy job could succeed while the GitOps PR job was intentionally skipped.

If the input is removed but this condition remains, the job can still skip:

```yaml
if: ${{ inputs.create_gitops_pr }}
```

If GitOps promotion should always follow a successful image copy, remove the obsolete input condition and retain the required job dependencies. If the job uses outputs from multiple jobs, preserve all required `needs` entries. Also check whether an upstream job failed or skipped.

Start a new workflow run after committing a workflow change; rerunning an old run uses that run’s original revision. The user said the input was removed, but the final condition and run result were not independently verified.

**Interview lesson:** investigate job conditions, dispatch inputs and dependencies before debugging the job’s individual steps.

## 5. Incident: Helm installed ESO, then the script failed

### Symptom

```text
external-secrets has been deployed successfully ...
./install.sh: line 33: --set-string: command not found
```

### Confirmed cause

In `platform-addons/install.sh`, this line lacked a trailing backslash:

```bash
  --set certController.enablePartialCache=false
```

Bash ended the Helm command there. Helm ran successfully without the following service-account annotation argument. Bash then tried to execute `--set-string` as a new command.

Correct form:

```bash
  --set certController.enablePartialCache=false \
  --set-string "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn=${EXTERNAL_SECRETS_ROLE_ARN}"
```

The script uses `set -euo pipefail`, so the failure also prevented its later rollout checks from running. Rerunning the corrected `helm upgrade --install` applies the intended annotation, but a changed service-account annotation alone does not recreate existing pods.

**Interview lesson:** successful output from one command does not mean the complete installation script succeeded. Also, `bash -n` alone would not catch this problem: executing `--set-string` as a command is syntactically valid shell.

## 6. Main Kubernetes incident: app pods could not start

### Symptoms

Two dev application pods stayed at `0/1` with `CreateContainerConfigError`. Application logs were unavailable because the container had not started.

The pod event identified the immediate blocker:

```text
Error: secret "histdata-db" not found
```

The image was already present on the node, so this particular failure was not an image-pull issue.

The ExternalSecret reported:

```text
ClusterSecretStore "aws-secrets-manager" is not ready
```

The store reported `InvalidProviderConfig` and:

```text
failed to refresh cached credentials, no EC2 IMDS role found
```

### Diagnosis, from symptom to dependency

1. Confirmed the Kubernetes context was `histdata-dev-eks` in the dev account.
2. Inspected pod events because application logs were unavailable.
3. Followed the missing Kubernetes Secret back to the ExternalSecret.
4. Followed the ExternalSecret failure back to the ClusterSecretStore.
5. Checked credentials on the ESO controller, not just on the application pod.

The ESO service account had the correct annotation:

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::935776475838:role/histdata-dev-external-secrets
```

However, the existing controller pod showed `Environment: <none>` and only its regular Kubernetes API token mount. It had no `AWS_ROLE_ARN`, no `AWS_WEB_IDENTITY_TOKEN_FILE`, and no IRSA token mount.

This was consistent with the controller having been created before the annotation was corrected. The annotation was now present, but the running pod had not received IRSA injection. Without usable credentials, the AWS SDK fell back to EC2 instance metadata and failed.

### Fix recommended

Recreate the controller pods through the Deployment:

```bash
kubectl rollout restart deployment/external-secrets -n external-secrets
kubectl rollout status deployment/external-secrets \
  -n external-secrets --timeout=180s
```

New pods should receive the role environment variables and web-identity token at creation. This should enable credential acquisition, provided the IAM trust, OIDC provider, role permissions and network access are correct.

Check the complete dependency chain:

```bash
kubectl describe pods -n external-secrets \
  -l app.kubernetes.io/name=external-secrets
kubectl get clustersecretstore aws-secrets-manager
kubectl get externalsecret -n histdata
kubectl get secret histdata-db -n histdata
kubectl get pods -n histdata
```

No secret-value output is needed. Once the Secret exists, kubelet should retry starting the waiting application containers.

**Important distinctions:** the application’s IRSA role is separate from ESO’s role; a running controller can still fail reconciliation; and an IAM credential error is different from an authenticated `AccessDenied` response. If restart does not restore access, inspect new controller logs, role trust and current EKS OIDC issuer before changing permissions.

**Interview answer:** “I started with pod events because the container never started. They showed a missing database Secret. I traced that to an unready External Secrets store and found that the controller pod lacked IRSA credentials even though its service account was annotated. The earlier install script had omitted that annotation, and the existing pod needed recreation to receive it. I recommended a rollout restart and end-to-end checks of the store, Secret and workload.”

## 7. Incident: Git divergence and incomplete merge resolution

The local feature branch and its remote had different commits. Pull required an explicit merge/rebase strategy, and push was rejected as non-fast-forward.

The selected approach preserved history with a merge:

```bash
git pull --no-rebase origin feature/adding-prod-vaules
```

A conflict appeared in dev values. One side used an IAM role ARN for `externalSecret.remoteSecretArn`; the other used the correct Secrets Manager secret ARN. The secret field must identify the secret to retrieve, not the role used to authorize retrieval.

After resolution, `git add` stages the file, and `git commit` completes the merge. Starting a second merge before committing caused `MERGE_HEAD exists`. Pressing Ctrl+C did not clear the pending merge state.

**Critical mistake discovered later:** conflict markers were still committed. Git accepts a staged file as resolved; it does not guarantee that its contents are valid YAML.

| Command | What it checks |
| --- | --- |
| `git diff` | Unstaged edits |
| `git diff --cached` | Staged edits that will be committed |
| `git diff --cached --check` | Whitespace errors and introduced conflict markers in staged changes |
| `helm template` | Whether the selected chart and values render successfully |

After staging, an empty plain `git diff` does not mean there are no changes. A clean working tree does not mean the committed files are valid.

## 8. Incident: Argo CD could not render Helm manifests

### Symptom and confirmed cause

Argo CD reported a cached manifest-generation error:

```text
failed to parse environments/dev/values.yaml
yaml: line 14: could not find expected ':'
```

The current GitHub `main` file contained `<<<<<<< HEAD`, `=======`, and `>>>>>>> ...` markers. This blocked rendering before Argo CD could apply resources. Refreshing Argo CD alone could not repair invalid source YAML.

### Why the first local Helm check passed

The local checkout was an older valid commit:

```text
Local HEAD:   cd84159...
origin/main:  7e69132...
```

After fetching and fast-forwarding to GitHub `main`, the same Helm command reproduced Argo CD’s error. This established that the earlier test validated different source content.

```bash
git fetch origin
git rev-parse HEAD origin/main
git pull --ff-only origin main
helm template histdata ./charts/histdata \
  --namespace histdata -f environments/dev/values.yaml > /dev/null
```

### Fix and checks

A new branch, `fix/dev-values-conflict-markers`, removed all markers and the incorrect IAM role entry. The valid section became:

```yaml
externalSecret:
  remoteSecretArn: arn:aws:secretsmanager:us-east-1:935776475838:secret:histdata-dev/oracle/application-AFRfIt
```

The user also intentionally updated the dev ACM certificate ARN. The staged diff exposed this additional change, which was explicitly confirmed rather than overlooked. Helm rendering validates configuration syntax/templates, not certificate existence, issuance or hostname coverage.

The local Helm render passed, and the user reported merging the fix. A hard refresh was then recommended:

```bash
kubectl annotate application histdata-dev -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
kubectl get application histdata-dev -n argocd
```

This refresh requests a new comparison/render and invalidates relevant caches. It is not proof that the application is synced or healthy; inspect the resulting status and revision.

**Interview answer:** “Argo CD failed before deployment because committed conflict markers made the Helm values invalid. My first local render passed because my checkout was stale. Comparing commit hashes revealed that mismatch. After updating to the failing revision, I reproduced the error, removed the markers, retained the correct secret reference, validated the chart and merged the fix.”

## 9. Recovery acceptance criteria — still to verify

| Check | Required evidence |
| --- | --- |
| Correct environment | AWS identity and kubectl context match the intended account/cluster |
| Git source | Argo CD uses the merged fix revision |
| Manifest generation | No YAML parsing or Helm rendering errors |
| ESO credentials | New controller pod has intended role and web-identity token configuration |
| Store | `aws-secrets-manager` reports Ready=True |
| ExternalSecret | Successful synchronization status |
| Kubernetes Secret | `histdata-db` exists with required keys; do not print values |
| Application | Desired replicas ready; no config errors or repeated crashes |
| Argo CD | Synced and Healthy at the intended revision |
| Public service | ALB targets healthy, DNS correct, valid TLS and functional application smoke test |
| Prod promotion | Source and destination digest verification succeeds; resulting GitOps change reviewed |

A rollout completing for ESO is only one checkpoint. Database credentials, connectivity, certificate validity or application configuration could surface as subsequent failures once the original blockers are removed.

## 10. Prevention improvements

These are recommendations, not claims of completed implementation.

1. Make Helm lint/render checks for both dev and prod required before merging, and verify branch protection actually enforces them.
2. Add a tracked-file conflict-marker check to CI. Scope it to relevant configuration paths so documentation examples do not trigger false positives.
3. Review `git diff --cached` before committing; validate the exact branch/revision that will be deployed.
4. Prefer a Helm values file for lengthy add-on options and annotations, reducing fragile shell continuations. Pin chart versions for repeatability.
5. Include service-account annotation and controller credential checks in bootstrap verification. Recreate pods when required after IRSA configuration changes.
6. Preserve least privilege: separate app, ESO, CI and promotion roles; grant prod source-image reads only to the intended repository.
7. Import manually created Terraform-managed resources and confirm a clean plan against the correct remote state.
8. Keep operational error messages accurate: registry authentication/authorization failure should not be reported as an image digest mismatch.
9. Verify account identity, Kubernetes context and current resource outputs after environment recreation. Folders do not select AWS accounts.
10. Do not treat “Synced,” “Running,” a clean Git tree, or successful syntax validation as standalone evidence of end-to-end health.

## 11. Interview revision questions

**Why did the prod role need dev ECR access?** To read the selected source manifest/digest and download the layers for promotion. Dev owns the resource, so its repository policy must accept the cross-account caller.

**Does digest verification test the application?** No. It verifies artifact identity. Functional testing is a separate activity.

**What is the difference between a role ARN and a secret ARN?** A role defines an assumable AWS identity and permissions. A secret ARN identifies stored data. In this setup, ESO’s role is attached to its service account; the secret ARN belongs in the ExternalSecret reference.

**Why were there no application logs?** The container was waiting for configuration and never started. Pod events identified the missing Secret.

**Why was restarting ESO justified?** Its service account was annotated, but the existing pod lacked IRSA environment variables and token mount. Recreating it allows admission-time injection to run.

**Why did the app’s AWS token not prove ESO had access?** They are different pods with different service accounts and roles.

**Why did Argo CD fail while a local Helm command passed?** The local checkout was older. Testing the same revision reproduced the failure.

**Why did Git allow conflict markers to be committed?** Staging a file marks it resolved in Git’s index; Git does not validate its application semantics.

**What should you say about the final outcome?** “The ECR policy and Terraform reconciliation were verified, and the YAML fix was validated and reported merged. Final cluster and application recovery checks were pending in the recorded session.”

## 12. A concise interview story

“In my HistData DevOps lab, I investigated failures across image promotion and GitOps deployment. I separated the problems by stage instead of treating them as one outage. For promotion, I corrected cross-account ECR access and reconciled the working policy into Terraform. For Kubernetes, pod events led me from a missing database Secret to an External Secrets controller without IRSA credentials; a shell continuation error had omitted its service-account annotation during installation. Separately, Argo CD could not render Helm values because merge-conflict markers were committed. I compared Git revisions, reproduced the error locally, validated the corrected values and merged the fix. The main lesson was to verify the full dependency chain and test the exact revision being deployed. Final service-health checks remained pending in the session record.”

## 13. Reference material

Session evidence: user-supplied command output, inspected repository files and the 26 September HistData handover. The narrative distinguishes observed results from recommendations and pending checks.

- [AWS ECR repository policy examples](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-policy-examples.html)
- [AWS ECR set-repository-policy CLI](https://docs.aws.amazon.com/cli/latest/reference/ecr/set-repository-policy.html)
- [AWS cross-account ECR access](https://repost.aws/knowledge-center/secondary-account-access-ecr)
- [IAM Access Analyzer policy validation](https://docs.aws.amazon.com/cli/latest/reference/accessanalyzer/validate-policy.html)
- [Terraform ECR repository policy and import](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ecr_repository_policy)
- [External Secrets AWS Secrets Manager provider](https://external-secrets.io/latest/provider/aws-secrets-manager/)

Identifiers in commands document this lab. Verify current ARNs and contexts before reusing commands; do not copy passwords, tokens or secret values into interview notes.
