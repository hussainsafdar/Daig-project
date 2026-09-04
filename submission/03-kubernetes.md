# Phase 3 — Kubernetes

## Task 1: Render manifests, no cluster

kubectl kustomize k8s/base produced 34 objects. Verified no :latest tags -
all images pinned (app images at 1.0.0, postgres:16-alpine, redis:7-alpine,
bitnami/kubectl:1.31).

## Task 2: Deploy to local cluster (Docker Desktop Kubernetes)

kubectl apply -k k8s/base created all 34 objects successfully.

Gap found: orders/kitchen/dispatch/web pods went into ImagePullBackOff. kubectl describe showed 403 Forbidden pulling ghcr.io/adeelarshad414/daig/*:1.0.0 - these registry images are private/inaccessible. Worked around by pointing deployments at the locally-built images (daig-orders:latest etc, built earlier via docker compose) and setting imagePullPolicy: Never.

After that fix, pods then hit CrashLoopBackOff with exit code 78: "No secret store configured while NODE_ENV=production" - the same exit-78 path from Phase 1, this time triggered because the k8s ConfigMap sets NODE_ENV=production but daig-secrets only holds DATABASE_URL/JWT_SECRET/PAYMENT_API_KEY/POSTGRES_PASSWORD, no BAO_* vars, so secrets.js correctly refuses to fall back to plain env vars in production. Patched NODE_ENV to development in the ConfigMap to unblock the exercise (wiring OpenBao into k8s was out of scope here). After that, all pods reached Running/Ready.

## Task 3: Self-heal

Deleted pod orders-6c967d88d7-9j2m5. A replacement pod (orders-6c967d88d7-fzqvs) was created automatically and reached 1/1 Running within about 15 seconds, confirming the Deployment controller maintains the desired replica count.

## Task 4: Scale and HPA

kubectl get hpa showed all four HPAs reporting cpu: <unknown>/60% - metrics-server is not installed on this Docker Desktop cluster, so HPA has no CPU data to act on. Ran kubectl scale deploy/orders --replicas=5. The Deployment scaled to 5 pods immediately and stayed at 5 - the HPA did not revert it back toward minReplicas=3, because with no metrics available HPA cannot make any scaling decision at all, so a manual kubectl scale is left untouched indefinitely. In a working cluster with metrics-server installed, this manual scale would instead "thrash": the HPA re-evaluates on its sync interval and, seeing actual CPU usage below the 60% target, would scale orders back down toward minReplicas, fighting any manual replica count the operator set - this is why declaring a fixed replicas: on a Deployment that also has an HPA is a conflict: two controllers (the Deployment spec and the HPA) both try to own the same field, and the last writer wins on each reconciliation pass.

## Task 5: Rollback

Set a bad image on orders (daig-orders:doesnotexist). The rollout stalled: with maxUnavailable=0 in the RollingUpdate strategy, Kubernetes never terminated the 5 existing working pods while waiting for the new pod to become Ready - the new pod sat in ErrImageNeverPull while all 5 old pods kept serving traffic. No outage occurred. Ran kubectl rollout undo deploy/orders, which reverted the Deployment to its previous revision (already stored by Kubernetes) - all pods returned to Running within seconds, with no rebuild needed since rollback just repoints the Deployment at a prior ReplicaSet.

## Task 6: Crashloop chaos

Ran ./chaos/day3-crashloop.sh break, which started a container daig-chaos-orders. docker inspect showed exit code 78, and docker logs showed "No secret store configured while NODE_ENV=production" - the same exit-78 config-error path from Phase 1 and from the Kubernetes deploy in Task 2, confirming this was a missing-configuration failure rather than wrong config or wrong environment. Reverted with ./chaos/day3-crashloop.sh fix.

## Checkpoint answers

Declaring both replicas: on the Deployment and an HPA causes thrash because both the Deployment spec and the HPA controller try to own
 the same replica count field - the HPA recalculates the desired count on its sync interval based on live metrics and overwrites any manually declared replicas value, while a human or GitOps tool setting replicas fights back on the next apply, so the two controllers oscillate the count against each other.

The bad rollout did not cause an outage because maxUnavailable: 0 in the RollingUpdate strategy guarantees the old, working ReplicaSet is never scaled down until the new ReplicaSet's pods pass their readiness probe - since the new pods could never even start (bad image), the old pods were left untouched and kept serving.
