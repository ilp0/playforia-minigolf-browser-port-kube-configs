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
