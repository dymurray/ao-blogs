# Managing Status with Ansible Operator
In this blog post we will show how you, a developer of an Operator, now have
several options for managing the status of your Custom Resources when using
[Ansible
Operator](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md)

In v1.11.0 of Kubernetes, Custom Resources (CRs) have a mutable `status` object
in the CR's subresources field. With this subresource, Ansible Operator allows
you 3 modes for managing status:
* Ansible Operator binary adds generic information about Ansible runs (Default)
* Both Ansible Operator binary and Ansible code are able to update status with generic and customized information
* Solely maintained from within Ansible code

This allows you as the developer to take full control over what you would like
to be populated in the Custom Resource's status field without needing Golang
expertise and just the simplicity of Ansible.

## Ansible Operator's default status management
Ansible Operator by default provides useful information about the Ansible run
by updating the CR's status subresource with the output of the run. This output
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
be maintained solely from within Ansible using the `k8s_status` module and will
not publish the output of the runs to the CR status.

# Example Operator Managing Status
Now that you have a brief overview of managing status with the Ansible
Operator, learn how to [create your own][status_example_blog] Ansible Operator
that provides custom status updates.

[k8s_status_module]:https://github.com/fabianvf/ansible-k8s-status-module
[status_example_blog]:./status-example.md
