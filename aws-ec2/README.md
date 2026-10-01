# AWS EC2 Demo

## Overview

This project demonstrates how to use the AWS SDK for Go v2 to provision an EC2 instance programmatically. It shows how to create (or reuse) an SSH key pair, look up the latest Ubuntu 20.04 (Focal) AMI owned by Canonical, and launch a `t3.micro` EC2 instance using that AMI and key pair.

## Files

- `main.go` — Loads the default AWS config for `us-east-1`, checks for an existing `go-aws-ec2` key pair (creating one and saving the private key to `go-aws-ec2.pem` if it doesn't exist), finds the latest matching Ubuntu Focal HVM AMI via `DescribeImages`, and runs a single `t3.micro` EC2 instance with `RunInstances`, printing the resulting instance ID.
- `go.mod` — Declares the module `github.com/kaif/aws-ec2`, requires Go 1.25.4, and depends on `github.com/aws/aws-sdk-go-v2` along with the `config` and `service/ec2` sub-packages.
- `go.sum` — Checksums for all module dependencies used to verify integrity of downloaded packages.
- `.gitignore` — Ignores the generated `go-aws-ec2.pem` private key file so it isn't committed to source control.

## Prerequisites

- Go 1.25.4 or later (as specified in `go.mod`).
- Valid AWS credentials configured via `aws configure`, environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`), or another AWS SDK credential source.
- IAM permissions for `ec2:DescribeKeyPairs`, `ec2:CreateKeyPair`, `ec2:DescribeImages`, and `ec2:RunInstances` in the `us-east-1` region (the region is hardcoded in `main.go`).

## How to Run

```powershell
go run main.go
```

No command-line flags or arguments are required; the region (`us-east-1`) and instance type (`t3.micro`) are hardcoded in the source.

## Notes

- **This is a destructive/live-resource-creating program.** Running it will create a real EC2 key pair (if one doesn't already exist) and launch a real, billable `t3.micro` EC2 instance in your AWS account. It does not terminate the instance afterward.
- If a private key file `go-aws-ec2.pem` is created, it is written with `0600` permissions and is excluded from git via `.gitignore` — keep it safe as it is your only copy.
- There are no automated tests in this project.
