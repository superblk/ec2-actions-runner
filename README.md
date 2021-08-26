# ec2-actions-runner

⚠️ This is a new project and is very experimental

Composite actions for managing an on-demand, self-hosted GitHub actions _repository_ runner (Linux on EC2).

## Requirements

- AWS account
- Default VPC + subnet
- AWS credentials/role with EC2 permissions
- Linux AMI (amd64 or arm64), with
  - Non-root user to run actions-runner as
  - [Runner](https://github.com/actions/runner) software and [dependencies](https://github.com/actions/runner/blob/main/docs/start/envlinux.md)
  - See e.g. <https://github.com/superblk/ec2-actions-runner-ami-linux-arm64>
- GitHub personal access token (PAT) with `repo` scope
