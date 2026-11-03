<img src="https://github.com/user-attachments/assets/44305f56-034b-4a3f-8720-a53f5bbc9ce8" height=230px>

# Moving onto renkko

Your apps land in the `hiibolt` namespace. Your files land at `/var/mnt/media/hiibolt` on the disk she already uses. Her apps and her `library` and `downloads` stay as they are. Her node config does not change.

The old full cluster is on the `full-cluster` branch.

Push this `main` branch before any of the sync steps. ArgoCD reads GitHub, not your laptop.

## 1. Copy the NAS onto her disk, then unplug it

The apps read `/var/mnt/media/hiibolt/...` on the big disk her node already shares. That is one folder next to her `library` and `downloads`. When the copy finishes, the NAS can go.

Give the NAS a fixed address on her LAN for the copy. From a machine pointed at renkko:

```bash
export NAS=10.0.0.NN
kubectl create namespace hiibolt --dry-run=client -o yaml | kubectl apply -f -
sed "s/NAS_IP/$NAS/" <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nas-copy
  namespace: hiibolt
spec:
  restartPolicy: Never
  containers:
    - name: copy
      image: alpine
      command:
        - sh
        - -c
        - |
          mkdir -p /dest
          for d in media jellyfin jellyseerr sonarr radarr prowlarr transmission; do
            cp -a "/src/$d" /dest/
          done
          echo done
      volumeMounts:
        - name: src
          mountPath: /src
        - name: dest
          mountPath: /dest
  volumes:
    - name: src
      nfs:
        server: NAS_IP
        path: /mnt/meow
    - name: dest
      hostPath:
        path: /var/mnt/media/hiibolt
        type: DirectoryOrCreate
EOF
kubectl -n hiibolt wait --for=jsonpath='{.status.phase}'=Succeeded pod/nas-copy
kubectl -n hiibolt logs nas-copy
kubectl -n hiibolt delete pod nas-copy
```

That copies the old tree straight across:

| On the NAS | After the copy |
| --- | --- |
| `/mnt/meow/media` | `/var/mnt/media/hiibolt/media` |
| `/mnt/meow/jellyfin` | `/var/mnt/media/hiibolt/jellyfin` |
| `/mnt/meow/jellyseerr` | `/var/mnt/media/hiibolt/jellyseerr` |
| `/mnt/meow/sonarr` | `/var/mnt/media/hiibolt/sonarr` |
| `/mnt/meow/radarr` | `/var/mnt/media/hiibolt/radarr` |
| `/mnt/meow/prowlarr` | `/var/mnt/media/hiibolt/prowlarr` |
| `/mnt/meow/transmission` | `/var/mnt/media/hiibolt/transmission` |

Unplug the NAS after `nas-copy` logs `done`. Nothing in the apps points at it anymore, and her node config is unchanged.

## 2. Put the three passwords in her vault

Her cluster reads `vault.senk0.com` through the `bitwarden-login` store. The item ids in this repo are from your old vault, so they will not resolve. Do not apply your old ClusterSecretStore. Same name as hers.

If you need the old values, the vault data is on the NAS at `/mnt/meow/vaultwarden/data`:

```bash
docker run --rm -d --name old-vault -p 8222:80 \
  -v /mnt/meow/vaultwarden/data:/data \
  vaultwarden/server:latest
```

Open `http://127.0.0.1:8222`, copy the tunnel token, the degine database url, and the NordVPN username and password, then `docker stop old-vault`.

Create the three items in her vault:

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

bw get template item | jq \
  --arg user "$NORD_USER" --arg pass "$NORD_PASS" \
  '.type=1 | .name="hiibolt nordvpn" | .login.username=$user | .login.password=$pass' \
  | bw encode | bw create item

bw sync
CF=$(bw get item "hiibolt cloudflared" | jq -r .id)
DEG=$(bw get item "hiibolt degine" | jq -r .id)
NORD=$(bw get item "hiibolt nordvpn" | jq -r .id)
echo "$CF" "$DEG" "$NORD"
```

Paste those ids into `remoteRef.key`:

| Item | File | Field on the item |
| --- | --- | --- |
| `hiibolt cloudflared` | `cloudflared/external-secret.yaml` | password |
| `hiibolt degine` | `degine/external-secret.yaml` | password |
| `hiibolt nordvpn` | `transmission/external-secret.yaml` | username and password, same item, both secrets |

Commit and push.

## 3. Point Cloudflare at the new namespace

In the Cloudflare dashboard, set each public name to the service below. The tunnel token is the secret from step 2. The names themselves are not in this repo.

| Public name | Origin |
| --- | --- |
| `hiibolt.com` | `http://personal-site.hiibolt.svc.cluster.local:4000` |
| `degine.hiibolt.com` | `http://degine-frontend.hiibolt.svc.cluster.local:80` |
| `jellyfin.hiibolt.com` | `http://jellyfin.hiibolt.svc.cluster.local:8096` |
| `jellyseerr.hiibolt.com` | `http://jellyseerr.hiibolt.svc.cluster.local:5055` |
| `sonarr.hiibolt.com` | `http://sonarr.hiibolt.svc.cluster.local:8989` |
| `radarr.hiibolt.com` | `http://radarr.hiibolt.svc.cluster.local:7878` |
| `prowlarr.hiibolt.com` | `http://prowlarr.hiibolt.svc.cluster.local:9696` |
| `transmission.hiibolt.com` | `http://transmission.hiibolt.svc.cluster.local:9091` |

## 4. Add it in her ArgoCD

Her git repo stays as it is. In the ArgoCD UI, click New App.

| Field | Value |
| --- | --- |
| Repository URL | `https://github.com/hiibolt/kubernetes-configs.git` |
| Revision | `main` |
| Path | `.` |
| Cluster | `https://kubernetes.default.svc` |
| Namespace | `argocd` |

Sync that app. It only adds the list, named `hiibolt`. The `hiibolt-*` apps then show up on their own. Her existing apps are not part of that list.

## 5. Sync the apps

Sync every `hiibolt-*` app: personal-site, degine, cloudflared, jellyfin, jellyseerr, sonarr, radarr, prowlarr, flaresolverr, and transmission. On the first one, turn on auto-create namespace so `hiibolt` exists.

## 6. Turn on auto-sync

Still in the UI. Open each app, go to App Details, then Sync Policy, and enable auto-sync. Turn on prune and self-heal too.

Do that on `hiibolt`, and on every `hiibolt-*` app. After that, a push to this repo deploys on its own. Prune removes an app you delete from the repo. Self-heal puts back a hand edit made on the cluster. The list will not flip the switch back off.

## What shows up, and what does not

New on her cluster: the `hiibolt` namespace, those apps, and HTTPRoutes for the `*.hiibolt.com` names on the gateway she already has.

Not touched: her media apps, her Talos config, her git repo, Cilium, cert-manager, Ceph. A new folder, `hiibolt`, is written on the disk she already uses. The gateway will present her `senk0` certificate for your names on the LAN, so a browser at home will warn. Public HTTPS still ends at Cloudflare.

Renkko is ARM. Jellyfin, the linuxserver images, cloudflared, and transmission already publish arm64. `ghcr.io/hiibolt/socials`, `degine-backend`, and `degine-frontend` need an arm64 build or those pods will not start.
