# ec2-actions-runner

⚠️ This is a new project and is very experimental

Composite actions for managing an on-demand, self-hosted GitHub actions _repository_ runner (Linux on EC2).

Inspired by <https://github.com/machulav/ec2-github-runner>

## Requirements

- AWS account and VPC network
  - A default VPC works fine, too
- AWS credentials with EC2 permissions
  - You can use [OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) (recommended) to assume a role, or a plain IAM user
- VPC subnet with Internet access
  - Public subnet (public IP) **OR**
  - Private subnet and NAT gateway
- Linux runner AMI (amd64 or arm64), with the following things pre-configured:
  - Non-root user to run actions-runner service as
  - [Actions-runner](https://github.com/actions/runner) v2.283.1+ and required [dependencies](https://github.com/actions/runner/blob/main/docs/start/envlinux.md)
  - `git`, `docker`, `curl` and optionally `at` (if using the `auto-shutdown-at` feature)
  - See e.g. <https://github.com/superblk/ec2-actions-runner-ami-linux-arm64> for an example AMI build
- EC2 launch template (AMI, instance type, VPC subnet, security groups, spot options etc)
  - See example [Cloudformation template](https://gist.github.com/jpalomaki/003c4d173a856cf64c6d35f8869a2de8) that sets up a launch template
- GitHub personal access token (PAT) with `repo` scope

See [start/action.yml](start/action.yml) and [stop/action.yml](stop/action.yml) for all available input parameters.

## Example workflows

💡 EC2 instance ID is automatically assigned as a unique, self-hosted runner label

⚠️ Do not simply copy these examples verbatim, but adjust action version, AWS region, launch template name etc to match your config

### Simple

Simple default. Leverages ephemeral runners that are automatically deregistered from GitHub after the `main` job has run.

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

A more fail-safe alternative. Deregisters GitHub runner explicitly (not relying on ephemeral runner auto-deregistration behavior alone). Also leverages EC2 [instance-initiated shutdown](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html#Using_ChangingInstanceInitiatedShutdownBehavior) **terminate** behavior for ensuring the EC2 instance is terminated, even if the `stop-runner` job fails to run.

⚠️ For the automatic dead-man's switch termination to work, the AMI must include the `at` tool, and the EC2 launch template must specify instance-initiated shutdown behavior as **terminate**.

💡 This example also illustrates the use of extra runner labels and a matrix `main` job, that uses both GitHub-hosted and self-hosted runners.

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
