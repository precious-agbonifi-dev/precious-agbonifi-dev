# Terraform State Management Best Practices

> **Author:** Precious Agbonifi | **Last updated:** March 2026

Managing Terraform state well is the difference between a reliable infrastructure workflow and a team that's afraid to run `terraform apply`. This guide covers the patterns I use in production across AWS and GCP environments.

---

## Why State Management Matters

Terraform state is the source of truth for your infrastructure. It maps your configuration to real-world resources. Poor state management leads to:

- **State drift** — real infrastructure diverges from what Terraform knows about
- **Race conditions** — two engineers applying simultaneously corrupt state
- **Leaked secrets** — sensitive values stored in plain-text local `.tfstate` files
- **Lost state** — local state files deleted or corrupted with no recovery path

---

## Rule 1: Always Use Remote State

Never commit `.tfstate` files to Git. Always configure a remote backend.

### AWS S3 + DynamoDB (recommended for AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "prod/eks-cluster/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

Create the DynamoDB table for locking:

```bash
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### GCS (recommended for GCP)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-org-tfstate"
    prefix = "prod/gke-cluster"
  }
}
```

GCS provides native object locking — no separate lock table needed.

---

## Rule 2: Structure State by Environment & Component

Avoid a single monolithic state file. Split state by:

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── networking/    # state: dev/networking/terraform.tfstate
│   │   ├── eks/           # state: dev/eks/terraform.tfstate
│   │   └── rds/
│   ├── staging/
│   └── prod/
└── modules/               # reusable modules — no state here
```

**Why?** A `terraform apply` on networking shouldn't risk destroying your database because they share state.

---

## Rule 3: Use State Locking

State locking prevents concurrent operations. With S3 + DynamoDB, locking is automatic. You'll see this when it's working:

```
Acquiring state lock. This may take a few moments...
```

If a lock gets stuck (e.g. a pipeline crashed mid-apply):

```bash
# List current locks
terraform force-unlock <LOCK_ID>
```

Always investigate *why* the lock is stuck before forcing it.

---

## Rule 4: Use `terraform_remote_state` for Cross-Stack References

Instead of hardcoding values between stacks, reference outputs directly:

```hcl
# In eks/main.tf — read VPC outputs from networking stack
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-org-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

module "eks" {
  source     = "../../modules/eks"
  vpc_id     = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.networking.outputs.private_subnets
}
```

---

## Rule 5: Protect Production State

Apply strict IAM policies on your state bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-org-terraform-state/prod/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::123456789:role/TerraformProdRole"
        }
      }
    }
  ]
}
```

Also enable S3 versioning — it's your rollback mechanism if state gets corrupted.

---

## Rule 6: Never Manually Edit State

If you need to make state changes, use the proper commands:

| Task | Command |
|---|---|
| Remove a resource from state (without destroying it) | `terraform state rm <resource>` |
| Import an existing resource into state | `terraform import <resource> <id>` |
| Rename a resource in state | `terraform state mv <old> <new>` |
| List all resources in state | `terraform state list` |
| Inspect a specific resource | `terraform state show <resource>` |

---

## Summary Checklist

- [ ] Remote backend configured (S3/GCS)
- [ ] State locking enabled
- [ ] State files encrypted at rest
- [ ] State split by environment and component
- [ ] S3 versioning enabled on state bucket
- [ ] Production state protected by IAM
- [ ] No `.tfstate` files in Git (`.gitignore` updated)

---

*Found a mistake or have a suggestion? Open an issue or PR — contributions welcome.*
