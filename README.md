# ec2-actions-runner

⚠️ This is a new project and is very experimental

Composite actions for managing an on-demand, self-hosted GitHub actions _repository_ runner (Linux on EC2).

Inspired by <https://github.com/machulav/ec2-github-runner>

## Requirements

- AWS account
- Default VPC + subnet
- AWS credentials with EC2 permissions
- Linux runner AMI (amd64 or arm64), with
  - [Runner](https://github.com/actions/runner) software and [dependencies](https://github.com/actions/runner/blob/main/docs/start/envlinux.md)
  - Non-root user to run actions-runner with
  - See e.g. <https://github.com/superblk/ec2-actions-runner-ami-linux-arm64>
- GitHub personal access token (PAT) with `repo` scope

## Example workflow

:warning: do not copy this example verbatim, but adjust AWS region, AMI id etc to match your config

```yaml
jobs:
  start-runner:
    runs-on: ubuntu-18.04
    steps:
      - id: runner
        name: Start runner
        uses: superblk/ec2-actions-runner/start@v0.3.0
        with:
          aws-region: eu-north-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          github-token: ${{ secrets.GH_PAT }}
          github-repo: ${{ github.repository }}
          ami-id: ami-12345678901234567
          instance-type: t4g.small
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
            message: GitHub-hosted runner
          - runner: ubuntu-18.04-arm64
            message: Self-hosted EC2 runner
    steps:
      - run: echo '${{ matrix.message }}'

  stop-runner:
    if: always()
    needs: [start-runner, complex-build]
    runs-on: ubuntu-18.04
    steps:
      - name: Stop runner
        uses: superblk/ec2-actions-runner/stop@compose-moar
        with:
          aws-region: eu-north-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          github-token: ${{ secrets.GH_PAT }}
          github-repo: ${{ github.repository }}
          runner-id: ${{ needs.start-runner.outputs.runner-id }}
          instance-id: ${{ needs.start-runner.outputs.instance-id }}
```
