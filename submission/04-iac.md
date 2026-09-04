# Phase 4 — Infrastructure as Code

## Task 1: Terraform validate

terraform init -backend=false succeeded (AWS provider 5.100.0, random 3.9.0).
terraform validate: Success, configuration is valid.
terraform fmt -check -recursive flagged data.tf - a cosmetic alignment issue only (extra spaces before the equals sign on one line), not a functional bug.

## Task 2: Three production-safety controls

1. ECS lifecycle block (compute.tf): lifecycle { ignore_changes = [desired_count] }. Application Auto Scaling owns desired_count at runtime. Without this, a routine terraform apply would reset desired_count back to the static value in the Terraform config and terminate live capacity mid-spike - exactly the iftar scenario the stack teaches. This prevents Terraform from fighting the autoscaler.

2. DB TLS enforcement (data.tf): two layers. Client-side, the generated DATABASE_URL uses sslmode=require. Server-side, an aws_db_parameter_group sets rds.force_ssl=1, which makes RDS reject any cleartext connection outright regardless of what the client requests. This prevents a misconfigured or older client from silently connecting over plaintext.

3. VPC flow logs (network.tf): aws_flow_log captures ALL traffic type to CloudWatch Logs. This prevents "who talked to what" from being unanswerable after an incident - without it, a compromised instance's network activity would leave no trace to investigate.

## Task 3: Remote state backend

The S3 backend block in versions.tf is commented out, with the comment explaining why: "Local state is fine for one intern on one laptop and wrong the moment two people touch the same infrastructure." With local state, each engineer's terraform.tfstate lives only on their own machine. The moment two engineers apply against local state independently, neither sees the other's changes - Terraform has no shared record of what already exists, so it can create duplicate resources, silently drift from reality, or two applies running concurrently can corrupt each other's state. The commented S3 backend (with a DynamoDB table for locking) would fix this by giving every engineer the same shared state file and preventing concurrent applies via a lock.

## Task 4: Ansible idempotency

ansible/roles/docker/tasks/main.yml uses Ansible modules (file, get_url, apt_repository, apt, copy, service) that are declarative rather than imperative - each task describes a desired end state, not a sequence of actions, and every module checks current state before acting. Running the playbook twice produces "changed" then "ok" because on the first run nothing exists yet (directory absent, Docker not installed, daemon.json absent, service not started), so every task reports changed as it creates that state. On the second run, each module checks first: the keyrings directory already exists, the apt packages are already at the target state, daemon.json's content already matches, and the docker service is already started - so every task reports ok (no action taken) instead of changed.

The clearest example is "Ensure Docker is running" (service module, state: started). This task must be idempotent to make the whole run safe to repeat: it does not "start Docker" unconditionally, it checks whether Docker is already running and only issues a start command if it is not. Without that check, repeatedly running the playbook could send redundant start commands or, worse, if the module were naive it could restart a Docker daemon that has containers actively serving production traffic.
