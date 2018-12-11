# Managing Status with Ansible Operator
This is part 2 of 2 continuing with an example. Please see [part
1](https://github.com/dymurray/ao-blogs/blog/master/manage-status.md) before
proceeding.

When containerizing an application in Kubernetes, I found that it was difficult
when to determine an application was truly "Ready" as some applications have a
long start-up time where Kubernetes' liveness and readiness probes don't
provide enough granular support to monitor an application's endpoint looking
for specific data.

I decided to create an example of using an Ansible Operator to deploy my
application and update the Custom Resource's `status` subresource with the
state of an available API endpoint. The application we will use is a
containerized Bitcoin application. While the pod is in a `Running` state, it is
actually performing a long-running syncing process which is downloading a bunch
of data from the blockchain. Since other applications will not be able to use
our node reliably until the sync is complete, I want the Ansible tasks to check
the state of the available JSON-RPC API that the application exposes and update
the CR status so other pods can be aware of the state of the syncing process.

For this blog post, we will cover how to add a task to an operator's Ansible
run that updates the status on a Custom Resource.

## Start with a basic operator
For simplicity, I have provided a basic template of a project to get you
started. To get started, clone the skeleton project:
```
$ git clone https://github.com/dymurray/bitcoin-operator-skeleton
$ cd bitcoin-operator-skeleton
```

This provides us with an Ansible Operator that creates a
`PersistentVolumeClaim`, a `Deployment`, and a `Service` upon creation of a
`Bitd` Custom Resource.

Our task for this blog will be to add a `k8s_status` Ansible task that reports
data from an endpoint on our `Bitd` service and updates the `Bitd`'s Custom
Resource with this data.

### Optional: Create the project from scratch
If you would like to start with a barebones project, the above skeleton can be
created with:
```bash
$ operator-sdk new --type ansible --kind Bitd --api-version bitcoin.example.com/v1 bitcoin-operator
```
## Reporting RPC API Endpoints to Custom Resource Status
### Registering a service
In order to get the data from our endpoint, we need to ensure that our operator
can communicate with the deployed application. Let's look at the tasks in
`roles/Bitd/tasks/main.yml`:
```yaml
---
- name: create pvc
  k8s:
    definition:
      kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: bitcoin
        namespace: "{{ meta.namespace }}"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi # Ideally 200Gi minimum for our application but for an example not needed

# tasks file for Bitd
- name: start bitd
  k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ meta.name }}-bitd'
        namespace: '{{ meta.namespace }}'
      spec:
        selector:
          matchLabels:
            app: bitd
        template:
          metadata:
            labels:
              app: bitd
          spec:
            containers:
            - name: bitd
              image: "docker.io/dymurray/bitcoin-sv-bitd"
              ports:
                - containerPort: 8332
              volumeMounts:
                - name: bitcoin
                  mountPath: /root/.bitcoin
            volumes:
              - name: bitcoin
                persistentVolumeClaim:
                  claimName: bitcoin

- name: create service
  k8s:
    definition:
      kind: Service
      apiVersion: v1
      metadata:
        name: '{{ meta.name }}-bitd'
        namespace: '{{ meta.namespace }}'
      spec:
        type: NodePort
        selector:
          app: bitd
        ports:
        - protocol: TCP
          port: 8332
          targetPort: 8332
          name: rpc
  register: svc
```

We are primarily interested in adding a `uri` Ansible task that talks to our
bitd `Service` and supplies data to our Custom Resource via the `k8s_status`
Ansible module. Looking at our bitd `Service` task, we can see that our service
is talking to port `8332` and also exposing a port on our Kubernetes host to
allow for external traffic to communicate with our application.

Note that we have registered the creation of the service into object `svc` that
we can reference in our `uri` task to get the `clusterIP` and `port` to
communicate with.

### `uri` task to communicate with JSON-RPC API
We want to add a `uri` task that uses the registered `svc` object to get the
proper IP address and port. For simplicity, the skeleton project includes JSON
snippets in `roles/Bitd/files` that can be used as POST data to communicate
with the JSON-RPC API that Bitcoin exposes.

Since we want to add a status field that tells us whether or not the node is
synced, we will use `roles/Bitd/files/getblockchaininfo.json` which gets the
current block count the node is validating along with the total number of block
headers that exist against peers in the network. The `uri` task will look like:
```yaml
- uri:
    url: http://{{ svc.result.spec.clusterIP }}:{{ svc.result.spec.ports[0].port }}
    method: POST
    user: "{{ user }}"
    password: "{{ password }}"
    body: "{{ lookup('file', 'getblockchaininfo.json') }}"
    force_basic_auth: yes
    body_format: json
  register: chaininfo
```

This `uri` task uses the resulting `clusterIP` and `port` of the registered
bitd service to communicate with the JSON-RPC API. It is worth noting that
using these values only works when the Ansible task is run from within a pod on
the Kubernetes cluster. If these tasks are being run locally then the JSON-RPC
API could be contacted by setting `url: http://<kubernetes_cluster_ip>:{{
svc.result.spec.ports[0].nodePort }}`.

The `user` and `password` variables are defined in
`roles/Bitd/defaults/main.yml` which are hardcoded for simplicity in this
example. Ideally, the credentials are passed in as a secret and read from
Ansible.

### Determining if a node is synced
Using the `chaininfo` object, we can set a new fact to determine whether or not
the node is synced or not. To do this, we will use the `set_fact` Ansible
module to determine if the last validated block number equals the block header
count as seen by the node's peers:
```yaml
- set_fact:
    synced: "{{ true if chaininfo.json.result.blocks | int == chaininfo.json.result.headers | int else false }}"
```
This will only set `synced` to `true` when the validated block count equals the
block header count.

### Updating Custom Resource Status
The final piece to this is updating our CR's status subresource using the
`k8s_status` Ansible module. To do this, we get the CR name and namespace from
the `meta` variable that is passed in from Ansible Operator and supply some
custom key/value pairs we wish to append to the status. For this example, we
will supply three values: `synced`, `blockCount`, and `headerCount`. All of
these are values we have already determined, so the task is very simple:
```yaml
- k8s_status:
  api_version: bitcoin.example.com/v1
  kind: Bitd
  name: "{{ meta.name }}"
  namespace: "{{ meta.namespace }}"
  status:
    blockCount: "{{ chaininfo.json.result.blocks }}"
    headerCount: "{{ chaininfo.json.result.headers}}"
    synced: "{{ synced }}"
```

## A final look at the role
Here is what `roles/Bitd/tasks/main.yml` should now look like:
```yaml
---
- name: create pvc
  k8s:
    definition:
      kind: PersistentVolumeClaim
      apiVersion: v1
      metadata:
        name: bitcoin
        namespace: "{{ meta.namespace }}"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi # Ideally 200Gi minimum for this application but not needed for an example

# tasks file for Bitd
- name: start bitd
  k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ meta.name }}-bitd'
        namespace: '{{ meta.namespace }}'
      spec:
        selector:
          matchLabels:
            app: bitd
        template:
          metadata:
            labels:
              app: bitd
          spec:
            containers:
            - name: bitd
              image: "docker.io/dymurray/bitcoin-sv-bitd"
              ports:
                - containerPort: 8332
              volumeMounts:
                - name: bitcoin
                  mountPath: /root/.bitcoin
            volumes:
              - name: bitcoin
                persistentVolumeClaim:
                  claimName: bitcoin

- name: create service
  k8s:
    definition:
      kind: Service
      apiVersion: v1
      metadata:
        name: '{{ meta.name }}-bitd'
        namespace: '{{ meta.namespace }}'
      spec:
        type: NodePort
        selector:
          app: bitd
        ports:
        - protocol: TCP
          port: 8332
          targetPort: 8332
          name: rpc
  register: svc

- uri:
    url: http://{{ svc.result.spec.clusterIP }}:{{ svc.result.spec.ports[0].port }}
    method: POST
    user: "{{ user }}"
    password: "{{ password }}"
    body: "{{ lookup('file', 'getblockchaininfo.json') }}"
    force_basic_auth: yes
    body_format: json
  register: chaininfo

- set_fact:
    synced: "{{ true if chaininfo.json.result.blocks | int == chaininfo.json.result.headers | int else false }}"

- k8s_status:
    api_version: bitcoin.example.com/v1
    kind: Bitd
    name: "{{ meta.name }}"
    namespace: "{{ meta.namespace }}"
    status:
      blockCount: "{{ chaininfo.json.result.blocks }}"
      headerCount: "{{ chaininfo.json.result.headers }}"
      synced: "{{ synced }}"
```

## Build and test the operator
We can now build our image and test it on Kubernetes to verify that the status
is updated during the reconciliation loop of the operator:
```bash
$ operator-sdk build docker.io/dymurray/bitcoin-operator
$ docker push docker.io/dymurray/bitcoin-operator
```

Deploy the operator:
```bash
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
$ kubectl create -f deploy/crds/bitcoin_v1_bitd_crd.yaml
$ kubectl create -f deploy/operator.yaml
```

Create a `Bitd` Custom Resource named `example-bitd`:
```bash
$ kubectl create -f deploy/crds/bitcoin_v1_bitd_cr.yaml
```

Monitor the state of the `example-bitd` CR:
```bash
$ kubectl get bitd example-bitd -o yaml
apiVersion: bitcoin.example.com/v1
kind: Bitd
metadata:
  creationTimestamp: 2018-12-04T19:40:08Z
  generation: 1
  name: example-bitd
  namespace: default
  resourceVersion: "13657"
  selfLink: /apis/bitcoin.example.com/v1/namespaces/default/bitds/example-bitd
  uid: 67d00133-f7fc-11e8-9cdf-5254002f9b39
spec:
  size: 3
status:
  blockCount: "209424"
  conditions:
  - ansibleResult:
      changed: 1
      completion: 2018-12-04T19:46:44.489391
      failures: 0
      ok: 7
      skipped: 0
    lastTransitionTime: 2018-12-04T19:46:14Z
    message: Awaiting next reconciliation
    reason: Successful
    status: "True"
    type: Running
  headerCount: "559417"
  synced: false                            
```

Note above that the `status` subresource has been updated with `blockCount`,
`headerCount`, and `synced` fields. It also has the `conditions` supplied by
Ansible Operator of the most recent Ansible run.

After some time (a few hours for this application), we expect to see `synced`
become `true` when `headerCount` and `blockCount` are equal:
```bash
$ kubectl get bitd example-bitd -o yaml
apiVersion: bitcoin.example.com/v1
kind: Bitd
metadata:
  creationTimestamp: 2018-12-04T19:40:08Z
  generation: 1
  name: example-bitd
  namespace: default
  resourceVersion: "15623"
  selfLink: /apis/bitcoin.example.com/v1/namespaces/default/bitds/example-bitd
  uid: 67d00133-f7fc-11e8-9cdf-5254002f9b39
spec:
  size: 3
status:
  blockCount: "559425"
  conditions:
  - ansibleResult:
      changed: 1
      completion: 2018-12-04T21:42:41.629312
      failures: 0
      ok: 7
      skipped: 0
    lastTransitionTime: 2018-12-04T21:37:12Z
    message: Awaiting next reconciliation
    reason: Successful
    status: "True"
    type: Running
  headerCount: "559425"
  synced: true
```

## Conclusion
This blog post shows you as an operator developer how to update the status
subresource of a Custom Resource from within Ansible. This allows for more
granular control of the Custom Resource and opens up many more possibilities
with Ansible Operator. A way to extend this example would be to have a
dependent application watching the status of `Bitd` and perform it's deployment
when `synced` is set to `true`.

To see an up-to-date maintained version of the Bitcoin Operator, go
[here](https://github.com/dymurray/bitcoin-operator)
