## Traefik Enterprise Deployment

The image containing the fix is available at: `landryb/fix-vault-namespace:latest` 

```bash
kubectl create namespace traefikee
kubectl create secret generic tee-hcp-license --from-literal=license="$TRAEFIKEE_LICENSE" -n traefikee
kubectl create configmap --from-file=static.yaml tee-hcp-static-config -n traefikee
kubectl apply -f manifest.yaml
```
## Peering connections

The peering connections can be easily created. Just follow the guidelines from HCP. I didn't experience any issue with that. 


## Vault as PKI

```bash
export VAULT_ADDR=<vault private or public addr>
export VAULT_NAMESPACE="admin"
export VAULT_TOKEN=<generated on the UI>

vault secrets enable pki
vault write pki/root/generate/internal common_name="My Test CA" ttl=1000h
vault write pki/roles/hcp-role allowed_domains=hcp.demo.traefiklabs.tech allow_subdomains=true allow_bare_domains=true max_ttl=100h
```

## Testing using Vault CI

```bash

export VAULT_NAMESPACE="admin"
export VAULT_TOKEN=<generated on the UI>
curl -H "X-Vault-Token: ${VAULT_TOKEN}" -H "X-Vault-Namespace: ${VAULT_NAMESPACE}"  https://vault-cluster.vault.50108fb6-25db-4763-b37f-16866f03465a.aws.hashicorp.cloud:8200/v1/pki/roles/hcp-role 

```

## Adding Vault related configuration to the Traefik static.configuration

See the file: `static.yaml`. 
The attribute `namespace` has been added by Landry, as it is mandatory field that should sent while connecting to Vault Enterprise. 

```yaml
certificatesResolvers:
  vault-pki:
    vault:
      url: "https://vault-cluster.private.vault.50108fb6-25db-4763-b37f-16866f03465a.aws.hashicorp.cloud:8200"
      token: "s.09xBzqoGs3odPaZprgxrYNTu.qMxGN" # Token should be refreshed
      role: "hcp-role"
      namespace: "admin" # setting namespace is mandatory
```

## Deploying sample application and adding Ingressroute

```bash
kubectl apply -f app/
```

Please note that `certResolver` points to the Vault PKI.

## Testing

```bash
curl -k -vv https://foo.hcp.demo.traefiklabs.tech 
```

## Example Logs from Controller

```
tee-hcp-controller-0 tee-hcp-controller time="2021-10-06T09:39:33Z" level=info msg="Starting provider *vault.PKIProvider {\"ResolverName\":\"mypki\"}"
tee-hcp-controller-0 tee-hcp-controller time="2021-10-06T09:39:33Z" level=info msg="Starting Vault PKI certificate resolver" no_store=false node=tee-hcp-controller-0 role=controller service=provider providerName=mypki.vault ttl=100h0m0s
```

```
tee-hcp-controller-0 tee-hcp-controller time="2021-10-06T10:32:57Z" level=info msg="Resolving certificate" domains=foo.hcp.demo.traefiklab.tech service=provider node=tee-hcp-controller-0 role=controller providerName=mypki.vault
```

```
tee-hcp-controller-0 tee-hcp-controller time="2021-10-06T09:39:33Z" level=info msg="Starting provider *vault.PKIProvider {\"ResolverName\":\"mypki\"}"
tee-hcp-controller-0 tee-hcp-controller time="2021-10-06T09:39:33Z" level=info msg="Starting Vault PKI certificate resolver" no_store=false node=tee-hcp-controller-0 role=controller service=provider providerName=mypki.vault ttl=100h0m0s
```

## Errors from the previous configuration

Here are logs from the controller. The log `permission denied` actually comes from directly from Vault when `namespace` attribute is missing in the statick configuration. 

```bash
tee-hcp-controller-0 tee-hcp-controller time="2021-10-06T09:38:33Z" level=error msg="Unable to create Vault PKI provider" error="unable to get initial role configuration: unable to read Vault role configuration: Error making API request.\n\nURL: GET https://vault-cluster.private.vault.50108fb6-25db-4763-b37f-16866f03465a.aws.hashicorp.cloud:8200/v1/pki/roles/test-role\nCode: 403. Errors:\n\n* 1 error occurred:\n\t* permission denied\n\n"

```

If the request is sent without `X-Vault-Namespace` header the same error is being recieved. 


```bash
❯ curl -H "X-Vault-Token: ${VAULT_TOKEN}"   https://vault-cluster.vault.50108fb6-25db-4763-b37f-16866f03465a.aws.hashicorp.cloud:8200/v1/pki/roles/hcp-role  
```

```json
{
  "errors": [
    "1 error occurred:\n\t* permission denied\n\n"
  ]
}
```
