## Traefik Enterprise Deployment

kubectl create namespace traefikee
kubectl create secret generic tee-hcp-license --from-literal=license="$TRAEFIKEE_LICENSE" -n traefikee
kubectl create configmap --from-file=static.yaml tee-hcp-static-config -n traefikee
kubectl apply -f manifest.yaml

## Vault as PKI

export VAULT_ADDR=<vault private or public addr>
export VAULT_NAMESPACE="admin"
export VAULT_TOKEN=<generated on the UI>

vault secrets enable pki
vault write pki/root/generate/internal common_name="My Test CA" ttl=1000h
vault write pki/roles/test-role allowed_domains=docker.localhost allow_subdomains=true allow_bare_domains=true max_ttl=100h

# TODO we're missing a policy to access it probably