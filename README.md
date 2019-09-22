[vault]:[https://www.vaultproject.io/]
[oci]:[https://cloud.oracle.com/en_US/cloud-infrastructure]
[oke]:[https://cloud.oracle.com/containers/kubernetes-engine]
[oci-signup]:[https://cloud.oracle.com/tryit]
[kubectl]:[https://kubernetes.io/docs/tasks/tools/install-kubectl/]

# Running Hashicorp Vault on [Oracle OKE][oke] (Container Engine for Kubernetes)

## Introduction
Welcome to our introduction to [Vault][vault]! In this work instruction, we'll show you how to get up and running with Vault on OKE - Oracle's managed Kubernetes service running on Oracle Cloud Infrastructure ([OCI][oci]).

First we will introduce you to Hashicorp Vault, a comprehensive tool for securely managing secrets. A secret is anything that you want to tightly control access to, such as API keys, passwords, or certificates. Vault provides a unified interface to any secret, while providing important features such as an API driven framework delivering tight access control, version control, and detailed audit log.

Then we'll take you through a Vault deployment scenario on [Oracle Container Engine for Kubernetes (OKE)][oke], whereby Vault will be integrated with the OKE cluster control plane using the Vault Kubernetes auth method. The Kubernetes auth method can be used facilitate authentication with Vault by using a Kubernetes Service Account Token. This method of authentication leverages native Kubernetes identity and access management, and makes it easy to introduce a Vault token into a Kubernetes Pod.

## About [Vault][vault]

Secret management is one of the core use cases for Vault. Vault enables organisations to secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API. Many organisations have credentials hard coded in source code, littered throughout configuration files and configuration management tools, and stored in plaintext in version control, wikis, and shared volumes. Vault provides a central place to store these credentials, ensuring they are encrypted, access is audit logged, and exposed only to authorized clients.

Vault provides a wide array of features across the secrets management, data protecton, identity-based access, collaboration & operations, and governance and compliance.

Vault tightly controls access to secrets and encryption keys by authenticating against trusted sources of identity such as Active Directory, LDAP, Kubernetes, and cloud platforms. Vault enables fine grained authorization of which users and applications are permitted access to secrets and keys.

## Tutorial Overview

Let’s consider an example of how we can:

- Deploy Vault to our OKE cluster in a highly available, fault tolerant configuration
- Provide our applications deployed to OKE with a simple mechanism to authenticate with Vault using their respective Kubernetes service account tokens

![alt text](images/vault-deployment-scenario-v0.01.png "Vault on OKE deployment topology")

The Vault deployment scenario illustration describes the solution architecture implemented through this work instruction:

 - OKE Kubernetes cluster
   - etcd and Vault Kubernetes operators
   - Vault cluster
     - Vault configured to use Kubernetes auth method
     - Vault Key/Value (KV) store `secret/testapp`
     - Vault role 'testapp' associated with the Kubernetes service account 'testapp' in the 'default' namespace
     - Vault policy 'testapp-kv-crud' associated with 'testapp' role (providing CRUD access to the `secret/testapp` KV store)
   - etcd cluster serving as persistent storage tier for the Vault cluster
   - Test application authenticated to vault (via Kubernetes auth) using the 'testapp' service account
   - Test application creating and reading secrets from the Vault `secret/testapp` KV store

Both the etcd and Vault clusters will be created by their respective Kubernetes operators.
The Vault operator deploys and manages Vault clusters on Kubernetes. Vault instances created by the Vault operator are highly available and support automatic failover and upgrade. For each Vault cluster, the Vault operator will also create an etcd cluster for the storage backend.

_Note: For more information about Kubernetes operators and the Operator Framework, follow [this link](https://coreos.com/blog/introducing-operator-framework)._

High level outline of the steps that will be followed:

1. Deploy the Vault & etcd operators
2. Deploy the Vault & etcd clusters
3. Configure Vault Kubernetes auth
4. Create the Vault Key/Value (KV) store & associated policy for the test application
5. Deploy the test application and authenticate to Vault using Kubernetes auth
6. Create and read secrets from the KV store

### Prerequisites
 - You will need to have deployed your OKE Kubernetes cluster prior to commencing implementation of the deployment scenario. Follow the link to [this tutorial](https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-full/index.html) for guidance on the process.
 - Create a 'kube config' authentication artefact. This will be used later in the tutorial to connect to the OKE cluster. Follow the link to [this tutorial](https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-full/index.html#DownloadthekubeconfigFilefortheCluster) for guidance on the process.
 - Install the Vault CLI on the host from which you will be following this work instruction. Follow the link to [this tutorial](https://learn.hashicorp.com/vault/getting-started/install) for guidance on the process.

#### Clone the 'oke-hashicorp-vault-tutorial' git repository
Clone the oke-hashicorp-vault-tutorial repository:

``` bash
git clone https://github.com/oracle/cloudnative/security/oke-hashicorp-vault-tutorial.git
```

Commands from this point forward will assume that you are in the `../oke-hashicorp-vault-tutorial/` directory.

## Deploying Vault

This section deals with deploying Vault to the OKE cluster.

### Set up RBAC for etcd & Vault operators

As RBAC is enabled in our OKE custer, we need to implement RBAC configuration for the Kubernetes operators.
The Vault operator works in conjunction with the etcd operator to setup a Vault cluster. To do this both the etcd and Vault operators need RBAC permissions to access necessary resources.

For simplicity, this example binds a Role to the 'default' service account in the 'default' namespace.
*Note: For production usage you should consider a specific service account to bind the Role to.*

1. Create the Role and RoleBinding from the RBAC manifest:

``` bash
$ kubectl -n default create -f kubernetes/vault/operator-rbac.yaml
```

### Deploy the etcd operator

The Vault operator utilises the etcd operator to deploy an etcd cluster to be used as the storage backend for each Vault Cluster provisioned.
The following will implement the etcd operator:

``` bash
# Create the etcd operator Custom Resource Definitions (CRD)
$ kubectl -n default create -f kubernetes/vault/etcd-crd.yaml

# Deploy the etcd operator
$ kubectl -n default create -f kubernetes/vault/etcd-operator-deployment.yaml
```

### Deploy the Vault operator

The following will implement the Vault operator:

``` bash
# Create the Vault operator Custom Resource Definitions (CRD)
$ kubectl -n default create -f kubernetes/vault/vault-crd.yaml

# Deploy the Vault operator
$ kubectl -n default create -f kubernetes/vault/vault-operator-deployment.yaml
```

### Deploy a Vault cluster

A Vault cluster can be deployed by creating a VaultService Custom Resource (CR). For each Vault cluster the Vault operator will also create an etcd cluster as the storage backend.

The following will create a Vault CR that deploys a 2 node Vault cluster named 'example' in high availablilty (HA) mode:

``` bash
$ kubectl -n default create -f kubernetes/vault/vault-example-cluster.yaml
```

Vault will now be running in the OKE cluster. This can be validated as follows:

```
$ kubectl -n default get pods

NAME                              READY     STATUS    RESTARTS   AGE
etcd-operator-779446c7d8-6j74d    3/3       Running   0          2m
example-668f9f8f7d-5vl5z          1/2       Running   0          11s
example-668f9f8f7d-mgsdp          1/2       Running   0          11s
example-etcd-knqw5gbchb           1/1       Running   0          46s
example-etcd-lp4bjddqkh           1/1       Running   0          1m
example-etcd-x9mnhjmz24           1/1       Running   0          30s
vault-operator-7dc8b55b4d-5kzqb   1/1       Running   0          1m
```

The return data should be similar to the above:
 - the 'etcd-operator-' and 'vault-operator-' Pods are running
 - the 'example-' etcd and Vault pods are also running

We can check the VaultService Custom Resource status as follows:

```
$ kubectl -n default get vault example -o yaml

apiVersion: vault.security.coreos.com/v1alpha1
kind: VaultService
metadata:
  creationTimestamp: 2019-02-13T10:10:34Z
  generation: 1
  name: example
  namespace: default
  resourceVersion: "12019"
  selfLink: /apis/vault.security.coreos.com/v1alpha1/namespaces/default/vaultservices/example
  uid: 9a0cd50c-2f77-11e9-83d5-0a580aed1c35
spec:
  TLS:
    static:
      clientSecret: example-default-vault-client-tls
      serverSecret: example-default-vault-server-tls
  baseImage: quay.io/coreos/vault
  configMapName: ""
  nodes: 2
  version: 0.9.1-0
status:
  clientPort: 8200
  initialized: false
  phase: Running
  serviceName: example
  updatedNodes:
  - example-668f9f8f7d-5vl5z
  - example-668f9f8f7d-mgsdp
  vaultStatus:
    active: ""
    sealed:
    - example-668f9f8f7d-5vl5z
    - example-668f9f8f7d-mgsdp
    standby: null
```

The return data should be similar to the above:
  - the Vault cluster is currently running, uninitialized, and in a sealed state.

## Configuring Vault

Now that we have our Vault cluster up and running on OKE, this next section deals with unsealing, initializing, and configuring the Vault cluster.

### Configure accounts and RBAC

Let's create a dedicated service account 'vault-tokenreview' for Vault Kubernetes auth, and then provide the serviceaccount with the rights to perform delegated authentication and authorization checks (`system:auth-delegator` role):

``` bash
$ kubectl -n default create serviceaccount vault-tokenreview
$ kubectl -n default create -f kubernetes/vault/vault-tokenreview-rbac.yaml
```

### Connect to Vault

In order to connect to Vault, we will first configure port forwarding between the local machine and the first sealed Vault node. In a terminal window issue the following command:

``` bash
$ kubectl -n default get vault example -o jsonpath='{.status.vaultStatus.sealed[0]}' | xargs -0 -I {} kubectl -n default port-forward {} 8200
```

Next, open a new terminal and export the following environment for the Vault CLI client:

``` bash
$ export VAULT_ADDR='https://localhost:8200'
$ export VAULT_SKIP_VERIFY="true"
```

Verify that the Vault server is accessible using the `vault status` command:

``` bash
$ vault status

Error checking seal status: Error making API request.

URL: GET https://localhost:8200/v1/sys/seal-status
Code: 400. Errors:

* server is not yet initialized
```

The response confirms that the Vault CLI is successfully interacting with the Vault server. However, the output indicates that the Vault server is not initialized.

### Initialize the Vault cluster

When you first setup a Vault server, you need to begin by initializing it.
Initialization is the process configuring the Vault. This only happens once when the server is started against a new backend that has never been used with Vault before. When running in HA mode, this happens once per cluster, not per instance.
During initialization the encryption keys are generated, unseal keys are created, and the initial root token is setup.

To initialize Vault use the Vault CLI command `vault operator init`. This is an unauthenticated request, so it only works on brand new Vaults with no data:

```
$ vault operator init

Unseal Key 1: Hmw5YKMZyKj1JM2t0pxEthgGGU+5PKjOV1KKLZw0Qhcy
Unseal Key 2: FThbg/t4rznUnk/OtTahIIQRlqsQJp2SXvcA0neeAoCl
Unseal Key 3: 2qYJcMQ9x0trtg4bc/216d0xljwkJn6iWwfYXXcrgHhx
Unseal Key 4: UlbYndrXB6kbGti8LzhLDvj+IWRRyWhBjw0mgxT1uaMb
Unseal Key 5: ivzgZ6m6GK4s4T9wDymZK9fyVnkcQ/Z/YbSZuA6LJ9fh

Initial Root Token: 645655db-3aec-36e1-2cb9-c8eb2396c163

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

Initialization outputs two essential pieces of information:
 - the unseal keys
 - the initial root token

This is the only time that all of this data is known by Vault, and also the only time that the unseal keys should ever be so close together.
For the purpose of this work instruction - save all of these keys somewhere handy, and continue.

### Unseal the Vault and authenticate

Every initialized Vault server starts in the sealed state. From the configuration, Vault can access the physical storage, but it can't read any of it because it doesn't know how to decrypt it. The process of enabling Vault to decrypt the data is known as unsealing the Vault.

Unsealing has to happen every time Vault starts. It can be done via the API and via the command line. To unseal the Vault, you must have the threshold number of unseal keys. In the output above, notice that the 'key threshold' is 3. This means that to unseal the Vault, you need 3 of the 5 keys that were generated.

Begin unsealing the Vault by issuing the following command:

```
$ vault operator unseal

Unseal Key (will be hidden):
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       bbf64780-c7ef-d920-02e7-4363906f668f
Version            0.9.1
HA Enabled         true
```

After pasting in a valid key and confirming, you'll see that the Vault is still sealed, but progress is made.
Continue with `vault operator unseal` to complete unsealing the Vault. To unseal the vault you must use three different keys. When the value for 'Sealed' changes to false, the Vault is unsealed.

Finally, authenticate as the initial root token (it was included in the output with the unseal keys):

```
$ vault login 645655db-3aec-36e1-2cb9-c8eb2396c163

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                faa21577-f039-07f2-aafb-373e3978765b
token_accessor       e4398d13-013d-046c-b4ee-251ffd95f805
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

Before moving on to configure the Kubernetes auth method, let's create a Vault policy called 'testapp-kv-crud' which we will enable for Kubernetes auth in the following steps. The policy will be applied to a KV store (`secret/testapp/*`) where our test secrets will reside.

``` bash
# Create a policy file, testapp-kv-crud.hcl
$ tee testapp-kv-crud.hcl <<EOF
  path "secret/testapp/*" {
       capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

# Create a policy named testapp-kv-crud
$ vault policy write testapp-kv-crud testapp-kv-crud.hcl
```

### Enable the Kubernetes auth method

Now we enable the Kubernetes auth method in Vault at the default path:

```
$ vault auth enable kubernetes
```

Next, let's export the token for the 'vault-tokenreview' service account, which will be used in the Vault Kubernetes auth configuration:

``` bash
# Set VAULT_SA_NAME to the service account created previously
$ export VAULT_SA_NAME=$(kubectl get sa vault-tokenreview -o jsonpath="{.secrets[*]['name']}")

# Set SA_JWT_TOKEN value to the service account JWT used to access the TokenReview API
$ export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)

# Set SA_CA_CRT to the PEM encoded CA cert used to talk to the Kubernetes API
$ export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
```

Now we configure Vault with the Kubernetes master server URL, token reviewer JWT, and certificate authority data for communicating with the OKE control plane:

``` bash
$ vault write auth/kubernetes/config \
      token_reviewer_jwt="$SA_JWT_TOKEN" \
      kubernetes_host="https://kubernetes.default.svc:443" \
      kubernetes_ca_cert="$SA_CA_CRT"
```

And finally, create a Kubernetes service account & Vault role named 'testapp' mapping the Kubernetes service account to the Vault policy testapp-kv-crud and setting the default Vault token TTL to 4h:

``` bash
$ kubectl -n default create serviceaccount testapp
$ vault write auth/kubernetes/role/testapp \
      bound_service_account_names=testapp \
      bound_service_account_namespaces=default \
      policies=testapp-kv-crud \
      ttl=4h
```

That's it! Your Vault deployment is running and configured for Kubernetes auth.
Pods running in the context of the 'testapp' Kubernetes service account will be able to authenticate to Vault using the Kubernetes service account token, which will provide access to Vault resources associated with the 'testapp-kv-crud' policy.
Per the previous configuration steps - implemeted policy file 'testapp-kv-crud.hcl' grants CRUD access to the Vault KV secrets engine at `/secret/testapp/*`.

## Testing the Vault Kubernetes auth method

In order to explore the Kubernetes auth method on OKE, the following creates a Kubernetes deployment 'testapp' which will be authenticated to Vault using Kubernetes auth, then walks through the creation and reading of secrets from a Vault KV store.

First, create and connect to Pod with a container running alpine:3.7 image:

``` bash
kubectl run testapp --rm -i --tty --serviceaccount=testapp --image alpine:3.7
```

Once you are inside the container, install the Vault CLI:

```
/# wget https://releases.hashicorp.com/vault/1.0.2/vault_1.0.2_linux_amd64.zip
/# unzip vault_1.0.2_linux_amd64.zip -d /usr/local/bin/
```

*Note: The following interactions can also be performed directly using the Vault API via cURL.*

Set the `VAULT_ADDR` environment variable to point to the running Vault where you configured Kubernetes auth method, and test the connection:

```
/# export VAULT_ADDR=https://example.default.svc:8200
/# export VAULT_SKIP_VERIFY="true"
/# vault status

Key             Value
---             -----
Seal Type       shamir
Initialized     false
Sealed          false
Total Shares    5
Threshold       3
Version         0.9.1
Cluster Name    vault-cluster-05b1e306
Cluster ID      9d8e752c-d60e-b467-c19c-1b61330baace
HA Enabled      true
HA Cluster      https://example.default.svc:8201
HA Mode         active
```

Set `KUBE_TOKEN` to the service account token value (default/testapp):

```
/# export KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
/# echo $KUBE_TOKEN

eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InRlc3RhcHAtdG9rZW4tcjVwYnAiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidGVzdGFwcCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjI0MWU4ZWYxLTJmN2EtMTFlOS04M2Q1LTBhNTgwYWVkMWMzNSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnRlc3RhcHAifQ.eJvOr8wI-LwgV3uVugRQ0qR-5h3Ap59Ynf709DIQhL7AtLmS78oPGFq5cBI1biSv-DD4wjXHzTVbR1B-v2kQz87Hxz2I5R8-oO4FFPCYM3_8yI00DfsCIN34G1i7pLayHxHFpxkPiDAGHnodlHLboFhVDGZ00eIK0KY_6XDI1Lrz1vc5RDnX8yB89mO5nxXNNjJq1Ae0-0fFBZ52hA1lQ4TJ1HNWs0-ctCAdczFcA6Z96qvHO-HeHLSISjc9UHscbq--eZqXz64deFQWlCfhPnCitiZRBKCITcGiA5do_G_Uhu23kRya6-Z72P8IozBCJhmTLHp4AROgIIS26HpAUQ
```

Now authenticate to Vault using the 'testapp' service account token:

```
/# vault write auth/kubernetes/login role=testapp jwt=${KUBE_TOKEN}

Key                                       Value
---                                       -----
token                                     cc74f359-6c2e-6061-e56e-07d616a63836
token_accessor                            333984d4-b06c-5be9-3b64-55940797f19f
token_duration                            4h
token_renewable                           true
token_policies                            []
identity_policies                         []
policies                                  ["default" "testapp-kv-crud"]
token_meta_service_account_uid            241e8ef1-2f7a-11e9-83d5-0a580aed1c35
token_meta_role                           testapp
token_meta_service_account_name           testapp
token_meta_service_account_namespace      default
token_meta_service_account_secret_name    testapp-token-r5pbp
```

Notice from the return data that the Vault token is successfully generated, and the 'testapp-kv-crud' policy is associated with the token. The metadata reflects that its service account name ('token_meta_service_account_name') is testapp.

Login using the Vault token provided via the Kubernetes auth in the previous step:

```
/# vault login cc74f359-6c2e-6061-e56e-07d616a63836

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                                       Value
---                                       -----
token                                     2fefa26c-06da-33b5-0e91-e91f8d81ae88
token_accessor                            3e14e77a-498e-0089-33d4-d5ae96b4b93a
token_duration                            3h57m18s
token_renewable                           true
token_policies                            ["default" "testapp-kv-crud"]
identity_policies                         []
policies                                  ["default" "testapp-kv-crud"]
token_meta_role                           testapp
token_meta_service_account_name           testapp
token_meta_service_account_namespace      default
token_meta_service_account_secret_name    testapp-token-r5pbp
token_meta_service_account_uid            241e8ef1-2f7a-11e9-83d5-0a580aed1c35
```

### Create and retreive secret in the KV secret/testapp path

The Vault `write` command writes data to Vault at the given path. The data can be credentials, secrets, configuration, or arbitrary data. The specific behavior of this command is determined at the resource mounted at the path. Data is specified as "key=value" pairs.

To persist data in the KV secrets engine at the path `secret/testapp/*` to which we are delegated CRUD access:

```
/# vault write secret/testapp/foo value=bar

Key              Value
---              -----
created_time     2018-12-21T06:01:26.972484174Z
deletion_time    n/a
destroyed        false
version          1
```

The Vault `read` command reads data from Vault at the given path. This can be used to read secrets, generate dynamic credentials, get configuration details, and more.

To read back our secret foo at the path `secret/testapp/foo`:

```
/# vault read secret/testapp/foo

Key                 Value
---                 -----
refresh_interval    768h
value               bar
```

## Conclusion

In our example deployment scenario we interact manually between our 'testapp' deployment and the Vault KV store for the purposes of demonstration. In a production deployment, it's possible to make the process of authenticating to Vault more automated by using the [Vault Agent](https://www.vaultproject.io/docs/agent/).

Vault Agent provides a number of different helper features, specifically addressing automatic authentication, secure delivery/storage of tokens, & lifecycle management of Vault tokens (renewal & re-authentication).
Hashicorp provide a [work instruction](https://learn.hashicorp.com/vault/developer/vault-agent-k8s#step-4-leverage-vault-agent-auto-auth) which describes the process for configuring a Kubernetes deployment which leverages Vault Agent to automatically authenticate with Vault via Kubernetes auth, and retrieve a client token.
