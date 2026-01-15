<img src="https://github.com/user-attachments/assets/44305f56-034b-4a3f-8720-a53f5bbc9ce8" height=230px>

# `kubernetes-configs`
A collection of my cluster configuration spread:
- CI/CD done with ArgoCD
- CNI with Cilium
- Secrets Management with `external-secrets` operator, VaultWarden, and BitWarden CLI
- Custom DNS via AdGuard
- Zero Trust HTTPS routing via Cloudflare Tunnels
- TLS via `cert-manager`

## Noteable Deployments
- **[`mbo`](https://github.com/hiibolt/mbo)** - Mock Trading Monitor built with Databento ([Live Deployment](https://mbo.hiibolt.com))
- **[`socials`](https://github.com/hiibolt/socials)** - Links to my socials ([Live Deployment](https://socials.hiibolt.com))
- **Homepage** - Cluster Monitor / Homepage ([Live Deployment](https://cluster.hiibolt.com))
- **Jellyfin Servarr Stack** - Fully automated media management pipeline
- **Monikai** - Personal assistant capable of reading my calender and Notion to assist in daily workflows
- **ArchiSteamfarm** - Steam platform automation
- **AdGuard** - DNS-level ad filtering and rewrites for all local deployments

## Setting up Cilium
On your host machine, the one where you will be issuing commands from, you'll want to make a patch.yaml:
```bash
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

You'll also want to define a few convenience variables:
```bash
export TALOS_IP="10.0.0.73"
export TALOS_API_PORT="6443"
export CLUSTER_NAME="makko"
```

Next, we'll generate and apply the config (the second command should take a few minutes, monitor the machine's progress before moving on ^^):
```bash
talosctl gen config "$CLUSTER_NAME" "https://$TALOS_IP:$TALOS_API_PORT" --config-patch @patch.yaml
talosctl apply-config --insecure -n "$TALOS_IP" --file controlplane.yaml
```

After completion, we'll bootstrap the cluster and `kubeconfig`:
```bash
talosctl bootstrap --nodes "$TALOS_IP" --endpoints "$TALOS_IP" --talosconfig=./talosconfig
talosctl kubeconfig --nodes "$TALOS_IP" --endpoints "$TALOS_IP" --talosconfig=./talosconfig
```

You can set up endpoints in your `talosconfig` file with the following:
```bash
talosctl --talosconfig=./talosconfig config endpoints $CONTROL_PLANE_IP
```

Then, we'll install apply Cilium's configs:
```bash
kubectl create namespace certificate

helm repo add cilium https://helm.cilium.io/

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_gateways.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_grpcroutes.yaml
```

Then, apply the Cilium `Application` config:
```bash
export cilium_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/hiibolt/kubernetes-configs/refs/heads/main/nuclearbomb/namespaces/kube-system/cilium/application.yaml" | yq eval-all '. | select(.metadata.name == "cilium-application" and .kind == "Application")' -)
export cilium_name=$(echo "$cilium_applicationyaml" | yq eval '.metadata.name' -)
export cilium_chart=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.chart' -)
export cilium_repo=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.repoURL' -)
export cilium_namespace=$(echo "$cilium_applicationyaml" | yq eval '.spec.destination.namespace' -)
export cilium_version=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export cilium_values=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.helm.valuesObject' -)

echo "$cilium_values" | helm template $cilium_name $cilium_chart --repo $cilium_repo --version $cilium_version --namespace $cilium_namespace --values - | kubectl apply --filename -
```

...lastly, set up the other CRDs - namely, the IP pool for Cilium to allocate from:
```bash
kubectl apply -f nuclearbomb/namespaces/kube-system/cilium/crd.yaml
```

## Setting up ArgoCD Cyclically
```bash
kubectl create namespace argocd

export argocd_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/hiibolt/kubernetes-configs/refs/heads/main/nuclearbomb/namespaces/argocd/argocd/application.yaml" | yq eval-all '. | select(.metadata.name == "argocd-application" and .kind == "Application")' -)
export argocd_name=$(echo "$argocd_applicationyaml" | yq eval '.metadata.name' -)
export argocd_chart=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.chart' -)
export argocd_repo=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.repoURL' -)
export argocd_namespace=$(echo "$argocd_applicationyaml" | yq eval '.spec.destination.namespace' -)
export argocd_version=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export argocd_values=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.helm.valuesObject' - | yq eval 'del(.configs.cm)' -)
export argocd_config=$(curl -sL "https://raw.githubusercontent.com/hiibolt/kubernetes-configs/refs/heads/main/nuclearbomb/namespaces/argocd/argocd/appset.yaml" | yq eval-all '. | select(.kind == "AppProject" or .kind == "ApplicationSet")' -)

echo "$argocd_values" | helm template $argocd_name $argocd_chart --repo $argocd_repo --version $argocd_version --namespace $argocd_namespace --values - | kubectl apply --namespace $argocd_namespace --filename -

echo "$argocd_config" | kubectl apply --filename -
```

Then, deploy the service to log in:
```bash
k apply -f nuclearbomb/namespaces/argocd/argocd/service.yaml
ksc argocd
k get svc | grep -i 'argocd-application-server-lb'
```

At least initially, you'll also need the starter password:
```bash
argocd admin initial-password -n argocd
```

## Setting up External Secrets with Vaultwarden
Firstly, sync Vaultwarden, then the `external-secrets` namespace and Helm chart via ArgoCD (takes a sec ^^)

Switch to the namespace and create a (temporary) secret for `bitwarden-cli`. This is necessary because by default it's self-referential and fails.
```bash
ksc external-secrets
k create secret generic bitwarden-cli --from-literal='BW_USER=...' --from-literal='BW_PASSWORD=...'
```

Next, sync `bitwarden-cli`. It'll fail due to the external secret - delete this and the secret:
```bash
k delete es bitwarden-cli
k delete secret bitwarden-cli
```

Then, re-sync the external secret via ArgoCD.

## Setting up Certificates with `cert-manager`
First, sync `certificate-namespace` and `certificate` - **give it about 5 minutes to complete the certificate challenge** ^^

Then, sync `cert-manager`.

## Final Sync
Be sure to update any specific `nodeSelectors` in the `media` workspace, but otherwise, you're good to go! Well done :3
