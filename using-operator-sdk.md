# Working with Ansible Operator

## Using Operator SDK to develop operators
I ran into some trouble when I wanted to write some Ansible logic that is
dependent upon communication with an API endpoint on the provisioned
application due to networking. When developing the operator, I found
`operator-sdk up local` to be the most convenient tool to test my changes and
iterate quickly. Because of this, I needed a solution so that I could test the
Ansible logic inside of a containerized operator and so that I could test it
from my machine using `operator-sdk up local`.

There are a number of ways to solve this, but since I use MiniKube as a
development environment, I wanted Ansible to automatically use MiniKube's IP
address when it is detected that we are running the operator locally. If we are
running the operator inside of a pod in the cluster, then I wish to communicate
with the application using the IP address from the Kubernetes `Service`
resource.

Since `minikube` is not installed within the Ansible Operator image, we can
safely assume that if the `minikube` binary is installed, then we are running
`operator-sdk up local` otherwise we can safely communicate with the service
using the `clusterIP`.

### Getting MiniKube IP address
First, we need a task which will determine whether or not MiniKube is installed
and store the IP address off the cluster if it is installed:
```yaml
- name: Register minikube installation status
  command: which minikube
  changed_when: false
  failed_when: false
  register: minikube_installed

- name: Get minikube IP address if installed
  shell: minikube ip
  register: minikube_ip
  when: minikube_installed is not failed
```

Now in order for communication with the Service to work off-cluster, we need to
set the `ServiceType` to `NodePort` when `minikube_ip` is defined. When
`minikube_ip` is not defined, we can assume we are running in a pod on the
cluster and type `ClusterIP` will be sufficient. As an example, we will create
a service that exposes a JSON-RPC API endpoint:
```yaml
- name: Create Service
  k8s:
    definition:
      kind: Service
      apiVersion: v1
      metadata:
        name: '{{ meta.name }}-rcp-api'
        namespace: '{{ meta.namespace }}'
      spec:
        type: "{{ \"NodePort\" if minikube_ip is defined else \"ClusterIP\" }}"
        selector:
          app: rpc-api
        ports:
        - protocol: TCP
          port: 8332
          targetPort: 8332
          name: rpc
  register: rpc_svc
```

We can now set the IP address + Port combination we want to use to talk to the
service. To do this, we want to use `ClusterIP`+`Port` to communicate with the
service when `minikube_ip` is not defined. Alternatively, we want to use
`minikube_ip`+`NodePort` to communicate with the service when `minikube_ip` is
defined.

```yaml
- name: set ip+port combo to use cluster IP
  set_fact:
    ip_port: "{{ rpc_svc.result.spec.clusterIP }}:{{ rpc_svc.result.spec.ports[0].port }}"
  when: minikube_ip is not defined

- name: set ip+port combo to use minikube IP
  set_fact:
    ip_port: "{{ minikube_ip.stdout }}:{{ rpc_svc.result.spec.ports[0].nodePort }}"

- name: Test connection
  uri:
    url: http://{{ ip_port }}
    method: GET
  register: rpc_response

- debug:
    msg: "{{ rpc_response }}"
```

And just like that we have a single set of Ansible tasks that work both inside
of our production operator that runs in a pod on cluster, and for testing
locally with an operator off-cluster using `operator-sdk up local`.
