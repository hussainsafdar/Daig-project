# Phase 4 — Infrastructure as Code

## Task 1: Terraform validate

terraform init -backend=false succeeded (AWS provider 5.100.0, random 3.9.0).
terraform validate: Success, configuration is valid.
terraform fmt -check -recursive flagged data.tf - a cosmetic alignment issue only (extra spaces before = on one line), not a functional bug.
