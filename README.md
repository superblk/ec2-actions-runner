# ec2-actions-runner

⚠️ This is a new project and is very experimental

Composite actions for managing an on-demand, self-hosted GitHub actions _repository_ runner (Linux on EC2).

Inspired by <https://github.com/machulav/ec2-github-runner>

## Requirements

- AWS account and VPC network
- AWS credentials with EC2 permissions
- VPC subnet with Internet access (either assigning public IP or via NAT gateway)
- Linux runner AMI (amd64 or arm64), with the following things pre-configured:
  - [Actions-runner](https://github.com/actions/runner) and its [dependencies](https://github.com/actions/runner/blob/main/docs/start/envlinux.md)
  - Non-root user to run actions-runner service as
  - `git`, `docker`, `curl` and optionally `at` (if using the auto-shutdown feature)
  - See e.g. <https://github.com/superblk/ec2-actions-runner-ami-linux-arm64>
- EC2 launch template (AMI, instance type, VPC, security group, spot options etc)
  - See example [Cloudformation template](https://gist.github.com/jpalomaki/003c4d173a856cf64c6d35f8869a2de8) that sets up a launch template
- GitHub personal access token (PAT) with `repo` scope

## Example workflow

See [start/action.yml](start/action.yml) and [stop/action.yml](stop/action.yml) for all available input parameters.

:warning: do not copy this example verbatim, but adjust action version, AWS region, launch template etc to match your config

```yaml
jobs:
  start-runner:
    runs-on: ubuntu-20.04
    steps:
      - id: runner
        name: Start runner
        uses: superblk/ec2-actions-runner/start@<release>
        with:
          aws-region: eu-north-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-launch-template: LaunchTemplateName=my-arm64-runner
          github-token: ${{ secrets.GH_PAT }}
          runner-labels: ubuntu-18.04-arm64
    outputs:
      runner-id: ${{ steps.runner.outputs.runner-id }}
      instance-id: ${{ steps.runner.outputs.instance-id }}

  complex-build:
    needs: start-runner
    runs-on: ${{ matrix.runner }}
    strategy:
      matrix:
        include:
          - runner: ubuntu-18.04
          - runner: ubuntu-18.04-arm64
    steps:
      - run: uname -a

  stop-runner:
    if: always()
    needs: [start-runner, complex-build]
    runs-on: ubuntu-20.04
    steps:
      - name: Stop runner
        uses: superblk/ec2-actions-runner/stop@<release>
        with:
          aws-region: eu-north-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          github-token: ${{ secrets.GH_PAT }}
          runner-id: ${{ needs.start-runner.outputs.runner-id }}
          instance-id: ${{ needs.start-runner.outputs.instance-id }}
```
