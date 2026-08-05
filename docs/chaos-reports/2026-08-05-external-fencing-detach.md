# Chaos Testing Report — Dual-Confirmed External Fencing (EBS Detach)

Date: 2026-08-05.

## Summary

TODO-hardening.md item 7 documented a doc/code mismatch: several docs described external fencing as detach-then-confirm-then-bump, but `pkg/fencing/controller.go` bumped the generation on lease expiry alone, with zero AWS SDK usage anywhere in the package. This implements the documented behavior for real: `ec2:DetachVolume` + polled `ec2:DescribeVolumes` confirmation, generation bumped only once the detach is confirmed. Falls back to the previous single-signal behavior when no detacher is configured (Docker, bare metal, or a node with no instance ID recorded).

New permanent IAM role `etcfs-nodes` (created once via `scripts/infra/fencing-iam.sh`, reused by every cluster `create-infra.sh` provisions — not per-run, not torn down) grants exactly what a node needs: `DetachVolume`/`AttachVolume` scoped to volume+instance ARNs, and `Describe{Volumes,VolumeStatus,Instances,Tags}` (unavoidably unscoped — AWS does not support resource-level permissions for these Describe actions).

**Result: 13/13 unit/integration tests pass. 7/7 on real AWS, third attempt** — the first two runs found and fixed real bugs, not test flakes. See Findings.

## What was verified

| Layer | Coverage |
|---|---|
| `EBSDetacher` unit tests (fake EC2 client) | detach confirms via poll not just the request's return; times out if still attached after `PollTimeout`; a "detaching" state still counts as attached; already-detached is treated as success (fencing retries); other instances' attachments are ignored; empty instance ID rejected; a `DescribeVolumes` error is not treated as proof of detachment |
| `Controller` integration tests (real etcd) | generation bumps only after a confirmed detach; does **not** bump when the detach fails or times out; does **not** bump when no instance ID was recorded (rolling-upgrade case); falls back to single-signal fencing with no detacher configured; the instance ID round-trips correctly through the hand-rolled membership JSON, including the case of an older value with no `instance_id` field at all |
| Real AWS, end to end | node partitioned from etcd via iptables (same technique as `chaos-fencing-detach.sh`'s siblings — see 2026-08-05's other report for why SG swaps don't work here); survivor's controller calls the real EC2 API, the volume is confirmed detached, the generation bumps only afterward, survivors stay writable throughout |

## Findings

### 1. `LoadDefaultConfig` does not query EC2 instance metadata for region unless told to

First AWS run: every `DetachVolume` call failed immediately with `failed to resolve service endpoint ... Missing Region`. Traced into the SDK source (`aws-sdk-go-v2/config@.../resolve.go` and `load_options.go`) rather than guessing: `resolveEC2IMDSRegion` is in the default resolver chain, but `LoadOptions.getEC2IMDSRegion` returns not-found whenever `UseEC2IMDSRegion` is nil — the IMDS region lookup is opt-in via `config.WithEC2IMDSRegion()`, not automatic. The provisioned instances have no `AWS_REGION` set and no shared config file, so region resolution had nothing to fall back to. Fixed by passing `awsconfig.WithEC2IMDSRegion()` explicitly in `NewEBSDetacher`.

### 2. Two controllers can race to fence the same node

Both survivors' logs show them independently detecting n1's lease expiry and both completing a fence — `generation=1 previous=0` on one, `generation=2 previous=1` on the other, moments apart. `activeFences` is an in-memory, per-process dedup map; it has no cross-node coordination, so nothing stops two controllers reacting to the same watch event. Not a safety issue — `BumpGeneration`'s CAS serializes the two attempts correctly, one succeeds at gen 1 and the other's stale `expectedOld` fails and retries at gen 2 — but it means `DetachVolume` gets called twice (harmless — `alreadyDetached` treats the second as success) and is worth noting as inefficiency, not corrected here.

### 3. A test-script bug produced two false failures before the real bug was found

Second AWS run: detach confirmed correctly (logs show `"volume detach confirmed"` and `"node fenced"` from both survivors), but the script's own generation check reported `gen:n1 was never bumped ()`. Traced before accepting it as a product bug, since it contradicted what the daemon logs already proved: the check used `chaos-lib.sh`'s shared `etcdctl_on` helper, which always SSHes via **N1** to run its query, regardless of which node's state is being asked about. N1 is the node under test here — partitioned, unable to reach Raft quorum, so its own local etcd cannot serve a linearizable read. The query didn't error, it hung to its timeout and returned empty, which read exactly like "never bumped." Fixed by having this script query via N2 instead, which is never partitioned in this scenario. `chaos-lib.sh` itself is unchanged — this is specific to a script that partitions N1, which no other current script does.

## Reproduction

```
./scripts/test/chaos-fencing-detach.sh
```

AWS only — there is no EBS Multi-Attach volume to detach in Docker, so the controller there always runs single-signal (no detacher configured), a path already covered by `chaos-elastic-fault-injection.sh`'s FJ2. Requires the `etcfs-nodes` IAM role to exist; `provision_cluster` creates it if missing (idempotent, safe to call every run).
