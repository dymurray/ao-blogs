# The Bitcoin Operator
The Bitcoin Operator is powered by Ansible. Ansible is a simple IT automation
language that allows for idempotent runs of a set of tasks to ensure that the
application is healthy and performing as expected. It allows us to put the
logic for installation, deployment, and lifecycle management all inside of an
operator. An Operator allows us to package, deploy, and manage a Kubernetes
application (Bitcoin).

## Deploying Bitcoin Operator
To deploy the operator, we need to create the relevant RBAC resources along
with the Custom Resource Definition (CRD) of our Bitcoin application.

Create the RBAC resources:
```bash
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
```

Create the CRD:
```bash
$ kubectl create -f deploy/crds/bitcoin_v1_bitd_crd.yaml
```

Next, create the Bitcoin Operator deployment:
```bash
$ kubectl create -f deploy/operator.yaml
```

## Creating RPC Credentials
Bitcoin exposes a [JSON-RPC
API](https://en.bitcoin.it/wiki/API_reference_(JSON-RPC)) that uses Basic
Authentication for communication. To hide the credentials, a Kubernetes user
can create a secret with the user/password combination that the Bitcoin
Operator can read from when starting up the node.

In the namespace where the operator will live, create a secret with the fields
`user` and `password`:
```bash
$ kubectl create secret generic --from-literal=user=<user> --from-literal=password=<password> bitcoin-rpc-credentials
```

## Deploying a Bitcoin instance
Now that the operator is up and running:
```bash
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
bitcoinsv-operator-79cb6bdb87-sm97z   1/1     Running   0          4m
```

and we have our secret:
```bash
$ kubectl get secret bitcoin-rpc-credentials
NAME                      TYPE     DATA   AGE
bitcoin-rpc-credentials   Opaque   2      25m
```

We can create our Bitcoin instance by creating a Kubernetes Custom Resource.
This is included in our repository with some sane defaults:
```bash
$ cat deploy/crds/bitcoin_v1_bitd_cr.yaml
apiVersion: bitcoin.example.com/v1
kind: Bitd
metadata:
  name: example-bitd
spec:
  # Add fields here
  rpc_secret_name: bitcoin-rpc-credentials
  pvc_size: 200Gi
```
*Note:* `pvc_size` is set to 200Gi since Bitcoin stores all of the blockchain
data inside of the persistent volume. If you are interested in just testing
this out and don't have enough space, feel free to decrease this.

Since the secret name is correct, we can simply create this CR as is:
```bash
$ kubectl create -f deploy/crds/bitcoin_v1_bitd_cr.yaml
```

In a few seconds, you should have a Bitcoin instance running in a pod:
```bash
$ kubectl get pods
NAME                                  READY   STATUS              RESTARTS   AGE
bitcoinsv-operator-79cb6bdb87-sm97z   1/1     Running             0          17m
example-bitd-bitd-674bf9fb4f-czmsz    0/1     ContainerCreating   0          1s
```

Once the pod is running, the operator will update the status of the `Bitd` CR
we created where we can monitor it's syncing status:
```bash
$ kubectl get bitd example-bitd -o yaml
apiVersion: bitcoin.example.com/v1
kind: Bitd
metadata:
  creationTimestamp: "2018-12-11T17:32:11Z"
  generation: 1
  name: example-bitd
  namespace: default
  resourceVersion: "2351"
  selfLink: /apis/bitcoin.example.com/v1/namespaces/default/bitds/example-bitd
  uid: b12cdc2e-fd6a-11e8-8d6b-525400c036d2
spec:
  pvc_size: 200Gi
  rpc_secret_name: bitcoin-rpc-credentials
status:
  address: bitcoincash:qp4zfxj230h9dvq48ndmwnwrv79pc29gcvwc89uwuz
  balance: "0.0"
  blockCount: "76181"
  conditions:
  - ansibleResult:
      changed: 1
      completion: 2018-12-11T17:33:04.212035
      failures: 0
      ok: 17
      skipped: 4
    lastTransitionTime: "2018-12-11T17:32:52Z"
    message: Awaiting next reconciliation
    reason: Successful
    status: "True"
    type: Running
  headers: "560381"
  synced: false
  unconfirmedBalance: "0.0"
```

Note that the status of the CR provides us with the syncing status of the node
with the field `synced` and also gives us an address we can use to fund the
node if needed. This is useful because an application can monitor our `Bitd`
resource to detect chain splits.
