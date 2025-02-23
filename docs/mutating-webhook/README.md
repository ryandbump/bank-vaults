# Mutating Webhook

The mutating admission webhook injects an executable to containers (in a very non-intrusive way) inside a Deployments/StatefulSets which than can request secrets from Vault through special environment variable definitions. The project is inspired by many, already existing projects (e.g.: `channable/vaultenv`, `hashicorp/envconsul`). The webhook checks if a container has environment variables defined in the following form, and reads the values for those variables directly from Vault during startup time:

```yaml
        env:
        - name: AWS_SECRET_ACCESS_KEY
          value: vault:secret/data/accounts/aws#AWS_SECRET_ACCESS_KEY
# or
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-key-secret
              key: AWS_SECRET_ACCESS_KEY
# or
        - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              configMapKeyRef:
                name: aws-key-configmap
                key: AWS_SECRET_ACCESS_KEY
```

The webhook checks if a container has envFrom and parse defined configmaps and secrets:

```yaml
        envFrom:
          - secretRef:
              name: aws-key-secret
# or
          - configMapRef:
              name: aws-key-configmap
```

Secret and ConfigMap examples:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-key-secret
data:
  AWS_SECRET_ACCESS_KEY: vault:secret/data/accounts/aws#AWS_SECRET_ACCESS_KEY
type: Opaque
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-key-configmap
data:
  AWS_SECRET_ACCESS_KEY: vault:secret/data/accounts/aws#AWS_SECRET_ACCESS_KEY
```

In this case the a init-container will be injected to the given Pod which copies a small binary, called `vault-env` into an in-memory volume and mounts that Volume to all the containers which have an environment variable definition like that. It also changes the `command` of the container to run `vault-env` instead of your application directly. `vault-env` starts up, connects to Vault with (currently with the [Kubernetes Auth method](https://www.vaultproject.io/docs/auth/kubernetes.html) checks the environment variables, and that has a reference to a value stored in Vault (`vault:secret/....`) will be replaced with that value read from the Secret backend, after this `vault-env` immediately executes (with `syscall.Exec()`) your process with the given arguments, replacing itself with that process.

**With this solution none of your Secrets stored in Vault will ever land in Kubernetes Secrets, thus in etcd.**

`vault-env` was designed to work in Kubernetes at the first place, but nothing stops you to use it outside Kubernetes as well. It can be configured with the standard Vault client's [environment variables](https://www.vaultproject.io/docs/commands/#environment-variables) (because there is a standard Go Vault client underneath).

Currently the Kubernetes Service Account based Vault authentication mechanism is used by `vault-env`, so it requests a Vault token based on the Service Account of the container it is injected into. Implementation is ongoing to use [Vault Agent's Auto-Auth](https://www.vaultproject.io/docs/agent/autoauth/index.html) to request tokens in an init-container with all the supported authentication mechanisms.

## Deploying the webhook

### Helm chart

There is a Helm chart available to deploy the [Vault Secrets Webhook](https://github.com/banzaicloud/banzai-charts/tree/master/vault-secrets-webhook). 

```bash
helm init -c
helm repo add banzaicloud-stable http://kubernetes-charts.banzaicloud.com/branch/master
helm upgrade --namespace vswh --install vswh banzaicloud-stable/vault-secrets-webhook
```

For further details follow the webhook's Helm chart [repository](https://github.com/banzaicloud/banzai-charts/tree/master/vault-secrets-webhook).

## Example

Write a secret into Vault:

```bash
vault kv put secret/valami/aws AWS_SECRET_ACCESS_KEY=s3cr3t
```

This deployment will be mutated by the webhook, since it has at least one environment variable having a value which is a reference to a path in Vault:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault
  template:
    metadata:
      labels:
        app: vault
      annotations:
        vault.security.banzaicloud.io/vault-addr: "https://vault:8200"
        vault.security.banzaicloud.io/vault-role: "default"
        vault.security.banzaicloud.io/vault-skip-verify: "true"
        vault.security.banzaicloud.io/vault-path: "kubernetes"
    spec:
      serviceAccountName: default
      containers:
      - name: alpine
        image: alpine
        command: ["sh", "-c", "echo $AWS_SECRET_ACCESS_KEY && echo going to sleep... && sleep 10000"]
        env:
        - name: AWS_SECRET_ACCESS_KEY
          value: vault:secret/data/valami/aws#AWS_SECRET_ACCESS_KEY
```

## Getting secret data from vault and replace it in sercret data

You can mutate secrets as well if you set annotations and define proper vault path in secret data:
```
apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
  annotations:
    vault.security.banzaicloud.io/vault-addr: "https://vault.default.svc.cluster.local:8200"
    vault.security.banzaicloud.io/vault-role: "default"
    vault.security.banzaicloud.io/vault-skip-verify: "true"
    vault.security.banzaicloud.io/vault-path: "kubernetes"
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL2RvY2tlci5pbyI6eyJ1c2VybmFtZSI6InZhdWx0OnNlY3JldC9kYXRhL2RvY2tlcnJlcG8vI0RPQ0tFUl9SRVBPX1VTRVIiLCJwYXNzd29yZCI6InZhdWx0OnNlY3JldC9kYXRhL2RvY2tlcnJlcG8vI0RPQ0tFUl9SRVBPX1BBU1NXT1JEIiwiYXV0aCI6ImRtRjFiSFE2YzJWamNtVjBMMlJoZEdFdlpHOWphMlZ5Y21Wd2J5OGpSRTlEUzBWU1gxSkZVRTlmVlZORlVqcDJZWFZzZERwelpXTnlaWFF2WkdGMFlTOWtiMk5yWlhKeVpYQnZMeU5FVDBOTFJWSmZVa1ZRVDE5UVFWTlRWMDlTUkE9PSJ9fX0=
```

In the example above the secret type is `kubernetes.io/dockerconfigjson` and the webhook can get credentials from vault.
The base64 encoded data contain vault path in case of username and password for docker repository and you can create it with commands:
```
kubectl create secret docker-registry dockerhub --docker-username="vault:secret/data/dockerrepo#DOCKER_REPO_USER" --docker-password="vault:secret/data/dockerrepo#DOCKER_REPO_PASSWORD"
kubectl annotate secret dockerhub vault.security.banzaicloud.io/vault-addr="https://vault.default.svc.cluster.local:8200"
kubectl annotate secret dockerhub vault.security.banzaicloud.io/vault-role="default"
kubectl annotate secret dockerhub vault.security.banzaicloud.io/vault-skip-verify="true"
kubectl annotate secret dockerhub vault.security.banzaicloud.io/vault-path="kubernetes"
```


## Using charts without explicit container.command and container.args

The Webhook is now capable of determining the container's entrypoint and command with the help of image metadata queried from the image registry, this data is cached until the webhook Pod is restarted. If the registry is publicly accessible (without authentication) you don't need to do anything, but if the registry requires authentication the credentials have to be available in the Pod's `imagePullSecrets` section.

Future improvements:
- on AWS and GKE get a credential dynamically with the specific SDK
- query the ServiceAccount's `imagePullSecrets` as well

Some examples:

```bash
helm upgrade --install mysql stable/mysql --set mysqlRootPassword=vault:secret/data/mysql#MYSQL_ROOT_PASSWORD --set-string "podAnnotations.vault\.security\.banzaicloud\.io/vault-skip-verify=true"
```

When using a private image repository:

```bash
# Docker Hub

kubectl create secret docker-registry dockerhub --docker-username=${DOCKER_USERNAME} --docker-password=$DOCKER_PASSWORD

helm upgrade --install mysql stable/mysql --set mysqlRootPassword=vault:secret/data/mysql#MYSQL_ROOT_PASSWORD --set "imagePullSecrets[0].name=dockerhub" --set-string "podAnnotations.vault\.security\.banzaicloud\.io/vault-skip-verify=true" --set image="private-repo/mysql"

# GCR

kubectl create secret docker-registry gcr \
--docker-server=gcr.io \
--docker-username=_json_key \
--docker-password="$(cat ~/json-key-file.json)"

helm upgrade --install mysql stable/mysql --set mysqlRootPassword=vault:secret/data/mysql#MYSQL_ROOT_PASSWORD --set "imagePullSecrets[0].name=gcr" --set-string "podAnnotations.vault\.security\.banzaicloud\.io/vault-skip-verify=true" --set image="gcr.io/your-repo/mysql"

# ECR

TOKEN=`aws ecr --region=eu-west-1 get-authorization-token --output text --query authorizationData[].authorizationToken | base64 --decode | cut -d: -f2`

kubectl create secret docker-registry ecr \
 --docker-server=https://171832738826.dkr.ecr.eu-west-1.amazonaws.com \
 --docker-username=AWS \
 --docker-password="${TOKEN}"

 helm upgrade --install mysql stable/mysql --set mysqlRootPassword=vault:secret/data/mysql#MYSQL_ROOT_PASSWORD --set "imagePullSecrets[0].name=ecr" --set-string "podAnnotations.vault\.security\.banzaicloud\.io/vault-skip-verify=true" --set image="171832738826.dkr.ecr.eu-west-1.amazonaws.com/mysql" --set-string imageTag=5.7
```
