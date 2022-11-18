# Vaulted Istio Certificates

## Introduction

This blogpost explains how to avoid using Kubernetes [Secrets](https://kubernetes.io/docs/concepts/configuration/secret) in order to store Istio Certificates. By default secrets are stored in etcd using Base64 encoding, so additional security measures are needed to protect them. One such solution includes storing secrets in an external secret store provider. We will explain how to bootstrap Istio leveraging the `vault-agent-init` container to inject certificates and private key material. This avoids the dependency on Secrets to bootstrap the istio control plane. Exactly the same technique can be used for ingress and egress certificates.

More information on how certificates are used and managed with Istio, can be found in the official documentation:
 - [Identity and certificate management](https://istio.io/latest/docs/concepts/security/#pki)
 - [Plug in CA Certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert)
 - [Custom CA Integration using Kubernetes CSR](https://istio.io/latest/docs/tasks/security/cert-management/custom-ca-k8s)

For best practices based on real-life production scenario's, also check out the folling [Tetrate](https://tetrate.io) blogposts:
 - [Trusting trust: Root Istio’s trust in your existing PKI](https://tetrate.io/blog/istio-trust)
 - [Automate Istio CA rotation in production at scale](https://tetrate.io/blog/automate-istio-ca-rotation-in-production-at-scale)

</br>

### Istiod certificate handling

Although some of the decision logic is explained in the forementioned blogposts, it is good to also take a peak in the [source code](https://github.com/istio/istio/blob/master/pilot/pkg/bootstrap/istio_ca.go) to find some undocumented behavior. 


```go
// From istio/pilot/pkg/bootstrap/istio_ca.go
//
// For backward compat, will preserve support for the "cacerts" Secret used for self-signed certificates.
// It is mounted in the same location, and if found will be used - creating the secret is sufficient, no need for
// extra options.
//
// In old installer, the LocalCertDir is hardcoded to /etc/cacerts and mounted from "cacerts" secret.
//
// Support for signing other root CA has been removed - too dangerous, no clear use case.
//
// Default config, for backward compat with Citadel:
// - if "cacerts" secret exists in istio-system, will be mounted. It may contain an optional "root-cert.pem",
// with additional roots and optional {ca-key, ca-cert, cert-chain}.pem user-provided root CA.
// - if user-provided root CA is not found, the Secret "istio-ca-secret" is used, with ca-cert.pem and ca-key.pem files.
// - if neither is found, istio-ca-secret will be created.
// - a config map "istio-security" with a "caTLSRootCert" file will be used for root cert, and created if needed.
//   The config map was used by node agent - no longer possible to use in sds-agent, but we still save it for
//   backward compat. Will be removed with the node-agent. sds-agent is calling NewCitadelClient directly, using
//   K8S root.
```

Another feature we will leverage is an environment variable (documented [here](https://istio.io/latest/docs/reference/commands/pilot-discovery)) for `istio-pilot` (aka `istiod`), so certificates will be picked up from an alternative location within the Kubernetes POD. This is needed because the `vault-agent-init` injection container will create a new mounted volume `/vault/secrets` to drop the certificates and private key we instrument it to pull from the external vault server. 

| Variable Name | Type   | Default Value | Description                            |
|---------------|--------|---------------|----------------------------------------|
| ROOT_CA_DIR	  | String | /etc/cacerts  | Location of a local or mounted CA root |

</br>

### Pod annotations for vault-agent-init

We will be leveraging vault injector annotations to instruct the sidecar what data to pull and what vault role to use when doing so. We also make sure the `vault-agent-init` container is run before our actual `istiod` main containers, so the latter can pick up the certificates and key material to bootstrap itself correctly. Vault annotations are enumerated and documented [here](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations). The relevant annotations we will be using in this tutorial are the following.

| Annotation | Default Value | Description |
|------------|---------------|-------------|
| `vault.hashicorp.com/agent-inject` | "false" | configures whether injection is explicitly enabled or disabled for a pod. This should be set to a true or false value. |
| `vault.hashicorp.com/agent-init-first` | "false" | configures the pod to run the Vault Agent init container first if true (last if false). This is useful when other init containers need pre-populated secrets. This should be set to a true or false value. |
| `vault.hashicorp.com/role` | - | configures the Vault role used by the Vault Agent auto-auth method. Required when `vault.hashicorp.com/agent-configmap` is not set. |
| `vault.hashicorp.com/auth-path` | - | configures the authentication path for the Kubernetes auth method. Defaults to auth/kubernetes. |
| `vault.hashicorp.com/agent-inject-secret-` | - | configures Vault Agent to retrieve the secrets from Vault required by the container. The name of the secret is any unique string after `vault.hashicorp.com/agent-inject-secret-`, such as `vault.hashicorp.com/agent-inject-secret-foobar`. The value is the path in Vault where the secret is located. |
| `vault.hashicorp.com/agent-inject-template-` | - | configures the template Vault Agent should use for rendering a secret. The name of the template is any unique string after `vault.hashicorp.com/agent-inject-template-`, such as `vault.hashicorp.com/agent-inject-template-foobar`. This should map to the same unique value provided in `vault.hashicorp.com/agent-inject-secret-`. If not provided, a default generic template is used. |

</br>

### Vault server considerations

Vault supports several methods for clients to authenticate themselves. We will be leveraging the [kubernetes auth backend](https://developer.hashicorp.com/vault/docs/auth/kubernetes), which means we will be leveraging kubernetes ServiceAccount JWT token validation. Please note that ServiceAccount tokens are no longer automatically generated since kubernetes 1.24. You can still create those  API tokens manually, as documented [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-an-api-token-for-a-serviceaccount).

As to storage of our certificate and private key material we have 2 options:
 - [PKI Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
 - [KV Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/kv)

Because the PKI secret engine does not provide clean-cut APIs to retrieve the certificates and the private key we need, and because the PKI secret engine will generate a new intermediate certificate for every call (eg every `istiod` restart), we wil be using the generic KV secret engine instead, storing all the values we need in a simple key-value data structure. We will assume the renewal of intermediate certificates is handled out-of-band through some service portal or CI/CD process, that will store the renewed intermediate certificates in vault as well.

Istio's controlplane pods `istiod` need the following files in order to bootstrap its build in CA correctly.

| key | value (PEM encoded) | details |
|-----|---------------------|---------|
| ca-key.pem | CA private key | private key of the intermediate cert, used as root CA for istiod |
| ca-cert.pem | CA public certificate | intermediate cert, used as root CA for istiod |
| root-cert.pem | CA root certificate | the root of trust of our newly generated intermediate cert |
| cert-chain.pem | Full certificate chain | intermediate cert at the top, root cert at the bottom |

</br>
</br>

## Setup

### Prerequisites

Prerequisites in terms of installed software, if you want to follow the local set-up, include:

 - `kubectl` to interact with the kubernetes clusters ([download](https://kubernetes.io/docs/tasks/tools/#kubectl))
 - `helm` to install vault injector and istio charts ([download](https://helm.sh/docs/intro/install))
 - `vault` cli tool to configure the vault server ([download](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-install#install-vault))

If you want a local demo environment, please follow the instructions [here](local-setup.md), which use `docker-compose` to spin up a vault server and two seperate k3s clusters. In case you bring your own kubernetes clusters and externally hosted vault instance, skip ahead to the next section. 

 - `docker-compose` to spin-up a local environment ([download](https://github.com/docker/compose/releases))

In order to progress, we expect the following shell variables to be set according to your environment.

```console
export VAULT_SERVER=<vault server>
export K8S_API_SERVER_1=<kubernetes cluster1 apiserver>
export K8S_API_SERVER_2=<kubernetes cluster2 apiserver>
```

### Vault kubernetes auth backend

As mentioned in the introduction section on [vault server considerations](#vault-server-considerations), we will be using the [kubernetes auth backend](https://developer.hashicorp.com/vault/docs/auth/kubernetes). Since `istiod` will be fetching the certificates and private key material from vault, let's first create the corresponding service accounts in both cluster.

```console
kubectl --kubeconfig kubecfg1.yml create ns istio-system
kubectl --kubeconfig kubecfg2.yml create ns istio-system
kubectl --kubeconfig kubecfg1.yml apply -f istio-sa.yml
kubectl --kubeconfig kubecfg2.yml apply -f istio-sa.yml
```

ServiceAccount, Secret and ClusterRoleBinding as below.

```yaml
  # istio-sa.yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: istiod
    namespace: istio-system
    labels: # added for istio helm installation
      app: istiod
      app.kubernetes.io/managed-by: Helm
      release: istio-istiod
    annotations: # added for istio helm installation
      meta.helm.sh/release-name: istio-istiod
      meta.helm.sh/release-namespace: istio-system
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: istiod
    namespace: istio-system
    annotations:
      kubernetes.io/service-account.name: istiod
  type: kubernetes.io/service-account-token
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: role-tokenreview-binding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:auth-delegator
  subjects:
    - kind: ServiceAccount
      name: istiod
      namespace: istio-system
```

> **NOTE:** We added helm labels and annotations on the `istiod` ServiceAccount in order not to have conflicts with our istio helm deployment later on.

Once the ServiceAccount in both clusters is created, let's store their Secret `token` and `ca.cert` values in an output folder.

```console
mkdir -p ./output
kubectl --kubeconfig kubecfg1.yml get secret -n istio-system istiod -o go-template="{{ .data.token }}" | base64 --decode > output/istiod1.jwt
kubectl --kubeconfig kubecfg1.yml config view --raw --minify --flatten -o jsonpath="{.clusters[].cluster.certificate-authority-data}" | base64 --decode > output/k8sapi-cert1.pem
kubectl --kubeconfig kubecfg2.yml get secret -n istio-system istiod -o go-template="{{ .data.token }}" | base64 --decode > output/istiod2.jwt
kubectl --kubeconfig kubecfg2.yml config view --raw --minify --flatten -o jsonpath="{.clusters[].cluster.certificate-authority-data}" | base64 --decode > output/k8sapi-cert2.pem
```

More information on the detailed content of the kubernetes API certificate and the `istiod` ServiceAccount JWT token can be found [here](output), where we also describe to vault interaction process in more depth in terms of REST API calls made to authenticate and fetch secrets.

Let's now create the necessary vault configuration based on this.

```console
export VAULT_ADDR=http://localhost:8200
vault login root
vault auth enable --path=kubernetes-cluster1 kubernetes
vault auth enable --path=kubernetes-cluster2 kubernetes
vault write auth/kubernetes-cluster1/config \
  kubernetes_host="$K8S_API_SERVER_1" \
  kubernetes_ca_cert=@output/k8sapi-cert1.pem \
  token_reviewer_jwt=`cat output/istiod1.jwt` \
  disable_local_ca_jwt="true"
vault write auth/kubernetes-cluster2/config \
  kubernetes_host="$K8S_API_SERVER_2" \
  kubernetes_ca_cert=@output/k8sapi-cert2.pem \
  token_reviewer_jwt=`cat output/istiod2.jwt` \
  disable_local_ca_jwt="true"
```

> **NOTE:** `VAULT_ADDR` is set to localhost in case you are using the `docker-compose` provided environment. Set this to `$VAULT_SERVER` in case you brought your own vault server.

</br>

### Istio certificates and private key in vault kv secrets

Next we will create a new self-signed root certificate and generate intermediate certificates for both our clusters. We will be using the helper `makefile` scripts provided by upstream istio [here](https://github.com/istio/istio/tree/master/tools/certs).

```console
cd certs
make -f ../certs-gen/Makefile.selfsigned.mk root-ca
make -f ../certs-gen/Makefile.selfsigned.mk istiod-cluster1-cacerts
make -f ../certs-gen/Makefile.selfsigned.mk istiod-cluster2-cacerts
cd ..
```

More details on the actual content and the X509v3 extensions being set, can be found [here](certs). You can fine-tune the certificate generation, by the `Makefile` documentation [here](certs-gen) and correspondig `Makefile` override values.

Let's add the generated certificates and private key into vault kv secrets.

```console
export VAULT_ADDR=http://localhost:8200
vault login root
vault secrets enable -path=kubernetes-cluster1-secrets kv
vault secrets enable -path=kubernetes-cluster2-secrets kv
vault kv put kubernetes-cluster1-secrets/istiod-service/certs \
  ca_key=@certs/istiod-cluster1/ca-key.pem \
  ca_cert=@certs/istiod-cluster1/ca-cert.pem \
  cert_chain=@certs/istiod-cluster1/cert-chain.pem \
  root_cert=@certs/istiod-cluster1/root-cert.pem
vault kv put kubernetes-cluster2-secrets/istiod-service/certs \
  ca_key=@certs/istiod-cluster2/ca-key.pem \
  ca_cert=@certs/istiod-cluster2/ca-cert.pem \
  cert_chain=@certs/istiod-cluster2/cert-chain.pem \
  root_cert=@certs/istiod-cluster2/root-cert.pem
```

Move on by restricting access to those certificates and private key per cluster, bound to the kubernetes `istiod` ServiceAccount based auth backend.

```console
echo 'path "kubernetes-cluster1-secrets/istiod-service/certs" {
  capabilities = ["read"]
}' | vault policy write istiod-certs-cluster1 -
echo 'path "kubernetes-cluster2-secrets/istiod-service/certs" {
  capabilities = ["read"]
}' | vault policy write istiod-certs-cluster2 -
vault write auth/kubernetes-cluster1/role/istiod \
  bound_service_account_names=istiod \
  bound_service_account_namespaces=istio-system \
  policies=istiod-certs-cluster1 \
  ttl=24h
vault write auth/kubernetes-cluster2/role/istiod \
  bound_service_account_names=istiod \
  bound_service_account_namespaces=istio-system \
  policies=istiod-certs-cluster2  \
  ttl=24h
```

</br>

### Deploy vault-injector and istio helm charts

In order to deploy the vault injector, we will be leveraging the official [helm charts](https://github.com/hashicorp/vault-helm) for vault.

```console
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
kubectl --kubeconfig kubecfg1.yml create ns vault
kubectl --kubeconfig kubecfg2.yml create ns vault
helm --kubeconfig kubecfg1.yml install -n vault vault-inject hashicorp/vault --set "injector.externalVaultAddr=$VAULT_SERVER"
helm --kubeconfig kubecfg2.yml install -n vault vault-inject hashicorp/vault --set "injector.externalVaultAddr=$VAULT_SERVER"
kubectl --kubeconfig kubecfg1.yml -n vault get pods
kubectl --kubeconfig kubecfg2.yml -n vault get pods
```

```
  NAME                                           READY   STATUS    RESTARTS   AGE
  vault-inject-agent-injector-5776975795-9vt9w   1/1     Running   0          92s
  NAME                                           READY   STATUS    RESTARTS   AGE
  vault-inject-agent-injector-5776975795-9vjnx   1/1     Running   0          91s
```


To install istio, we will be using the Tetrate Istio Distribution [helm charts](https://github.com/tetratelabs/helm-charts).

```console
helm repo add tetratelabs https://tetratelabs.github.io/helm-charts
helm repo update
helm --kubeconfig kubecfg1.yml install -n istio-system istio-base tetratelabs/base
helm --kubeconfig kubecfg2.yml install -n istio-system istio-base tetratelabs/base
helm --kubeconfig kubecfg1.yml install -n istio-system istio-istiod tetratelabs/istiod --values=./cluster1-values.yaml
helm --kubeconfig kubecfg2.yml install -n istio-system istio-istiod tetratelabs/istiod --values=./cluster2-values.yaml
kubectl --kubeconfig kubecfg1.yml -n istio-system get pods
kubectl --kubeconfig kubecfg2.yml -n istio-system get pods
```

Note how we leverage several istio helm chart value overrides to accomplish our desired goal.
 - inject a pilot pod environment variable `ROOT_CA_DIR` to tell `istiod` where to fetch certificates and private key
 - tell the `vault-agent-init` container to run before `istiod` container, so the secrets are available within the `/vault/secrets` mounted volume
 - instruct the vault injector to fetch secrets based on the correct location and data keys
 - assume the vault `istiod` role while doing so
 - override the default kubernetes `auth-path`, because we have multiple clusters

```yaml
pilot:
  env:
    ROOT_CA_DIR: /vault/secrets
  podAnnotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-init-first: "true"
    vault.hashicorp.com/agent-inject-secret-ca-key.pem: "kubernetes-cluster1-secrets/istiod-service/certs"
    vault.hashicorp.com/agent-inject-template-ca-key.pem: |
        {{- with secret "kubernetes-cluster1-secrets/istiod-service/certs" -}}
        {{ .Data.ca_key }}
        {{ end -}}
    vault.hashicorp.com/agent-inject-secret-ca-cert.pem: "kubernetes-cluster1-secrets/istiod-service/certs"
    vault.hashicorp.com/agent-inject-template-ca-cert.pem: |
        {{- with secret "kubernetes-cluster1-secrets/istiod-service/certs" -}}
        {{ .Data.ca_cert }}
        {{ end -}}
    vault.hashicorp.com/agent-inject-secret-root-cert.pem: "kubernetes-cluster1-secrets/istiod-service/certs"
    vault.hashicorp.com/agent-inject-template-root-cert.pem: |
        {{- with secret "kubernetes-cluster1-secrets/istiod-service/certs" -}}
        {{ .Data.root_cert }}
        {{ end -}}
    vault.hashicorp.com/agent-inject-secret-cert-chain.pem: "kubernetes-cluster1-secrets/istiod-service/certs"
    vault.hashicorp.com/agent-inject-template-cert-chain.pem: |
        {{- with secret "kubernetes-cluster1-secrets/istiod-service/certs" -}}
        {{ .Data.cert_chain }}
        {{ end -}}
    vault.hashicorp.com/role: "istiod"
    vault.hashicorp.com/auth-path: "auth/kubernetes-cluster1"
```

When we look at the `vault-agent-init` container traces, we should see something like this. Our control plane has correctly picked up the vault injected secrets.

```console
kubectl --kubeconfig kubecfg1.yml logs -n istio-system -l app=istiod -c vault-agent-init --tail=-1
```

```
  ==> Vault agent started! Log data will stream in below:

  ==> Vault agent configuration:

                      Cgo: disabled
                Log Level: info
                  Version: Vault v1.12.0, built 2022-10-10T18:14:33Z
              Version Sha: 558abfa75702b5dab4c98e86b802fb9aef43b0eb

  2022-11-18T11:01:21.398Z [INFO]  sink.file: creating file sink
  2022-11-18T11:01:21.398Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
  2022-11-18T11:01:21.398Z [INFO]  template.server: starting template server
  2022-11-18T11:01:21.398Z [INFO]  sink.server: starting sink server
  2022-11-18T11:01:21.398Z [INFO]  auth.handler: starting auth handler
  2022-11-18T11:01:21.398Z [INFO]  auth.handler: authenticating
  2022-11-18T11:01:21.398Z [INFO] (runner) creating new runner (dry: false, once: false)
  2022-11-18T11:01:21.398Z [INFO] (runner) creating watcher
  2022-11-18T11:01:21.402Z [INFO]  auth.handler: authentication successful, sending token to sinks
  2022-11-18T11:01:21.402Z [INFO]  auth.handler: starting renewal process
  2022-11-18T11:01:21.402Z [INFO]  sink.file: token written: path=/home/vault/.vault-token
  2022-11-18T11:01:21.402Z [INFO]  sink.server: sink server stopped
  2022-11-18T11:01:21.402Z [INFO]  sinks finished, exiting
  2022-11-18T11:01:21.402Z [INFO]  template.server: template server received new token
  2022-11-18T11:01:21.402Z [INFO] (runner) stopping
  2022-11-18T11:01:21.402Z [INFO] (runner) creating new runner (dry: false, once: false)
  2022-11-18T11:01:21.402Z [INFO] (runner) creating watcher
  2022-11-18T11:01:21.402Z [INFO] (runner) starting
  2022-11-18T11:01:21.403Z [INFO]  auth.handler: renewed auth token
  2022-11-18T11:01:21.515Z [INFO] (runner) rendered "(dynamic)" => "/vault/secrets/root-cert.pem"
  2022-11-18T11:01:21.515Z [INFO] (runner) rendered "(dynamic)" => "/vault/secrets/ca-cert.pem"
  2022-11-18T11:01:21.515Z [INFO] (runner) rendered "(dynamic)" => "/vault/secrets/cert-chain.pem"
  2022-11-18T11:01:21.516Z [INFO] (runner) rendered "(dynamic)" => "/vault/secrets/ca-key.pem"
  2022-11-18T11:01:21.516Z [INFO] (runner) stopping
  2022-11-18T11:01:21.516Z [INFO]  template.server: template server stopped
  2022-11-18T11:01:21.516Z [INFO] (runner) received finish
  2022-11-18T11:01:21.516Z [INFO]  auth.handler: shutdown triggered, stopping lifetime watcher
  2022-11-18T11:01:21.516Z [INFO]  auth.handler: auth handler stopped
```

When we look at the `discovery` container traces, we should see something like this.

```console
kubectl --kubeconfig kubecfg1.yml logs -n istio-system -l app=istiod -c discovery --tail=-1
```

```
  info	Using istiod file format for signing ca files
  info	Use plugged-in cert at /vault/secrets/ca-key.pem
  info	x509 cert - Issuer: "CN=Intermediate CA,O=Istio,L=istiod-cluster1", Subject: "", SN: 39f67569f10d36a1fc91e9d82156b07d, NotBefore: "2022-11-18T11:11:59Z", NotAfter: "2032-11-15T11:13:59Z"
  info	x509 cert - Issuer: "CN=Root CA,O=Istio", Subject: "CN=Intermediate CA,O=Istio,L=istiod-cluster1", SN: dedf298a147681d6, NotBefore: "2022-11-17T22:01:54Z", NotAfter: "2024-11-16T22:01:54Z"
  info	x509 cert - Issuer: "CN=Root CA,O=Istio", Subject: "CN=Root CA,O=Istio", SN: f5bcd7e89bdb6248, NotBefore: "2022-11-17T22:01:52Z", NotAfter: "2032-11-14T22:01:52Z"
  info	Istiod certificates are reloaded
  info	spiffe	Added 1 certs to trust domain cluster.local in peer cert verifier
```

We can see that our istio control plane has correctly picked up our vault injects certificates and private key. Mission accomplished!


## Conclusion

In this blog we have successfully bootstrapped the istio control plane with external vault stored certificates and private keys. The steps to achieve this included:
 - storing the certificates and private key in per cluster dedicated vault secret mount paths
 - setup kubernetes vault auth backends per cluster, linked to the proper ServiceAccount
 - define a proper role and policy to allow access from the `istiod` ServiceAccount to the vault secrets
 - adjust istio `pilot` bootstrap parameters to
    - inject the `vault-agent-init` sidecars 
    - fetch the correct vault secrets containing our certificates and private key
    - using the right role and auth backend to do so
    - pickup the certificates and private key from the correct vault secret mount path

We can use exaclty the same technique to inject `ingress-gateway` and `egress-gateway` certificates. When creating istio [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings) objects, make sure to point `serverCertificate`, `privateKey` and `caCertificates` to the correct files within the `/vault/secrets` mounted volume. We'll leave this as an exercise for the reader.

By tying our certificate injection to kubernetes ServiceAccount, we have now delegated certificate lifecycle management to an external secret vault. External processes, like a service portal or a CI/CD pipeline, can now be created with dedicated roles and write/update policies, to provide the necessary certificate life-cycle management security.
