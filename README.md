# Playing with K8s Service Accounts

## Creating the files

Let's say the cluster id called `rpi`, the GitHub user is `carnei-ro`, the GitHub repo is `k8s-oidc`.

```bash
CLUSTER_NAME="rpi"
GITHUB_USER="carnei-ro"
GITHUB_REPO="k8s-oidc"
JWKS_URI="https://${GITHUB_USER}.github.io/${GITHUB_REPO}/${CLUSTER_NAME}/openid/v1/jwks"

mkdir -p "./${CLUSTER_NAME}/.well-known"
kubectl get --raw '/.well-known/openid-configuration' | jq --arg jwks_uri "${JWKS_URI}" '.jwks_uri=$jwks_uri' > ./${CLUSTER_NAME}/.well-known/openid-configuration
mkdir -p ./${CLUSTER_NAME}/openid/v1/
kubectl get --raw '/openid/v1/jwks' | jq . > ./${CLUSTER_NAME}/openid/v1/jwks
```

## Serving content

Now go to your GitHub repository and enable GitHub Pages for the `main` branch.
