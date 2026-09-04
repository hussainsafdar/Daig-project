# Phase 5 — CI/CD

## Task 1: Job dependency graph

ci.yml defines 4 jobs. static and test have no dependencies and run in parallel. build and test have no dependencies and run in parallel. build and integration both declare needs: [static, test], so they only start once both static checks and unit tests pass. If either static or test fails, build and integration never run at all - a broken test blocks both image building and full-stack integration testing.

Images are built in the build job (docker/build-push-action) but only pushed to the registry when github.event_name == 'push' - meaning on a pull request the image builds (validating the Dockerfile) but is not published; it is only pushed once code actually lands on a real branch via push. Every build gets tagged with both the immutable commit SHA and, only on the default branch, the mutable latest tag - so latest can never be moved by a feature branch, and rollback always targets a specific SHA rather than latest.

## Task 2: Break a test, show red then green

Locally simulated what CI's test job runs. Baseline: npm test in services/orders passed 3/3 (EX_CONFIG is 78, required() returns a present value, paisa arithmetic uses integers only).

Broke it by changing the assertion in test/health.test.js from assert.strictEqual(EX_CONFIG, 78) to assert.strictEqual(EX_CONFIG, 99). Re-ran npm test: 1 failing, 2 passing, with AssertionError showing "78 !== 99". In real CI this failure in the test job would mean build and integration (both needs: [static, test]) never run - a broken assertion blocks both image publishing and full-stack integration testing before either even starts.

Fixed the assertion back to 78. Re-ran npm test: 3/3 passing again.

## Task 3: Supply-chain hygiene

Every uses: line in ci.yml is pinned to a full commit SHA, not a mutable tag - e.g. uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4, and uses: docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8 # v6 (line references: ci.yml checkout step and ci.yml build-push-action step). The trailing comment records the human-readable version for reference, but the actual pin is the immutable SHA - a tag like @v4 could in principle be moved to point at different code, whereas a commit SHA cannot be reused, which is what prevents a compromised or hijacked action from silently substituting malicious code into the pipeline.

:latest gating is confirmed in the Docker metadata step: type=raw,value=${{ github.sha }} always tags the build with the immutable commit SHA, while type=raw,value=latest,enable={{is_default_branch}} only applies the latest tag when the build is running on the default branch (main) - so a feature or fix branch build can never move :latest.

## Task 4: cd.yml - canary, rollout, rollback

cd.yml implements progressive delivery in four jobs: guard runs first and structurally blocks any deploy during 16:00-21:00 PKT (the iftar peak window) by checking the clock and exiting 1 - the comment explains the intent: "do not ask people to remember, make it structurally impossible." If guard passes, canary shifts 10% of traffic to the new version and watches the SLO for 5 minutes. If canary succeeds, rollout (needs: canary) promotes to 100% traffic and records a deploy marker in Grafana (so incident diagnosis can correlate "what changed, and when"). rollback only runs if: failure() on canary or rollout - it is not part of the normal path, it exists purely as an automatic safety net that reverts traffic to the previous revision.

concurrency: cancel-in-progress: false on this workflow protects against a deploy being cancelled mid-flight - if a rollout were cancelled while, say, 50% of traffic had shifted to the new version, the system would be left in an inconsistent half-migrated state, which is itself a potential outage; refusing to cancel in-progress deploys avoids that.

## Checkpoint answers

Rollback must be one step and must not require a rebuild because if production is actively failing, any rollback path that requires rebuilding an image first extends the outage by however long that build takes (potentially many minutes) - a rollback that is just "shift traffic back to the already-built previous revision" can complete in seconds, which is why cd.yml's rollback job explicitly comments "must not require a rebuild."
