# ec2-linux-runner

⚠️ This is a new project and is very experimental

Composite actions for starting and stopping an on-demand, self-hosted GitHub actions _repository_ runner (Linux on EC2).

## Requirements

- AWS account
- Default VPC + subnet
- Linux AMI (amd64 or arm64), with
  - Non-root user to run the runner as
  - [Runner](https://github.com/actions/runner) software and [dependencies](https://github.com/actions/runner/blob/main/docs/start/envlinux.md)
- AWS authentication with permission to run and terminate EC2 instances
  - You can use [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) to authenticate prior to using this action
- GitHub personal access token (PAT) with `repo` scope
