# Using HashiCorp Vault

Vault can be used to store JDBC credentials used by the Tea application, with the credentials being
provided to the application by an in-memory volume mount from a sidecar container. The credentials
are read in using an ExternalCredentialsProviders script configured in server.conf.yaml.

![vault picture](vault-container.png)

## Initial Vault setup

The simplest way to use Vault is to install it in "dev" mode, which is insecure but works for the
purposes of a pipeline demo. See https://developer.hashicorp.com/vault/docs/platform/k8s/helm/run 
for details, with this page being a quick summary. 

Assuming a Kubernetes namespace of "vault", the install is as follows:
```
helm install -n vault vault hashicorp/vault --set "server.dev.enabled=true"
```
and the vault must then be configured. To do this, use `kubectl exec` to get into the Vault container:
```
kubectl exec -i -t -n vault vault-0 sh
```
Assuming the Tea application container is running in the "default" namespace using a service account of 
"default", then the configuration commands once inside the container would be
```
vault kv put secret/tea type=jdbc username=<db2user> password=<db2password>
vault auth enable kubernetes
vault write auth/kubernetes/role/myapp bound_service_account_names=default bound_service_account_namespaces=default policies=app ttl=30s
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF
vault policy write app /home/vault/app-policy.hcl
```

At this point, the vault should contain the correct secret and have correct permissions, so the
Tea application container Deployment can be modified to add the Vault annotations. OpenShift users
should examine the files in [/tekton/os/hashicorp-vault](/tekton/os/hashicorp-vault), while IKS 
users can modify the tea-tekton Deployment to add the following:
```
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-inject-secret-tea: secret/tea
        vault.hashicorp.com/agent-inject-template-tea: |
          {{- with secret "secret/tea" -}}
          type={{ .Data.data.type }}
          username={{ .Data.data.username }}
          password={{ .Data.data.password }}
          {{- end }}
        vault.hashicorp.com/role: myapp
```
where the "role" must match the auth record created in the Vault container.

Once the containers have restarted, the Vault sidecar should connect successfully and provide
credentials in the /vault/secrets directory, with a single file called "tea" that contains the
necessary information:
```
tea-tekton-8497896f95-c9krg:/tmp/maven-output$ cat /vault/secrets/tea
type=jdbc
username=<db2user>
password=<db2password>
```

This file is detected by the [init-creds.sh](/demo-infrastructure/init-creds.sh) startup
script, and the server.conf.yaml file for the server is configured with a 
[script](/demo-infrastructure/read-hashicorp-creds.sh) to load the credentials:
```
Credentials:
  ExternalCredentialsProviders:
    TeaJDBCHashiCorp:
      loadAllCredentialsCommand: '/home/aceuser/ace-server/read-hashicorp-creds.sh'
      loadAllCredentialsFormat: 'yaml'
```
This load happens at startup time, and credentials are not reloaded if they change.

## Notes for non-dev mode

Vault without using dev mode is more complicated; some notes from the experiments:
```
oc new-project vault
helm install vault hashicorp/vault --set "global.enabled=true" --set "global.openshift=true" --set "injector.image.repository=docker.io/hashicorp/vault-k8s" --set "server.image.repository=docker.io/hashicorp/vault"

/ $ export VAULT_ADDR='http://127.0.0.1:8200'
/ $  vault operator init
Unseal Key 1: <redacted>
Unseal Key 2: <redacted>
Unseal Key 3: <redacted>
Unseal Key 4: <redacted>
Unseal Key 5: <redacted>

Initial Root Token: <redacted>

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
/ $ vault operator unseal
Unseal Key (will be hidden): 
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       ef983ea3-8e92-4d55-d954-fa90fdbed923
Version            1.13.1
Build Date         2023-03-23T12:51:35Z
Storage Type       file
HA Enabled         false
/ $ vault operator unseal
Unseal Key (will be hidden): 
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       ef983ea3-8e92-4d55-d954-fa90fdbed923
Version            1.13.1
Build Date         2023-03-23T12:51:35Z
Storage Type       file
HA Enabled         false
/ $ vault operator unseal
Unseal Key (will be hidden): 
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.13.1
Build Date      2023-03-23T12:51:35Z
Storage Type    file
Cluster Name    vault-cluster-a41b0de2
Cluster ID      9660a198-2e26-5fe7-b0ba-8b033e4f0746
HA Enabled      false
/ $ vault login
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                <redacted>
token_accessor       <redacted>
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
/ $ 



/ $ vault secrets enable -path=secret kv
Success! Enabled the kv secrets engine at: secret/
/ $ vault kv put secret/tea type=jdbc username=<db2user> password=<db2password>
Success! Data written to: secret/tea
/ $ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/

/ $ vault write auth/kubernetes/role/myapp bound_service_account_names=app bound_service_account_namespaces=default policies=app ttl=1h
Success! Data written to: auth/kubernetes/role/myapp

vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
  capabilities = ["read"]
}
EOF

vault policy write app /home/vault/app-policy.hcl


https://www.hashicorp.com/blog/injecting-vault-secrets-into-kubernetes-pods-via-a-sidecar

modified by

https://discuss.hashicorp.com/t/what-is-the-default-format-of-vault-agent-template-is-it-in-go-struct-or-pure-key-value-pairs/7486

https://developer.hashicorp.com/vault/docs/platform/k8s/helm/openshift
```
