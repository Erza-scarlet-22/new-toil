# Stack Update Steps
## Upgrading Your Existing Deployed Stack

---

## What This Update Adds to Your Stack

Your existing stack has: 3 ECS services (app, dashboard, chatbot),
log processor Lambda, CodeBuild (3 images), scheduler, EFS, ALB, S3, Secrets.

This update adds on top without touching any existing resource:

| What is added | Why |
|---|---|
| `EcrDummyAppRepo` | ECR repo for the new dummy app Docker image |
| `ServiceNowSecret` | Secrets Manager secret for ServiceNow credentials |
| `SSLCertSecret` | Secrets Manager secret where SSL Lambda stores cert ARN |
| `AlbSecurityGroup` port 443 | Opens HTTPS port on ALB (safe — no traffic until listener added) |
| `EcsTaskSecurityGroup` port 5001 | Allows ALB to reach dummy app container |
| `DummyAppTargetGroup` | ALB target group for dummy app |
| `ListenerRuleDummyAppRoot/Api` | Routes `/dummy` and `/api/dummy` to dummy app |
| `ActionLambdaRole` | New IAM role for the 5 action Lambdas |
| `DummyAppTaskRole` | New IAM role for dummy app ECS task |
| `EcsTaskRole` ReadServiceNowSecret | New policy on existing role (allows dashboard to poll ticket state) |
| `CodeBuildRole` action Lambda permissions | Allows CodeBuild to update the 5 new Lambdas |
| Log groups (7 new) | CloudWatch log groups for dummy app + 5 action Lambdas |
| `DummyAppTaskDefinition` | ECS task definition for dummy app |
| `DummyAppService` | ECS service running the dummy app |
| `ServiceNowLambda` | Bedrock action: createIncident / getTicketStatus / updateTicket |
| `SSLRemediationLambda` | Bedrock action: SSL cert remediation (demo mode by default) |
| `PasswordResetLambda` | Bedrock action: password rotation |
| `DBRemediationLambda` | Bedrock action: RDS storage/upgrade |
| `ComputeRemediationLambda` | Bedrock action: ECS scale-up |
| 5× Bedrock Lambda permissions | Allow Bedrock agent to invoke each action Lambda |
| `CodeBuildProject` updated buildspec | Builds dummy-app image, packages + deploys all 6 Lambdas |
| New Outputs | DummyApp URL, Lambda ARNs, ALB ARN, secret ARNs |

**Nothing existing is deleted or renamed.**
All original logical IDs, resource names, and parameter names are preserved.

---

## STEP 1 — Update Your Source ZIP

Add the new files to your project before uploading:

```
Your project must contain these files for the new CodeBuild buildspec:

lambda/servicenow_lambda_handler.py   ← from SERVICENOW_SETUP output
lambda/ssl_lambda_handler.py          ← from ssl_lambda_handler_final output
lambda/password_reset_lambda_handler.py  ← original file (already in your project)
lambda/db_lambda_handler.py           ← original file (already in your project)
lambda/compute_lambda_handler.py      ← original file (already in your project)
Dummy-infra-app/dummy_app.py          ← from dummy_app.py output
Dummy-infra-app/static/index.html     ← from index_final.html output
dockerfiles/Dockerfile.dummy-app      ← original file (already in your project)
Dashboard/dashboard_blueprint.py      ← from dashboard_blueprint.py output
Dashboard/templates/dashboard.html    ← apply the 5-change patch from STEPS.md
orchestrator_agent_prompt.txt         ← from orchestrator_agent_prompt.txt output
```

Once all files are in place:

```bash
# From your project root
zip -r log-aggregator.zip . \
  --exclude "*.pyc" \
  --exclude "__pycache__/*" \
  --exclude ".git/*" \
  --exclude "*.zip"

# Upload to your existing artifact bucket
# (get the name from your stack Outputs → ArtifactBucketName)
aws s3 cp log-aggregator.zip \
  s3://YOUR_ARTIFACT_BUCKET_NAME/log-aggregator.zip \
  --region us-east-1
```

---

## STEP 2 — Upload the New Template to S3

CloudFormation stacks over ~50KB must be uploaded to S3 first.
The upgraded template is larger than before so use S3.

```bash
aws s3 cp template_upgraded.yaml \
  s3://YOUR_ARTIFACT_BUCKET_NAME/template_upgraded.yaml \
  --region us-east-1
```

Note the S3 URL — you will paste it in Step 3.
Format: `https://YOUR_ARTIFACT_BUCKET_NAME.s3.amazonaws.com/template_upgraded.yaml`

---

## STEP 3 — Update the Stack via CloudFormation Console

1. Open **CloudFormation → Stacks → your stack name**
   (e.g. `log-aggregator-prod`)

2. Click **Update**

3. Select **Replace existing template**

4. Select **Amazon S3 URL** and paste:
   ```
   https://YOUR_ARTIFACT_BUCKET_NAME.s3.amazonaws.com/template_upgraded.yaml
   ```

5. Click **Next**

6. On the **Parameters** page — most values are already filled in from your existing stack.
   You only need to fill in the **new parameters** (all have safe defaults):

| Parameter | Recommended value | Note |
|---|---|---|
| `DummyAppDesiredCount` | `1` | Leave as-is |
| `DummyAppServiceName` | `dummy-infra-app` | Leave as-is |
| `ServiceNowSecretName` | `servicenow/credentials` | Leave as-is |
| `RdsDbInstanceId` | *(leave blank)* | Only needed for DB Lambda in real mode |
| `DomainName` | *(leave blank)* | Leave blank for demo |
| `HostedZoneId` | *(leave blank)* | Leave blank for demo |
| `AcmCertificateArn` | *(leave blank)* | Leave blank for demo |

   All other parameters (EnvironmentName, VpcId, subnets, S3SourceBucket, etc.)
   keep their existing values — CloudFormation pre-fills them.

7. Click **Next**

8. On **Configure stack options** — leave everything as-is, click **Next**

9. On **Review** page:
   - Check **I acknowledge that AWS CloudFormation might create IAM resources with custom names**
   - Review the **Change set preview** — you should see only ADD operations, no REMOVE or MODIFY on existing critical resources

10. Click **Submit**

11. **Wait for UPDATE_COMPLETE** — takes approximately 5–8 minutes.

---

## STEP 4 — Verify the Update Succeeded

```bash
STACK_NAME="log-aggregator-prod"   # replace with your stack name
REGION="us-east-1"

# Check stack status
aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query 'Stacks[0].StackStatus' \
  --output text \
  --region $REGION
# Expected: UPDATE_COMPLETE

# List new resources that were created
aws cloudformation describe-stack-resources \
  --stack-name $STACK_NAME \
  --region $REGION \
  --query 'StackResources[?ResourceStatus==`CREATE_COMPLETE`].{Type:ResourceType,ID:LogicalResourceId}' \
  --output table
```

You should see these in the CREATE_COMPLETE list:
- `AWS::ECR::Repository` — EcrDummyAppRepo
- `AWS::SecretsManager::Secret` — ServiceNowSecret, SSLCertSecret
- `AWS::ElasticLoadBalancingV2::TargetGroup` — DummyAppTargetGroup
- `AWS::ElasticLoadBalancingV2::ListenerRule` — ListenerRuleDummyAppRoot, ListenerRuleDummyAppApi
- `AWS::IAM::Role` — ActionLambdaRole, DummyAppTaskRole
- `AWS::Logs::LogGroup` — 7 new log groups
- `AWS::ECS::TaskDefinition` — DummyAppTaskDefinition
- `AWS::ECS::Service` — DummyAppService
- `AWS::Lambda::Function` — ServiceNowLambda, SSLRemediationLambda, PasswordResetLambda, DBRemediationLambda, ComputeRemediationLambda
- `AWS::Lambda::Permission` — 5 Bedrock permissions

---

## STEP 5 — Run CodeBuild

CodeBuild must run once to:
- Build the dummy-app Docker image and push to the new ECR repo
- Package all 5 action Lambdas and update them from stubs to real code
- Force-redeploy all 4 ECS services so they pick up the updated code

1. Open **CodeBuild → Build projects → log-aggregator-build-ENVNAME**
2. Click **Start build → Start build**
3. Watch the build log. You should see these phases complete:
   ```
   === Building Dummy App image ===
   === Packaging ServiceNow Lambda ===
   === Packaging SSL Lambda ===
   === Packaging Password Reset Lambda ===
   === Packaging DB Lambda ===
   === Packaging Compute Lambda ===
   === Deploying Action Lambdas ===
   === Redeploying ECS Services ===
   === Build complete. Tag -> 20260422XXXXXX ===
   ```
4. Wait ~8 minutes for completion

If the build fails at "Building Dummy App image" with
"repository does not exist":
→ The ECR repo was just created. Wait 1 minute and retry — ECR repos
  occasionally take a moment to become available after creation.

---

## STEP 6 — Verify Lambda Handlers Are Real Code

After CodeBuild completes, confirm the stubs were replaced:

```bash
ENV="prod"   # your EnvironmentName

for FUNC in servicenow ssl password-reset db compute; do
  HANDLER=$(aws lambda get-function-configuration \
    --function-name "log-aggregator-${FUNC}-${ENV}" \
    --query Handler --output text --region us-east-1)
  echo "log-aggregator-${FUNC}-${ENV}: $HANDLER"
done
```

Expected output:
```
log-aggregator-servicenow-prod:     servicenow_lambda_handler.handler
log-aggregator-ssl-prod:            ssl_lambda_handler.handler
log-aggregator-password-reset-prod: password_reset_lambda_handler.handler
log-aggregator-db-prod:             db_lambda_handler.handler
log-aggregator-compute-prod:        compute_lambda_handler.handler
```

If any still shows `index.handler` → that Lambda's source file was missing
from the ZIP. Check the build log for errors on that specific package step.

---

## STEP 7 — Verify Dummy App Is Running

```bash
# Get ALB DNS from stack outputs
ALB=$(aws cloudformation describe-stacks \
  --stack-name log-aggregator-prod \
  --query "Stacks[0].Outputs[?OutputKey=='AlbUrl'].OutputValue" \
  --output text --region us-east-1)

echo "ALB: $ALB"

# Test dummy app health
curl -s "$ALB/health" | python3 -m json.tool
# Expected: {"status": "healthy", "service": "dummy-infra-app", ...}

# Open SSL demo page in browser
echo "Open in browser: $ALB/dummy"
```

If `/health` returns 502 or connection refused:
- ECS dummy app task may still be starting (wait 2–3 minutes)
- Check ECS → Clusters → your cluster → DummyAppService → Tasks
- Check CloudWatch logs: `/ecs/log-aggregator-dummy-app-ENVNAME`

---

## STEP 8 — Update ServiceNow Secret with Real Credentials

The ServiceNow secret was created with placeholder values.
Update it now with your real developer instance credentials:

```bash
aws secretsmanager put-secret-value \
  --secret-id servicenow/credentials \
  --secret-string '{
    "instance_url": "https://devXXXXX.service-now.com",
    "username":     "aws-log-aggregator",
    "password":     "YourRealPassword123!"
  }' \
  --region us-east-1
```

Then set DEMO_MODE=false on the ServiceNow Lambda:

```bash
ENV="prod"
aws lambda update-function-configuration \
  --function-name "log-aggregator-servicenow-${ENV}" \
  --environment "Variables={
    SERVICENOW_SECRET_NAME=servicenow/credentials,
    AWS_DEFAULT_REGION=us-east-1,
    DEMO_MODE=false
  }" \
  --region us-east-1
```

---

## STEP 9 — Set Up Bedrock Agent (if not done yet)

If you have not created the Bedrock agent yet:

1. **Bedrock → Agents → Create Agent**
   - Name: `log-aggregator-orchestrator`
   - Model: Claude 3 Sonnet
   - Instructions: paste full contents of `orchestrator_agent_prompt.txt`

2. Add action groups using the Lambda ARNs from your stack Outputs:
   - `servicenow_action_group` → ServiceNowLambdaArn (3 functions: createIncident, getTicketStatus, updateTicket)
   - `ssl_remediation_action_group` → SSLLambdaArn (1 function: remediateSSL)
   - `password_reset_action_group` → PasswordResetLambdaArn
   - `db_remediation_action_group` → DBLambdaArn
   - `compute_remediation_action_group` → ComputeLambdaArn

3. Click **Prepare** → **Create alias** → name: `v1`

4. Update BedrockSecret with real agent credentials:
```bash
aws secretsmanager put-secret-value \
  --secret-id log-aggregator/bedrock-prod \
  --secret-string '{
    "BEDROCK_AGENT_ID":       "YOUR_AGENT_ID",
    "BEDROCK_AGENT_ALIAS_ID": "YOUR_ALIAS_ID"
  }' \
  --region us-east-1
```

5. Force redeploy dashboard so it picks up the new Bedrock credentials:
```bash
aws ecs update-service \
  --cluster log-aggregator-cluster-prod \
  --service log-aggregator-dashboard-svc-prod \
  --force-new-deployment \
  --region us-east-1
```

---

## STEP 10 — Run the Full Demo

Open in your browser:
```
http://YOUR_ALB_DNS/dummy
```

Follow the three-step flow:
1. Click **🔴 Trigger SSL Expired** — error injected, log shipped to S3
2. Click **🎫 Create Ticket** — Bedrock creates ServiceNow INC ticket
3. Click **⚡ Fix Issue** — Bedrock calls SSL Lambda, ticket resolved, banner goes green

---

## Troubleshooting the Update

### "UPDATE_ROLLBACK_COMPLETE" after stack update

The stack rolled back. Check the **Events** tab for the first FAILED event.

Common causes:
- **IAM role name already exists** — if you previously deployed a partial version of the new template, IAM role names like `log-aggregator-action-lambda-role-prod` may already exist but belong to a different stack or were created manually. Delete the conflicting role from IAM Console and retry the update.
- **S3 bucket name conflict** — the dummy-app ECR repo name already exists in your account. Go to ECR → delete the empty repo → retry.
- **Template syntax error** — validate the template first:
  ```bash
  aws cloudformation validate-template \
    --template-url https://YOUR_BUCKET.s3.amazonaws.com/template_upgraded.yaml \
    --region us-east-1
  ```

### "DummyAppService stuck in UPDATE_IN_PROGRESS"

The dummy app ECR repo exists but has no image yet (CodeBuild hasn't run).
ECS cannot pull the image so the task keeps failing.

Fix: Run CodeBuild (Step 5) first. ECS will stabilise after the dummy-app
image is pushed.

Alternatively, set `DummyAppDesiredCount=0` in the stack parameters
to skip the dummy app service until CodeBuild has run.

### Security group update causes brief traffic disruption

The `AlbSecurityGroup` and `EcsTaskSecurityGroup` updates add new inbound rules.
CloudFormation updates security groups in-place — this does NOT replace them.
Existing rules are preserved. No traffic disruption to your running services.

### "Resource already exists" on ServiceNowSecret or SSLCertSecret

You created these secrets manually before running the update.
CloudFormation cannot import existing secrets.

Fix: Delete the manually created secrets first, then re-run the update:
```bash
aws secretsmanager delete-secret \
  --secret-id servicenow/credentials \
  --force-delete-without-recovery \
  --region us-east-1

aws secretsmanager delete-secret \
  --secret-id dummy-app/ssl-cert-arn \
  --force-delete-without-recovery \
  --region us-east-1
```

Wait 30 seconds, then re-run the stack update.
