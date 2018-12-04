# Managing Status with Ansible Operator
When containerizing an application in Kubernetes, I found that it was difficult
when to determine an application was truly "Ready" as some applications have a
long start-up time where Kubernetes' liveness and readiness probes don't
provide enough granular support to monitor an application's endpoint looking
for specific data.

I decided to create an example of using an Ansible Operator to deploy my application and update the Custom Resource's `status` subresource

## Reporting RPC API Endpoints to Custom Resource Status
For this example, I was interested in using the `status` subresource of a Kubernetes Custom Resource to mirror data that is available via API endpoint on an application. For this example, I chose an application that has a long startup time and publishes 

## Ansible Operator's existing status management
Ansible Operator by default provides useful information about the Ansible run
by updating the CR's status subresource with output of the run. This output
includes the number of `changed`, `ok`, and `failed` tasks in the previous run
along with an associated error message if it exists. An example output is shown
below:
```yaml
status:
  conditions:
    - ansibleResult:
      changed: 3
      completion: 2018-12-03T13:45:57.13329
      failures: 1
      ok: 6
      skipped: 0
    lastTransitionTime: 2018-12-03T13:45:57Z
    message: 'Status code was -1 and not [200]: Request failed: <urlopen error [Errno
      113] No route to host>'
    reason: Failed
    status: "True"
    type: Failure
  - lastTransitionTime: 2018-12-03T13:46:13Z
    message: Running reconciliation
    reason: Running
    status: "True"
    type: Running
```

This information is useful, but not incredibly specific. It is often desired
that the application can provide custom key/value pairs to provide a more
granular look at the application's status.  To do this, Ansible Operator has a
built-in Ansible Module [k8s_status][k8s_status_module].

## k8s_status Ansible Module
The [k8s_status][k8s_status_module] Ansible Module allows an operator developer
to update the `status` subresource of a CR from within the Ansible run. The
`k8s_status` module has a field `status` which takes in key/value pairs which
will simply be passed along and put on the CR's status. As a basic example, say
we are developing an Ansible Role to be used within an operator which is
watching `Kind` Foo and `ApiVersion` `app.example.com/v1`. We want to update
the status with key `applicationStatus` and value `Ready` at the end of our
Ansible run.

To do this, we can add an Ansible task using the `k8s_status` module that looks
like:
```
- name: Update Custom Resource status field
  k8s_status:
    api_version: app.example.com/v1
    kind: Foo
    name: "{{ meta.name }}"
    namespace: "{{ meta.namespace }}"
    status:
      applicationStatus: Ready
```

## manageStatus Configuration Option
Ansible Operator gives the developer the ability to specify how the status is
managed inside of `watches.yaml`. In this file, a new field `manageStatus` can
be set which defaults to `true`. When `manageStatus` is set to `true` or left
undefined, Ansible Operator will maintain status updates with the Ansible run's
output as shown above. Note that this does not prevent the user from also
updating the CR status with custom key/value pairs when `manageStatus` is
`true`. When `manageStatus` is `false`, Ansible Operator expects the status to
be maintained soley from within Ansible using the `k8s_status` module and will
not publish the output of the runs to the CR status.

[k8s_status_module]:https://github.com/fabianvf/ansible-k8s-status-module
