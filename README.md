<img src="https://github.com/user-attachments/assets/44305f56-034b-4a3f-8720-a53f5bbc9ce8" height=230px>

# `kubernetes-configs`

The bit of my homelab that lives on [renkko](https://github.com/senk02/kubernetes-configs), senk's ARM cluster. Each app is a folder at the root of this repo. ArgoCD on her cluster syncs them into the `hiibolt` namespace.

Renkko already runs Cilium, ArgoCD, cert-manager, external-secrets, and the gateway. This repo does not install any of that.

The old full cluster (Talos, Cilium, Vaultwarden, monitoring, the rest) is on the [`full-cluster`](https://github.com/hiibolt/kubernetes-configs/tree/full-cluster) branch.

Moving in is written up in [MIGRATION.md](MIGRATION.md).

## What runs here

- Automatic GitHub deploys via her ArgoCD
- Secrets from her Vaultwarden, through the `external-secrets` operator she already runs
- Public HTTPS through a Cloudflare Tunnel
- LAN hostnames through her existing Cilium gateway

## Notable deployments

- **[`personal-site`](https://github.com/hiibolt/personal-site)** / [`socials`](https://github.com/hiibolt/socials). My site and links ([hiibolt.com](https://hiibolt.com))
- **[`degine`](https://github.com/hiibolt/degine)** ([degine.hiibolt.com](https://degine.hiibolt.com))
- **Jellyfin servarr stack.** Jellyfin, Jellyseerr, Sonarr, Radarr, Prowlarr, FlareSolverr, and Transmission, with the library on the NAS
- **Cloudflared.** The tunnel that publishes the names above

## How a folder becomes an app

In her ArgoCD UI, one new app points at this repo. That app is `appset.yaml`, which makes one app per top-level folder (`hiibolt-jellyfin`, `hiibolt-degine`, and so on) and drops it in the `hiibolt` namespace. Auto-sync is a switch on each app in that same UI. Her repo is not changed.

Her cluster is ARM. The public images already publish an arm64 build. `ghcr.io/hiibolt/socials`, `degine-backend`, and `degine-frontend` need one too, or those pods will not start.

## Where the data lives

Files live at `/var/mnt/media/hiibolt` on the disk renkko already mounts. Her `library` and `downloads` stay beside that folder. The copy, and how to put the three passwords in her vault, is in [MIGRATION.md](MIGRATION.md).
