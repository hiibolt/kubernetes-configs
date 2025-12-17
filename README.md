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

## Setting up ArgoCD Cyclically
```bash
export argocd_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/hiibolt/kubernetes-configs/refs/heads/main/nuclearbomb/argocd/argocd/application.yaml" | yq eval-all '. | select(.metadata.name == "argocd-application" and .kind == "Application")' -)
export argocd_name=$(echo "$argocd_applicationyaml" | yq eval '.metadata.name' -)
export argocd_chart=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.chart' -)
export argocd_repo=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.repoURL' -)
export argocd_namespace=$(echo "$argocd_applicationyaml" | yq eval '.spec.destination.namespace' -)
export argocd_version=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export argocd_values=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.helm.valuesObject' - | yq eval 'del(.configs.cm)' -)
export argocd_config=$(curl -sL "https://raw.githubusercontent.com/hiibolt/kubernetes-configs/refs/heads/main/nuclearbomb/argocd/argocd/appset.yaml" | yq eval-all '. | select(.kind == "AppProject" or .kind == "ApplicationSet")' -)

echo "$argocd_values" | helm template $argocd_name $argocd_chart --repo $argocd_repo --version $argocd_version --namespace $argocd_namespace --values - | kubectl apply --namespace $argocd_namespace --filename -

echo "$argocd_config" | kubectl apply --filename -
```
