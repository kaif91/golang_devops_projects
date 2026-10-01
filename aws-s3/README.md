# AWS S3 Demo

## Overview

This project demonstrates basic Amazon S3 operations using the AWS SDK for Go v2. It shows how to create an S3 bucket, upload a text file to it using the SDK's managed uploader, and then download that same file back into memory using the managed downloader, printing its contents to confirm the round trip worked.

## Files

- `main.go` — Initializes an S3 client for `us-east-1`, creates a bucket named `go-aws-test-xz9` via `CreateBucket`, uploads an object `test.txt` containing the string `"this is a test"` using `manager.Uploader`, then downloads `test.txt` back using `manager.Downloader` into an in-memory buffer and prints the downloaded content.
- `go.mod` — Declares the module `github.com/kaif/aws-s3`, requires Go 1.25.4, and depends on `github.com/aws/aws-sdk-go-v2` plus the `config`, `service/s3`, and `feature/s3/manager` sub-packages.
- `go.sum` — Checksums for all module dependencies used to verify integrity of downloaded packages.

## Prerequisites

- Go 1.25.4 or later (as specified in `go.mod`).
- Valid AWS credentials configured via `aws configure`, environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN`), or another AWS SDK credential source.
- IAM permissions for `s3:CreateBucket`, `s3:PutObject`, and `s3:GetObject` in the `us-east-1` region (the region and bucket name `go-aws-test-xz9` are hardcoded in `main.go`).
- Note: S3 bucket names must be globally unique; `go-aws-test-xz9` may already be taken by another AWS account, which would cause `CreateBucket` to fail.

## How to Run

```powershell
go run main.go
```

No command-line flags or arguments are required; the region (`us-east-1`) and bucket name (`go-aws-test-xz9`) are hardcoded in the source.

## Notes

- **This program creates real AWS resources.** It creates an S3 bucket and uploads an object to your AWS account; it does not delete or clean up the bucket/object afterward.
- If the bucket already exists (e.g., from a prior run), `CreateBucket` may return an error (such as `BucketAlreadyOwnedByYou` or `BucketAlreadyExists`) depending on region/ownership, which will cause the program to exit early.
- There are no automated tests in this project.
