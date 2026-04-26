# Playforia Minigolf — k3s manifests

Drop-in Kubernetes manifests for deploying the
[browser port of Playforia Minigolf](https://github.com/ilp0/playforia-minigolf-browser-port)
to a k3s cluster behind an external load balancer that terminates TLS.

The image is built and published from the upstream repo's GitHub Actions to
`ghcr.io/ilp0/playforia-minigolf-browser-port:latest` — a single-process
Node WebSocket server that also serves the built web bundle on port 4242.

## What's deployed

| Resource    | Notes                                                         |
| ----------- | ------------------------------------------------------------- |
| Namespace   | `minigolf`                                                    |
| Deployment  | `minigolf`, 1 replica, `Recreate` strategy, non-root, RO root |
| Service     | `minigolf` ClusterIP, port `80` → container `4242`            |
| Ingress     | host `minigolf.0m.fi`, path `/`, ingress class `traefik`      |

## Deploy

```sh
kubectl apply -k .
# or, without kustomize:
kubectl apply -f namespace.yaml -f deployment.yaml -f service.yaml -f ingress.yaml
```

Roll a fresh image (when `:latest` advances on GHCR):

```sh
kubectl -n minigolf rollout restart deployment/minigolf
```

Tail logs:

```sh
kubectl -n minigolf logs -f deployment/minigolf
```

## Analytics events in the log stream

The server emits structured JSON event lines to stdout alongside the existing
`[server] / [connection] / …` diagnostic lines. Each line is a single JSON
object starting with `{` — easy to separate from the bracketed diagnostics.

Event kinds today: `server_start`, `snapshot` (every 60s), `player_login`,
`player_reconnect`, `player_disconnect`, `lobby_join`, `lobby_leave`,
`game_create`, `game_join`, `game_end`. All include `t` (ISO timestamp) and
`evt` (event name).

Filter just the events:

```sh
kubectl -n minigolf logs deployment/minigolf | grep '^{'
```

Population time-series (jq required):

```sh
kubectl -n minigolf logs --since=24h deployment/minigolf \
  | jq -c 'select(.evt=="snapshot") | {t, players, single: .single.lobby + .single.in_game, multi: .multi.lobby + .multi.in_game, daily: .daily.lobby + .daily.in_game}'
```

### Persistence — what's actually configurable here

Container stdout is captured by the kubelet at
`/var/log/pods/minigolf_*/server/0.log` on the node. Defaults are
**~10 MB per file × 5 files** before rotation, and the files are deleted when
the pod is removed. Pod is recreated on every `:latest` rollout, so
out-of-the-box retention is bounded by your rollout cadence.

Retention is a **node-level kubelet flag**, not a Deployment setting. It
cannot be configured via the manifests in this repo. The two practical knobs:

1. **Increase rotation on the k3s host** (one-time, on the cluster):

   ```sh
   # In /etc/systemd/system/k3s.service or wherever k3s is launched, append:
   --kubelet-arg=container-log-max-size=50Mi
   --kubelet-arg=container-log-max-files=20
   # Then: systemctl daemon-reload && systemctl restart k3s
   ```

   Bumps retention to ~1 GB per container before rotation.

2. **Periodic dump to durable storage** (off-cluster, e.g. cron on a workstation):

   ```sh
   kubectl -n minigolf logs --since=24h deployment/minigolf \
     | grep '^{' >> ~/minigolf-events.jsonl
   ```

   Run daily, well within the in-cluster retention window. This sidesteps the
   need for an in-cluster log shipper or PVC.

For a properly durable in-cluster pipeline you'd want a log shipper
(Vector/Fluent-Bit/Promtail) — out of scope for this single-deployment repo.

## Pinning a specific image

Edit `kustomization.yaml`:

```yaml
images:
  - name: ghcr.io/ilp0/playforia-minigolf-browser-port
    newTag: sha-abcd123    # or v0.1.0, etc.
```

The CI workflow tags every build with `sha-<short>`, every tag push with
`v<version>`, and the master tip with `latest`.

## Notes & trade-offs

- **TLS** is terminated upstream — the Ingress listens only on plain HTTP
  port 80. Add a `tls:` section here only if you ever want the cluster to
  terminate.
- **Single replica** is intentional. Lobby state, in-progress games, and
  the daily-challenge room are kept in-memory in the server process; a
  second replica would be a parallel world. WebSocket sticky-session
  affinity isn't configured for the same reason.
- **Per-IP connection cap** is set to `0` (disabled). Behind the cluster
  ingress every client appears as one upstream IP from the server's
  perspective, so the cap would be a global limit. Rate-limit at your LB
  or WAF instead if needed.
- **GHCR visibility**: the package must be public for the cluster to pull
  it without an `imagePullSecret`. On the upstream repo's Packages page,
  set the `playforia-minigolf-browser-port` package visibility to
  **Public** the first time it's published.
- **Read-only root filesystem** is enforced. The server writes nothing to
  disk — all state is in-memory and tracks/assets are baked into the
  image — so this is safe.
- **Chat toggle**: `CHAT_ENABLED` env var on the container. Default `"1"`
  in the manifest; flip to `"0"` (or `"false"`/`"off"`/`"no"`) and roll
  out to drop incoming chat packets server-side. Useful if you don't
  want to moderate. The CLI equivalents are `--chat-disabled` /
  `--chat-enabled`.
