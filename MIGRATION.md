<img src="https://github.com/user-attachments/assets/44305f56-034b-4a3f-8720-a53f5bbc9ce8" height=230px>

# Moving onto renkko

Your apps land in the `hiibolt` namespace. Nothing in this repo mounts a disk. Her node config does not change.

The old full cluster is on the `full-cluster` branch.

Push this `main` branch before any of the sync steps. ArgoCD reads GitHub, not your laptop.

## 1. Put the two passwords in her vault

Her cluster reads `vault.senk0.com` through the `bitwarden-login` store. The item ids in this repo are from your old vault, so they will not resolve. Do not apply your old ClusterSecretStore. Same name as hers.

If you need the old values, the vault data is on the NAS at `/mnt/meow/vaultwarden/data`:

```bash
docker run --rm -d --name old-vault -p 8222:80 \
  -v /mnt/meow/vaultwarden/data:/data \
  vaultwarden/server:latest
```

Open `http://127.0.0.1:8222`, copy the tunnel token and the degine database url, then `docker stop old-vault`.

Create the two items in her vault:

```bash
export BW_HOST=https://vault.senk0.com
bw login
export BW_SESSION=$(bw unlock --raw)

bw get template item | jq \
  --arg pass "$CF_TUNNEL_TOKEN" \
  '.type=1 | .name="hiibolt cloudflared" | .login.username="cloudflared" | .login.password=$pass' \
  | bw encode | bw create item

bw get template item | jq \
  --arg pass "$DATABASE_URL" \
  '.type=1 | .name="hiibolt degine" | .login.username="degine" | .login.password=$pass' \
  | bw encode | bw create item

bw sync
CF=$(bw get item "hiibolt cloudflared" | jq -r .id)
DEG=$(bw get item "hiibolt degine" | jq -r .id)
echo "$CF" "$DEG"
```

Paste those ids into `remoteRef.key`:

| Item | File | Field on the item |
| --- | --- | --- |
| `hiibolt cloudflared` | `cloudflared/external-secret.yaml` | password |
| `hiibolt degine` | `degine/external-secret.yaml` | password |

Commit and push.

## 2. Point Cloudflare at the new namespace

In the Cloudflare dashboard, set each public name to the service below. The tunnel token is the secret from step 1. The names themselves are not in this repo.

| Public name | Origin |
| --- | --- |
| `hiibolt.com` | `http://personal-site.hiibolt.svc.cluster.local:4000` |
| `degine.hiibolt.com` | `http://degine-frontend.hiibolt.svc.cluster.local:80` |

Media stays on her stack. Copy your files into her library at `/var/mnt/media/library`. The old copies are on the NAS at `/mnt/meow/media`.

## 3. Add it in her ArgoCD

Her git repo stays as it is. In the ArgoCD UI, click New App.

| Field | Value |
| --- | --- |
| Repository URL | `https://github.com/hiibolt/kubernetes-configs.git` |
| Revision | `main` |
| Path | `.` |
| Cluster | `https://kubernetes.default.svc` |
| Namespace | `argocd` |

Sync that app. It only adds the list, named `hiibolt`. The `hiibolt-*` apps then show up on their own. Her existing apps are not part of that list.

## 4. Sync the apps

Sync `hiibolt-personal-site`, `hiibolt-degine`, and `hiibolt-cloudflared`. On the first one, turn on auto-create namespace so `hiibolt` exists.

## 5. Turn on auto-sync

Still in the UI. Open each app, go to App Details, then Sync Policy, and enable auto-sync. Turn on prune and self-heal too.

Do that on `hiibolt`, and on every `hiibolt-*` app. After that, a push to this repo deploys on its own. Prune removes an app you delete from the repo. Self-heal puts back a hand edit made on the cluster. The list will not flip the switch back off.

## What shows up, and what does not

New on her cluster: the `hiibolt` namespace, those apps, and HTTPRoutes for the `*.hiibolt.com` names on the gateway she already has.

Not touched: her media apps, her disk, her Talos config, her git repo, Cilium, cert-manager, Ceph. The gateway will present her `senk0` certificate for your names on the LAN, so a browser at home will warn. Public HTTPS still ends at Cloudflare.

Renkko is ARM. Cloudflared, `ghcr.io/hiibolt/socials`, `degine-backend`, and `degine-frontend` all publish an arm64 build.
