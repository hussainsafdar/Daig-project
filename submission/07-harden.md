# Phase 7 — Harden

## Task 1: NetworkPolicy - default-deny plus tier allows

k8s/base/networkpolicy.yaml defines five NetworkPolicy objects. default-deny applies podSelector: {} with policyTypes [Ingress, Egress] to every pod in the namespace, blocking all traffic by default. allow-dns-egress opens UDP/TCP port 53 to any namespace for every pod, since without DNS the whole namespace looks broken for reasons unrelated to the app. Three tier-scoped policies then reopen only what is needed: web (tier=presentation) accepts ingress on 8080 and may only egress to tier=application; application accepts ingress from presentation and from peer application pods, and may egress to tier=data, to peer application pods, and to the OTLP collector ports (4317/4318); data (tier=data, i.e. postgres/redis) accepts ingress only from tier=application and has no other egress beyond DNS.

Demonstrated on the live cluster (Docker Desktop Kubernetes) with kubectl exec + nc:
- kubectl -n daig exec deploy/web -- nc -zv -w 5 postgres 5432 -> "Operation timed out" (blocked, as expected: web is presentation tier and the data policy only allows ingress from application tier)
- kubectl -n daig exec deploy/orders -- nc -zv -w 5 postgres 5432 -> "postgres (10.244.0.11:5432) open" (allowed, as expected: orders is application tier)

Note: networkpolicy.yaml itself warns that kind's default CNI does not enforce NetworkPolicy objects and recommends installing Calico for the lesson to be real. This cluster's enforcement was verified directly rather than assumed - Docker Desktop's Kubernetes networking layer did enforce the policy in this test, contrary to what a plain kind cluster's default CNI would do.

## Task 2: Pod Security Admission

The daig namespace sets three PSA labels: pod-security.kubernetes.io/enforce: baseline (hard-blocks any pod that fails even the baseline standard - the minimal, widely-known privilege-escalation prevention), pod-security.kubernetes.io/warn: restricted, and pod-security.kubernetes.io/audit: restricted (both warn and audit the stricter restricted standard without blocking on it).

This was observed directly during Phase 3's kubectl apply -k k8s/base: the postgres StatefulSet triggered "Warning: would violate PodSecurity restricted:latest (container postgres must set securityContext.capabilities.drop=[ALL], runAsNonRoot=true)" but was still created, because enforce is only set to baseline, not restricted - the postgres pod's securityContext passes baseline but not restricted.

To enforce restricted, two changes would be needed: (1) change the namespace label to pod-security.kubernetes.io/enforce: restricted, and (2) fix postgres's pod spec to actually satisfy restricted before that label change - add capabilities.drop: [ALL], set runAsNonRoot: true, and set allowPrivilegeEscalation: false on the postgres container's securityContext (the other Deployments in this repo - orders/kitchen/dispatch/web - already set these and would pass unmodified). Without fixing postgres first, flipping enforce to restricted would block postgres from deploying at all.

## Task 3: Chaos egress block

Baseline confirmed first: kubectl -n daig exec deploy/orders -- nc -zv -w 5 postgres 5432 returned "postgres open".

Applied kubectl apply -f k8s/chaos/deny-egress.yaml, which creates deny-orders-egress (podSelector app=orders, egress restricted to DNS only). Re-tested: the connection was STILL open ("postgres open"), and all orders pods stayed Running/Ready - the chaos policy had no observable effect.

Root cause investigated via kubectl describe networkpolicy: both application (podSelector tier=application, which matches orders since orders carries tier: application) and deny-orders-egress (podSelector app=orders) target the same pods for the same policyType (Egress). Kubernetes NetworkPolicies are additive/union, not overriding - when multiple Egress policies match a pod, traffic is allowed if ANY matching policy allows it. Since the pre-existing application policy already grants orders unrestricted egress to tier=data (which includes postgres), the new deny-orders-egress policy's narrower DNS-only rule adds an additional allowed path (DNS) without removing the already-allowed data-tier path. The egress was therefore never actually denied - the policy's name is aspirational, not descriptive of the actual resulting behavior. Removed the chaos policy with kubectl delete -f k8s/chaos/deny-egress.yaml.

## Checkpoint answers

A NetworkPolicy is only meaningful with a policy-aware CNI (Calico, Cilium, Antrea) because NetworkPolicy objects are just API declarations - the Kubernetes control plane accepts and stores them regardless of whether anything actually enforces them at the network layer. The CNI plugin is the component that reads these objects and actually programs iptables/eBPF rules to block or allow packets; without a policy-aware CNI, the objects exist in etcd but traffic flows exactly as it would with no policy at all, which is deceptively dangerous because kubectl get networkpolicy shows the policy exists.

Enforcement should be verified rather than assumed by actually testing connectivity, not by inspecting the YAML. This session did exactly that: kubectl exec into a pod and use nc -zv to attempt a real connection to a target that should be blocked and one that should be allowed, then compare the actual results (web to postgres: timed out; orders to postgres: open) against what the policy claims. Task 3 in this same phase is a direct demonstration of why this matters - the deny-orders-egress policy was applied and looked correct in isolation, but the actual live behavior (connection still open) contradicted what reading the YAML alone would suggest, because of the additive-policy interaction with an existing rule. Reading policy YAML tells you intent; only a live connectivity test tells you the actual enforced behavior.
