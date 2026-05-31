# shard-listener Helm chart

Helm chart for [shard-listener](https://github.com/lightwebinc/shard-listener) — the IPv6 multicast shard subscriber in the BSV multicast transaction distribution pipeline.

This repository packages templates, default values, JSON Schema validation, and CI workflows for the listener. The application source lives in [`shard-listener`](https://github.com/lightwebinc/shard-listener).

## Install

> The chart references `ghcr.io/lightwebinc/shard-listener:<appVersion>`. Until the image is published from the application repo, `helm install` will succeed but pods will `ImagePullBackOff`.

```bash
# DaemonSet over a labeled set of fabric nodes (recommended)
helm install listener oci://ghcr.io/lightwebinc/charts/shard-listener \
  --version 0.1.0 -n bsv-mcast --create-namespace \
  --set workloadType=DaemonSet \
  --set 'nodeSelector.bsv-mcast/role=listener' \
  --set config.retryEndpoints='[fd20::24]:9300\,[fd20::25]:9300\,[fd20::26]:9300'

# Single-replica Deployment
helm install listener . -n bsv-mcast --create-namespace \
  --set networking.mode=host
```

## Workload type

| `workloadType` | Use case |
|---|---|
| `Deployment` (default) | Small clusters, one or a few listener replicas; `replicaCount` controls quantity. |
| `DaemonSet` | One listener pod per labeled fabric node; recommended for production. |

## Networking modes

Same as the proxy chart — `multus` (default), `host`, or `unicast` (reserved). See the [composition spec](https://github.com/lightwebinc/bsv-multicast/blob/main/containerization/composition-spec.md).

## Important constraint — `NUM_WORKERS=1`

The Linux kernel delivers each multicast datagram to **all** sockets in a SO_REUSEPORT group with no load balancing. Multiple listener workers cause N-fold frame duplication. The chart hardcodes `NUM_WORKERS=1` in the rendered Deployment regardless of `config.numWorkers`, and `values.schema.json` rejects any value other than `1`.

## Values reference

See [`values.yaml`](values.yaml). Every flag accepted by the listener binary is exposed under `.config`, including:

- Multicast egress / domain bridging (BRC-128)
- BRC-131 block header retransmission (unicast + multicast)
- BRC-127 subtree group subscriptions
- BRC-132 subtree data caching
- Cross-listener TxID dedup via Redis
- Sender allow/deny CIDR lists
- Beacon-driven retry endpoint discovery (BRC-126)
- SSM (RFC 4607) opt-in: `config.sourceMode=ssm` + per-control-group bootstrap source lists

### SSM (Source-Specific Multicast)

`config.sourceMode` defaults to `asm`. When `ssm`, supply at least one
of `config.ssmBootstrap.{manifest,beacon,subtreeAnnounce}` (DNS names
or IPv6 literals — headless-Service names are the production pattern).
Each list renders to its own `SSM_BOOTSTRAP_*` env var and is resolved
via `shard-common/bootstrap.Resolver` (fail-closed startup; last-good
retention on transient refresh failures). `config.ssmPublishersStatic`
is a lab/CI escape hatch for the data-plane source list; production
must use manifest-driven discovery. Chart validation fails closed when
`sourceMode=ssm` and no bootstrap or static list is provided. See the
[SSM Support Plan](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/SourceSpecificMulticast/ssm-support-plan.md)
for fabric prerequisites (PIM-SSM, MLDv2, raised `mld_max_msf`).

### BRC-137 auto-shard-config (opt-in)

`config.autoShardConfig` exposes the BRC-137 manifest consumer. Off by
default; manual `config.shardBits`/`sourceMode`/`shardInclude` always win.
When `enabled: true` the listener decodes manifests off its beacon socket
and adopts `ShardBits`/`SourceMode` once `pilotQuorum` distinct
authoritative announcers agree for the hysteresis window. With
`shardIncludeFromManifest: true` it additionally joins pilot groups at
runtime (`union(shardInclude, pilot groups)`).

| Key | Env var | Default | Notes |
|-----|---------|---------|-------|
| `enabled` | `MANIFEST_CONSUMER_ENABLED` | `false` | master switch |
| `bootstrap` | `MANIFEST_BOOTSTRAP` | `optional` | `required` fails closed: no data-plane bind until quorum |
| `pilotQuorum` | `PILOT_QUORUM` | `2` | min distinct authoritative announcers |
| `pilotHysteresis` | `PILOT_HYSTERESIS` | `0s` | `0s` ⇒ 2 × AnnounceInterval |
| `shardIncludeFromManifest` | `SHARD_INCLUDE_FROM_MANIFEST` | `false` | additive auto-join to pilot groups |
| `liveResharding` | `LIVE_RESHARDING` | `false` | bridging vs restart-on-adopt |
| `bridgingWindow` | `BRIDGING_WINDOW` | `0s` | `0s` ⇒ honour pilot `TransitionEpoch` |

See the [Automatic Shard Configuration Plan](https://github.com/lightwebinc/bsv-multicast/blob/main/docs/AutoShardConfig/auto-shard-config-plan.md).

## Helm test

```bash
helm test listener -n bsv-mcast
```

## Release

`release.yml` is gated — `workflow_dispatch` with `confirm: RELEASE` and a `production` GitHub Environment review.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
