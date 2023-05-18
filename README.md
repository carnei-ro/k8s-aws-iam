# Playing with K8s Service Accounts and AWS IAM

Table of Contents:

- [tl;dr](#tldr)
- [Exposing the OpenID Connect Discovery and JWKs](#exposing-the-openid-connect-discovery-and-jwks)
  - [Via GitHub Pages](#via-github-pages)
    - [Creating the files](#creating-the-files)
    - [Serving content](#serving-content)
  - [Via Public Kubernetes API](#via-public-kubernetes-api)
- [Kubernetes Configuration](#kubernetes-configuration)
- [K8S AWS Integration](#k8s-aws-integration)
  - [AWS IAM](#aws-iam)
  - [K8S](#k8s)
- [References](#references)

## tl;dr

- This is a POC to use K8s Service Accounts with AWS IAM.
- The "Issuer" endpoint must be public acessible. At leaset the path `/.well-known/openid-configuration` and the endpoint to the JWKs (jwks_uri). It does not mean the K8s Cluster API Server must be public acessible.
- The Service Account Token must have the `iss` claim matching the "Issuer" endpoint. It is possible to configure that by changing the `kube-apiserver` flag `service-account-issuer` in control plane.
- Creating a new `aud` (audience) for the Service Account Token is a good practice.
- Apps must support `AssumeRoleWithWebIdentity` to be seamlessly integrated with AWS IAM.

## Exposing the OpenID Connect Discovery and JWKs

There are many ways to expose the OpenID Connect Discovery and JWKs. You could use a reverse proxy (ingress) to the Kubernetes API, or you could use the Kubernetes API itself, if it is public acessible, or yet, you could gather the information and expose it via GitHub Pages (or any other static content hosting, such as an S3 bucket).

### Via GitHub Pages

#### Creating the files

Let's say the cluster id called `rpi`, the GitHub user is `carnei-ro`, the GitHub repo is `k8s-aws-iam`.

```bash
CLUSTER_NAME="rpi"
GITHUB_USER="carnei-ro"
GITHUB_REPO="k8s-aws-iam"
JWKS_URI="https://${GITHUB_USER}.github.io/${GITHUB_REPO}/${CLUSTER_NAME}/openid/v1/jwks"
ISSUER="${GITHUB_USER}.github.io/${GITHUB_REPO}/${CLUSTER_NAME}"

mkdir -p "./${CLUSTER_NAME}/.well-known"
kubectl get --raw '/.well-known/openid-configuration' | jq --arg jwks_uri "${JWKS_URI}" '.jwks_uri=$jwks_uri' | jq --arg issuer "https://${ISSUER}" '.issuer=$issuer' > ./${CLUSTER_NAME}/.well-known/openid-configuration
mkdir -p ./${CLUSTER_NAME}/openid/v1/
kubectl get --raw '/openid/v1/jwks' | jq . > ./${CLUSTER_NAME}/openid/v1/jwks
```

#### Serving content

Now go to your GitHub repository and enable GitHub Pages for the `main` branch on `/ (root)`.

Add an empty file called `.nojekyll` to the root of the `main` branch.

### Via Public Kubernetes API

Allow public access to the Kubernetes API.

```bash
kubectl create clusterrolebinding oidc-reviewer \
    --clusterrole=system:service-account-issuer-discovery \
    --group=system:unauthenticated
```

## Kubernetes Configuration

Your Kubernetes must be configured to issue Service Account Tokens where the `issuer` is the base endpoint to the OIDC.

If you're using GitHub pages, the `issuer` will be `https://${GITHUB_USER}.github.io/${GITHUB_REPO}/${CLUSTER_NAME}`.

We can do that by changing the `kube-apiserver` flag `service-account-issuer` in control plane. e.g:

```bash
kube-apiserver --service-account-issuer=https://carnei-ro.github.io/k8s-aws-iam/rpi --service-account-key-file=/etc/kubernetes/ssl/sa.pub --service-account-lookup=True --service-account-signing-key-file=/etc/kubernetes/ssl/sa.key # many other flags ....
```

Now, check how the Service Account Token looks like by `exec` into a running pod and getting the token from the mounted file.

```bash
cat /run/secrets/kubernetes.io/serviceaccount/token | cut -d"." -f2 | base64 -d
```

The value of the claim `iss` must match the `issuer` from the OpenID Connect Discovery http endpoint (`/.well-known/openid-configuration`).

## K8S AWS Integration

### AWS IAM

Go to IAM, and creates a new `Identity Provider` with the following settings:

- Provider Type: OpenID Connect
- Provider URL: https://${ISSUER}
- Audience: sts.amazonaws.com

Then create/edit the desired IAM `Role` to include the new `Identity Provider` and add the following `Trust Relationship`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${ISSUER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${ISSUER}:sub": "system:serviceaccount:${K8S_NAMESPACE}:${K8S_SA_NAME}",
          "${ISSUER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### K8S

Edit your pod manifest to include some `env` vars, and `volumeMount` and `volume` for the _service account token_:

```yaml
#...
          env:
            - name: AWS_DEFAULT_REGION
              value: us-east-1
            - name: AWS_REGION
              value: us-east-1
            - name: AWS_ROLE_ARN
              value: arn:aws:iam::${AWS_ACCOUNT_ID}:role/${AWS_ROLE_NAME} # edit as needed
            - name: AWS_STS_REGIONAL_ENDPOINTS
              value: regional
            - name: AWS_WEB_IDENTITY_TOKEN_FILE
              value: /var/run/secrets/amazonaws.com/serviceaccount/token
#...
          volumeMounts:
            - mountPath: /var/run/secrets/amazonaws.com/serviceaccount
              name: &iam_volume_name web-identity
              readOnly: true
#...
      volumes:
        - name: *iam_volume_name
          projected:
            sources:
              - serviceAccountToken:
                  path: token
                  expirationSeconds: 7200
                  audience: sts.amazonaws.com
```

Then it is possible to use the AWS CLI or any other client that uses `AssumeRoleWithWebIdentity` to get temporary credentials.

```bash
aws sts get-caller-identity
```

Note: you could configure to use the `aud` / `audience` as the discovered in the "regular" service account token, but, the pods service account token TTL is generally longer. Creating a token just for AWS integration is a good practice, and it is possible to control the `expirationSeconds`.

## References

- [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) (IAM Roles for Service Accounts)
- [ServiceAccount token volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection)
- [Vault Kubernetes OIDC Provider](https://developer.hashicorp.com/vault/docs/auth/jwt/oidc-providers/kubernetes)
