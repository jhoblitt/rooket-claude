---
name: rooket
description: Use when bringing up, populating, reaching, diagnosing, or tearing down a rooket cluster — a disposable Rook/Ceph cluster on kind; running any `rooket` command or setting `ROOKET_NAME`; running a project's cluster targets such as `make cluster-up` or `make populate`; running kubectl, helm, or ceph against a test cluster; deciding whether to leave a cluster up; or meeting a cluster that will not become ready.
---

# rooket

[rooket](https://github.com/jhoblitt/rooket) stands up disposable Rook
clusters on kind: kind nodes, a local registry, iSCSI-backed disks for real
OSDs, and the rook-ceph charts. A consumer's harness runs one worker with
one OSD, deployed from a released Rook. Everything below works only through
rooket; `rooket <command> --help` is the authority on its flags.

## The one hard rule

Reach a cluster ONLY as `ROOKET_NAME=<name> rooket k ...`. Never the
ambient kubectl context, a bare `kubectl`, `--context`, `--kubeconfig`, an
exported `KUBECONFIG`, or any cluster rooket did not make — not even
read-only verbs like `get`, `logs`, or `describe`. The ambient context
points at a cluster the user cares about, and no command is harmless enough
to send there.

- Set `ROOKET_NAME` on every rooket command, never rely on the name rooket
  derives from the working directory: outside a rook clone it refuses, and
  inside one it names that clone's cluster. Each Bash call is a fresh
  shell, so an `export` does not carry over: prefix the command.
- The other ways in carry the same name: `rooket helm`, the toolbox
  through `rooket k -n rook-ceph exec deploy/rook-ceph-tools -- ceph ...`,
  and `rooket ceph-config --out <dir>` for a host client (host-networked
  clusters only).
- No way in exists without rooket? Say so and leave it unverified.

## Run cluster commands outside the sandbox

A Claude Code sandbox blocks the container engine and keeps
`~/.local/share/rooket` read-only, so every rooket command that touches a
cluster, an image, or state fails inside it. Run them with the sandbox
disabled (`dangerouslyDisableSandbox: true`). That covers a project's
targets that call rooket, too. Lifting the sandbox does
not lift the hard rule.

## Prerequisites

- `rooket` on `PATH` (`go install github.com/jhoblitt/rooket@latest`).
- A container engine: podman (rootful, its system socket running) or
  docker; `--engine` or `ROOKET_ENGINE` picks one.
- `kind`, `kubectl`, `helm` on `PATH`.
- The iSCSI tooling: `targetcli`, `iscsiadm`, `lvm2`.
- Root for the iSCSI target. Creating a cluster's target (its first `up`)
  and removing it (`down --delete-disks`) need root, through a `pkexec`
  prompt you cannot answer. Check `rooket sudoers status` first; when it
  exits non-zero, ask the user to run `rooket sudoers install`. Never run
  it yourself: it authenticates as the user and grants root-equivalent
  rights.

## Bring-up and readiness

```sh
ROOKET_NAME=<name> rooket up --rook-version <vX.Y.Z> --workers 1 --config-dir <dir> --wait
ROOKET_NAME=<name> rooket wait --timeout 30m
rooket list
```

- `up` defaults to three workers; pass `--workers 1`. The shape, the Rook
  version, and the config directory are recorded with the cluster, so a
  later `up`, `deploy`, or `down` needs none of them again.
- `--config-dir` holds the pins: `config.yaml` (its `profiles:`, such as
  `host-network`, from the first `up` on, since Rook cannot move a running
  cluster onto host networking) and `values/<chart>.yaml` (pin Ceph with
  `cephImage.tag` in `values/rook-ceph-cluster.yaml`).
- `wait` (or `up --wait`) blocks until the CephCluster is `Ready`, every OSD
  is up and in, every PG is active and clean, and every CephObjectStore is
  `Ready` with an RGW answering HTTP. HEALTH_WARN is normal on one worker.
  The default timeout is 20 minutes.
- `list` shows every cluster: `NAME`, `LIVE` (the engine it runs under, `-`
  when down), `REGISTRY PORT`, `STATE DIR`.
- State lives in `~/.local/share/rooket/<name>/`: the cluster's kubeconfig
  (rooket never writes `~/.kube/config`), its disk images, its registry
  port, and its recorded shape. Charts are cached in
  `~/.cache/rooket/charts/`.

## Diagnostics

`rooket wait` names every unmet condition at its timeout and prints
`ceph status`, `ceph health detail`, `ceph osd tree`, and every unready
rook-ceph pod; read that first. Then these, each prefixed with
`ROOKET_NAME=<name> timeout 60` so a wedged API server or mon costs a
minute, not the task:

```sh
rooket list
rooket k get pods -A -o wide
rooket k get events -A --sort-by=.lastTimestamp
rooket k -n rook-ceph describe pods
rooket k -n rook-ceph get cephclusters,cephobjectstores -o yaml
rooket k -n rook-ceph logs <pod> --all-containers --prefix   # --previous after a restart
rooket k -n rook-ceph exec deploy/rook-ceph-tools -- ceph --connect-timeout=20 health detail
```

A refusal reading `cluster "<name>" is locked by another rooket (pid N:
argv)` means another rooket command is at work on that cluster: wait, or
use a different name. Never delete the lock file.

## Teardown, or leaving it up

```sh
ROOKET_NAME=<name> rooket down                 # cluster gone; disks, targets, state kept; no root
ROOKET_NAME=<name> rooket down --delete-disks  # also targets, disk images, state dir; needs root
```

Tear down what you brought up once the task no longer needs it, unless the
user wants it kept. A cluster the next task will reuse may stay up: say so
in the reply, naming it and its state. Before reusing or tearing down a live
cluster, decide whether it is in use. rooket records no owning session:
its lock covers only a command in flight, and `rooket list` shows only
whether a cluster runs. A cluster you did not bring up in this session is
someone else's until the user says otherwise.

## Coexistence

- One cluster per name. Pick a name specific to the project and task, and
  one `rooket list` does not already show as someone else's.
- Never act on another session's cluster: no `down`, no `deploy`, no `up`
  against its name, no writes through `rooket k`.
- Never run `rooket down --all`, `rooket prune`, or `--delete-cache`
  without the user asking for that command: they reach every cluster on
  the host, or the `rooket-cache` pull-through cache they all share.

## No benchmarks

Never run a benchmark on a rooket cluster unless the user asks for one. The
host is shared with other clusters and sessions, so the numbers measure the
host as much as the code, and the load disturbs everyone else's runs.

## When a project wraps rooket

Prefer the project's own targets over raw rooket: they carry its pins, its
names, and the checks it relies on. rgw-go's `make cluster-up RELEASE=squid`
runs `rooket up` for the `rgw-go-squid` cluster with its config directory,
then checks the pinned image and writes the host client config; `make
populate RELEASE=squid` loads its data set; `make cluster-down
RELEASE=squid` runs `rooket down --delete-disks`. Read the project's harness
README (rgw-go's is `hack/rooket/README.md`) before driving it, and reach
the cluster by the name it gives.
