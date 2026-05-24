# bitcoin-shard-listener Helm chart

Helm chart for [bitcoin-shard-listener](https://github.com/lightwebinc/bitcoin-shard-listener) — the IPv6 multicast shard subscriber in the BSV multicast transaction distribution pipeline.

This repository packages templates, default values, JSON Schema validation, and CI workflows for the listener. The application source lives in [`bitcoin-shard-listener`](https://github.com/lightwebinc/bitcoin-shard-listener).

## Install

> The chart references `ghcr.io/lightwebinc/bitcoin-shard-listener:<appVersion>`. Until the image is published from the application repo, `helm install` will succeed but pods will `ImagePullBackOff`.

```bash
# DaemonSet over a labeled set of fabric nodes (recommended)
helm install listener oci://ghcr.io/lightwebinc/charts/bitcoin-shard-listener \
  --version 0.1.0 -n bitcoin-mcast --create-namespace \
  --set workloadType=DaemonSet \
  --set 'nodeSelector.bitcoin-mcast/role=listener' \
  --set config.retryEndpoints='[fd20::24]:9300\,[fd20::25]:9300\,[fd20::26]:9300'

# Single-replica Deployment
helm install listener . -n bitcoin-mcast --create-namespace \
  --set networking.mode=host
```

## Workload type

| `workloadType` | Use case |
|---|---|
| `Deployment` (default) | Small clusters, one or a few listener replicas; `replicaCount` controls quantity. |
| `DaemonSet` | One listener pod per labeled fabric node; recommended for production. |

## Networking modes

Same as the proxy chart — `multus` (default), `host`, or `unicast` (reserved). See the [composition spec](https://github.com/lightwebinc/bitcoin-multicast/blob/main/containerization/composition-spec.md).

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

## Helm test

```bash
helm test listener -n bitcoin-mcast
```

## Release

`release.yml` is gated — `workflow_dispatch` with `confirm: RELEASE` and a `production` GitHub Environment review.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
