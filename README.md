# devops-agent-demo

CI/CD for deploying an [AWS DevOps Agent](https://aws.amazon.com/devops-agent/) demo environment via CloudFormation, driven by GitHub Actions using OIDC (no static AWS credentials stored in GitHub).

Based on AWS's official guides:

- [Getting started with AWS DevOps Agent using AWS CloudFormation](https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-getting-started-with-aws-devops-agent-using-aws-cloudformation.html)
- [Creating a test environment](https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-creating-a-test-environment.html)

## Repo layout

```
cloudformation/
  devops-agent-stack.yaml   # Agent Space, IAM roles, monitor association (Part 1)
  ec2-cpu-test.yaml         # Intentionally-failing EC2 CPU stress demo
  lambda-error-test.yaml    # Intentionally-failing Lambda error demo

.github/workflows/
  test-oidc-config.yml      # Sanity check: assumes the role, lists stacks
  deploy-devops-agent.yml   # Deploys the Agent Space stack
  deploy-failure-demos.yml  # Deploys the EC2/Lambda failure demo stacks (manual)
  destroy-failure-demos.yml # Tears down the failure demo stacks (manual)
```

## Prerequisites

- An AWS IAM role (`aws-devops-demo` in this setup) trusted by GitHub's OIDC provider (`token.actions.githubusercontent.com`), scoped to this repo via the `sub` claim.
- That role needs permissions to create/delete: IAM roles (`CAPABILITY_NAMED_IAM`), the `AWS::DevOpsAgent::AgentSpace`/`AWS::DevOpsAgent::Association` resource types, EC2 instances/security groups/key pairs, Lambda functions, and CloudWatch alarms.
- Two GitHub Actions secrets on this repo (Settings → Secrets and variables → Actions):

  | Secret         | Value                                                                       |
  | -------------- | --------------------------------------------------------------------------- |
  | `AWS_ROLE_ARN` | ARN of the OIDC role, e.g. `arn:aws:iam::<account-id>:role/aws-devops-demo` |
  | `AWS_REGION`   | AWS region to deploy into, e.g. `us-east-1`                                 |

## Workflows

### `test-oidc-config.yml`

Runs on every push to `main`. Confirms the OIDC role can be assumed and lists existing CloudFormation stacks. No resources created.

### `deploy-devops-agent.yml`

Runs on push to `main` (when `cloudformation/devops-agent-stack.yaml` or the workflow itself changes), or manually via `workflow_dispatch`. Deploys the `DevOpsAgentStack`: the Agent Space, its IAM roles, and a monitor association for this account. This is the one-time setup step — after it succeeds you have a working Agent Space.

### `deploy-failure-demos.yml`

Manual only (`workflow_dispatch`), with a `demo` input (`both` / `ec2` / `lambda`). Deploys one or both intentionally-broken stacks so the Agent has something to investigate:

- **EC2 CPU test** (`AWS-DevOpsAgent-EC2-Test`) — auto-discovers the account's default VPC/subnet, launches a `t3.micro` that runs a CPU stress script ~5 minutes after boot and trips a `CPUUtilization > 70%` alarm. Auto-terminates after 2 hours.
- **Lambda error test** (`AWS-DevOpsAgent-Lambda-Test`) — deploys a Lambda that always raises, paired with an `Errors > 0` alarm.

Triggering the actual failure is manual by design (so you control demo timing):

- EC2: the stress script runs on its own, or re-run it via Session Manager (`./cpu-stress-test.sh`).
- Lambda: invoke it 2-3 times, ~10s apart:
  ```
  aws lambda invoke --function-name AWS-DevOpsAgent-test-lambda \
    --payload '{"test":"trigger"}' response.json --region <region>
  ```

### `destroy-failure-demos.yml`

Manual only, same `demo` input. Deletes the corresponding demo stack(s). Run this after each demo to avoid leaving resources running.

## Running a demo end to end

1. Push to `main` (or run `deploy-devops-agent.yml` manually) to stand up the Agent Space, if not already deployed.
2. Run `deploy-failure-demos.yml` with `demo: both`.
3. Wait a few minutes for the CloudWatch alarms to trip (EC2 alarm is automatic; invoke the Lambda manually to trip its alarm).
4. In the DevOps Agent Space web app (Admin access → Start Investigation), start an investigation referencing the alarm(s) and watch the Agent triage it.
5. Run `destroy-failure-demos.yml` with `demo: both` to clean up.

## Notes

- All AWS auth is via GitHub OIDC — no long-lived AWS access keys are stored anywhere in this repo.
- Region and role ARN are read from GitHub Actions secrets (`AWS_REGION`, `AWS_ROLE_ARN`), not hardcoded in the workflow files..
