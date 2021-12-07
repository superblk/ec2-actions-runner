# ec2-actions-runner

Composite actions for managing an on-demand, self-hosted GitHub actions _repository_ runner (Linux on EC2)

⚠️ This is a new project and as such, backwards-incompatible changes may occur between releases

⚠️ Self-hosted runners should **not** be used with _public_ repositories (see GitHub [documentation](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners#self-hosted-runner-security))

Inspired by <https://github.com/machulav/ec2-github-runner> ❤️

## Pre-requisites

- AWS account
  - Permissions to provision IAM, EC2 and VPC resources (to set up the runner AWS scaffolding)
- VPC network
  - Subnet with Internet access (required because self-hosted runners communicate with `github.com`)

## Limitations

- GitHub Enterprise Server (GHES) is not currently supported

## Setup

1. AWS: Configure GitHub OIDC identity provider (GitHub [documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services))
    - Use of OIDC is recommended (safer), because static AWS access keys need not be stored in GitHub secrets
    - NOTE: if you cannot configure OIDC-assumable roles, it is possible to use an IAM user with static access keys
2. AWS: Configure the IAM role that is assumed by the workflow, used only _for starting and stopping the runner EC2 instances_
    - Example OIDC assume role (trust) policy, that defines which GitHub repos can assume the role (see example [CloudFormation template](https://github.com/aws-actions/configure-aws-credentials#sample-iam-role-cloudformation-template))

      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Principal": {
              "Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": [
              "sts:AssumeRoleWithWebIdentity",
              "sts:TagSession"
            ],
            "Condition": {
              "StringLike": {
                "token.actions.githubusercontent.com:sub": "repo:<owner>/<repository>:*"
              }
            }
          }
        ]
      }
      ```
    - Example role policy (inline or customer-managed), that defines the _minimum_ permissions needed for starting/stopping runner EC2 instances

      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": [
              "ec2:RunInstances",
              "ec2:TerminateInstances"
            ],
            "Resource": "*"
          },
          {
            "Effect": "Allow",
            "Action": [
              "ec2:CreateTags"
            ],
            "Resource": "*",
            "Condition": {
              "StringEquals": {
                "ec2:CreateAction": "RunInstances"
              }
            }
          }
        ]
      }
      ```
    - If you need to assign an IAM instance profile (role) to the EC2 instances, you need to use a policy that includes the `iam:PassRole` permission

      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": [
              "ec2:RunInstances",
              "ec2:TerminateInstances"
            ],
            "Resource": "*"
          },
          {
            "Effect": "Allow",
            "Action": [
              "ec2:CreateTags"
            ],
            "Resource": "*",
            "Condition": {
              "StringEquals": {
                "ec2:CreateAction": "RunInstances"
              }
            }
          },
          {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<account>:role/<role for EC2>"
          }
        ]
      }
      ```
4. AWS: Linux runner AMI (amd64 or arm64), with the following things pre-configured:
    - Non-root user to run the actions-runner service as
    - [Actions-runner](https://github.com/actions/runner) v2.283.1+ and required [dependencies](https://github.com/actions/runner/blob/main/docs/start/envlinux.md)
    - `git`, `docker`, `curl` and optionally `at` (if using the `auto-shutdown-at` feature)
    - See e.g. <https://github.com/superblk/ec2-actions-runner-ami-ubuntu-18.04-arm64> for an example
5. AWS: EC2 runner launch template (defines AMI, instance type, VPC subnet, security groups, instance profile, spot options etc)
    - See example [Cloudformation template](https://gist.github.com/jpalomaki/003c4d173a856cf64c6d35f8869a2de8) that sets up a launch template
6. GitHub: personal access token (PAT) with `repo` scope, required for registering self-hosted repository runners

## Example workflows

See [start/action.yml](start/action.yml) and [stop/action.yml](stop/action.yml) for all available input parameters

💡 EC2 instance ID is automatically assigned as a unique, self-hosted runner label

⚠️ Do not simply copy these examples verbatim, but adjust action version, AWS region, launch template name etc to match your configuration

### Simple

Leverages ephemeral runners that are automatically deregistered from GitHub after the `main` job has run to completion

```yaml
jobs:
  start-runner:
    permissions:
      id-token: write
    runs-on: ubuntu-20.04
    steps:
      - id: runner
        name: Start runner
        uses: superblk/ec2-actions-runner/start@<release>
        with:
          aws-region: eu-north-1
          aws-role-to-assume: arn:aws:iam::<account>:role/<role>
          aws-launch-template: LaunchTemplateName=my-special-runner
          github-token: ${{ secrets.GH_PAT }}
    outputs:
      instance-id: ${{ steps.runner.outputs.instance-id }}

  main:
    needs: start-runner
    runs-on: ${{ needs.start-runner.outputs.instance-id }}
    steps:
      - run: uname -a

  stop-runner:
    if: always()
    permissions:
      id-token: write
    needs: [start-runner, main]
    runs-on: ubuntu-20.04
    steps:
      - name: Stop runner
        uses: superblk/ec2-actions-runner/stop@<release>
        with:
          aws-region: eu-north-1
          aws-role-to-assume: arn:aws:iam::<account>:role/<role>
          instance-id: ${{ needs.start-runner.outputs.instance-id }}
```

### Advanced

A more fail-safe alternative. Deregisters GitHub runner explicitly (not relying on ephemeral runner auto-deregistration behavior alone). Also leverages EC2 [instance-initiated shutdown](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html#Using_ChangingInstanceInitiatedShutdownBehavior) terminate behavior for ensuring the EC2 instance is terminated, even if the `stop-runner` job fails to run

💡 This example also illustrates the use of extra runner labels and a matrix `main` job, that uses both GitHub-hosted and self-hosted runners

⚠️ For the automatic dead-man's switch termination to work, the AMI must include the `at` tool, and the EC2 launch template must specify instance-initiated shutdown behavior as _terminate_

```yaml
jobs:
  start-runner:
    permissions:
      id-token: write
    runs-on: ubuntu-20.04
    steps:
      - id: runner
        name: Start runner
        uses: superblk/ec2-actions-runner/start@<release>
        with:
          aws-region: eu-north-1
          aws-role-to-assume: arn:aws:iam::<account>:role/<role>
          aws-launch-template: LaunchTemplateName=my-special-runner
          runner-labels: ubuntu-18.04-arm64-${{ github.run_id }}
          github-token: ${{ secrets.GH_PAT }}
          auto-shutdown-at: 'now + 3 hours'
    outputs:
      instance-id: ${{ steps.runner.outputs.instance-id }}
      runner-id: ${{ steps.runner.outputs.runner-id }}

  main:
    needs: start-runner
    runs-on: ${{ matrix.runner }}
    strategy:
      matrix:
        include:
          - runner: ubuntu-18.04
          - runner: ubuntu-18.04-arm64-${{ github.run_id }}
    steps:
      - run: uname -a

  stop-runner:
    if: always()
    permissions:
      id-token: write
    needs: [start-runner, main]
    runs-on: ubuntu-20.04
    steps:
      - name: Stop runner
        uses: superblk/ec2-actions-runner/stop@<release>
        with:
          aws-region: eu-north-1
          aws-role-to-assume: arn:aws:iam::<account>:role/<role>
          instance-id: ${{ needs.start-runner.outputs.instance-id }}
          runner-id: ${{ needs.start-runner.outputs.runner-id }}
          github-token: ${{ secrets.GH_PAT }}
```
