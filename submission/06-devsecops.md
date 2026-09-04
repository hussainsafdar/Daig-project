# Phase 6 — DevSecOps

## Task 1: Enable vulnerabilities and scan

make insecure-on enabled INSECURE_MODE on orders. Ran ./security/scan-all.sh sast: 18 findings across 330 rules / 257 files.

Triage table:

| Finding | File | Real vs Teaching | Notes |
|---|---|---|---|
| dependabot-missing-cooldown x11 | .github/dependabot.yml | Real (generic, low priority) | Built-in Semgrep registry rule, unrelated to the planted vulns |
| gha-workflow-env-secret | infracost.yml | Real | Secret in workflow-level env block |
| aws-subnet-has-public-ip | infra/aws/network.tf | Real | Generic Terraform best-practice rule |
| gcp-sql-database-ssl-insecure | infra/gcp/data.tf | Real | Generic Terraform best-practice rule |
| daig-env-in-response | insecure.js:92 | Teaching (planted) | Custom daig-* rule - matches VULN-3 |
| daig-stack-trace-in-response | insecure.js:92 | Teaching (planted) | Custom daig-* rule - matches VULN-3 |
| daig-weak-hash | insecure.js:115 | Teaching (planted) | Custom daig-* rule - matches VULN-4 (MD5) |
| request-host-used | web/nginx.conf:45 | Real (generic) | Built-in nginx rule |

Pattern used to distinguish real from teaching: findings whose check_id is prefixed daig-* are this repo's own custom rules aimed at the six planted vulnerabilities in insecure.js; all other findings are generic Semgrep registry rules that fire on any repo and are unrelated to this exercise.

## Task 2 & 3: What Semgrep missed, and the one that's essentially invisible

insecure.js documents six named vulnerabilities (VULN-1 through VULN-6, each tagged with a CWE). Semgrep's SAST scan only caught two of them:
- VULN-3 (verbose error disclosure, CWE-209) - caught via daig-env-in-response and daig-stack-trace-in-response
- VULN-4 (weak hashing / MD5, CWE-327) - caught via daig-weak-hash

Semgrep missed VULN-1 (SQL injection via string concatenation, CWE-89), VULN-2 (IDOR / broken object-level authorization, CWE-639), VULN-5 (no rate limiting, CWE-770), and VULN-6 (SSRF, CWE-918) - none of these produced a finding in this run, likely because the six custom daig-* rules in security/semgrep/daig.yml only cover a subset of the six vulnerabilities (matching VERIFICATION.md's own warning that the custom rule syntax was the most likely thing needing adjustment).

The one that is essentially invisible to every scanner by design, not by rule-coverage gap, is VULN-2 (IDOR). The code comment in insecure.js explains why directly: "the code is not wrong in any local sense - the query is parameterised, there is no injection, nothing is unsafe. What is missing is a business rule, and a tool cannot know your business rules." A SAST tool pattern-matches code shape (string concatenation, weak hash functions, secrets in responses) - all local, syntactic properties. IDOR requires knowing that the /insecure/orders/:id handler should but does not check req.user.id against the order's owner - that is a missing semantic/business rule, not a code-shape defect, and no generic scanner can know what "ownership" means in this domain. This is exactly why code review still exists alongside automated scanning.

## Task 2: Fixed VULN-4 (weak hashing), and Task 3: gate

Fixed VULN-4 in services/orders/src/insecure.js: replaced crypto.createHash('md5') with Node's built-in crypto.scryptSync, using a random 16-byte salt stored alongside the derived key (salt:hash format). scrypt is deliberately slow and memory-hard, the opposite property from MD5, which is fast and therefore wrong for password hashing.

Before fix: SAST scan reported 18 findings, including daig-weak-hash at insecure.js:115.
After fix: rebuilt and restarted orders, re-ran ./security/scan-all.sh sast: 17 findings - daig-weak-hash no longer fires. The gate for this vulnerability class already exists as the custom Semgrep rule daig-weak-hash in security/semgrep/daig.yml, which matched crypto.createHash('md5'|'sha1', ...) usage; that same rule now protects against regression - if MD5/SHA1 password hashing is reintroduced anywhere in the codebase, this rule will flag it again in CI's static job.

Gap found along the way: INSECURE_MODE, like
 BAO_ROLE_ID, BAO_SECRET_ID, and BAO_ADDR earlier, was missing from the x-app-env anchor in docker-compose.yml, so exporting INSECURE_MODE=true and recreating the orders container had no effect until the anchor was patched to include INSECURE_MODE.

## Task 4: Secrets from OpenBao

make vault-up bootstrapped OpenBao: initialised, unsealed, audit device enabled before any secret was written, KV v2 mounted, database/app secrets generated (not typed), four policies uploaded (daig-orders, daig-kitchen, daig-dispatch, daig-intern-readonly), and AppRole auth configured per service.

Confirmed orders/kitchen/dispatch loaded credentials from OpenBao rather than the environment via structured logs: "credential_source":"openbao" (as opposed to "credential_source":"environment" seen earlier before Vault was wired up). This was verified repeatedly across the session, including after several environment resyncs.

vault/policies/orders.hcl (and the equivalent kitchen/dispatch policies) grant read access only to daig/data/database, not daig/data/app (which holds JWT_SECRET and PAYMENT_API_KEY). This was directly observed in the OpenBao audit log during this session: a request from the kitchen AppRole to read daig/data/app was logged with "policy_results":{"allowed":false} and "error":"permission denied", while the same role's read of daig/data/database was "allowed":true. This is least-privilege in practice - kitchen and dispatch need database credentials to do their job, but have no legitimate reason to read the payment API key, so their policy simply does not grant that capability.

## Checkpoint answers

The security gates are ordered secrets -> SAST -> ... -> DAST because each stage answers a cheaper, more certain question before spending time on a more expensive, less certain one. Secrets scanning is nearly instant and a hardcoded credential is unambiguous - fix it before anything else. SAST is static analysis on source, still fast, and a code-shape defect (SQL concatenation, weak hash) is also close to unambiguous. DAST requires standing up the running application and probing it over HTTP, which is slow and produces findings that need more interpretation (a WARN might or might not be exploitable in context) - so it runs last, after cheaper gates have already caught what they can.

secrets.js exits 78 rather than falling back to environment variables in production because a service that silently degrades from "secrets from the vault" to "secrets from the environment" turns a misconfiguration into an undetected credential leak - the comment in the code says this directly: that silent degradation "is how a misconfiguration becomes a credential leak nobody notices for six months." Refusing to start loudly, with a named cause, is safer than starting quietly in a weaker security posture.
