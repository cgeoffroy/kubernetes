<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->

<!-- BEGIN STRIP_FOR_RELEASE -->

![WARNING](http://kubernetes.io/img/warning.png)
![WARNING](http://kubernetes.io/img/warning.png)
![WARNING](http://kubernetes.io/img/warning.png)

<h1>PLEASE NOTE: This document applies to the HEAD of the source
tree only. If you are using a released version of Kubernetes, you almost
certainly want the docs that go with that version.</h1>

<strong>Documentation for specific releases can be found at
[releases.k8s.io](http://releases.k8s.io).</strong>

![WARNING](http://kubernetes.io/img/warning.png)
![WARNING](http://kubernetes.io/img/warning.png)
![WARNING](http://kubernetes.io/img/warning.png)

<!-- END STRIP_FOR_RELEASE -->

<!-- END MUNGE: UNVERSIONED_WARNING -->
# Kubernetes Proposal - Admission Control

**Related PR:**

| Topic | Link |
| ----- | ---- |
| Separate validation from RESTStorage | https://github.com/GoogleCloudPlatform/kubernetes/issues/2977 |

## Background

High level goals:

* Enable an easy-to-use mechanism to provide admission control to cluster
* Enable a provider to support multiple admission control strategies or author their own
* Ensure any rejected request can propagate errors back to the caller with why the request failed

Authorization via policy is focused on answering if a user is authorized to perform an action.

Admission Control is focused on if the system will accept an authorized action.

Kubernetes may choose to dismiss an authorized action based on any number of admission control strategies.

This proposal documents the basic design, and describes how any number of admission control plug-ins could be injected.

Implementation of specific admission control strategies are handled in separate documents.

## kube-apiserver

The kube-apiserver takes the following OPTIONAL arguments to enable admission control

| Option | Behavior |
| ------ | -------- |
| admission_control | Comma-delimited, ordered list of admission control choices to invoke prior to modifying or deleting an object. |
| admission_control_config_file | File with admission control configuration parameters to boot-strap plug-in. |

An **AdmissionControl** plug-in is an implementation of the following interface:

```go
package admission

// Attributes is an interface used by a plug-in to make an admission decision on a individual request.
type Attributes interface {
  GetNamespace() string
  GetKind() string
  GetOperation() string
  GetObject() runtime.Object
}

// Interface is an abstract, pluggable interface for Admission Control decisions.
type Interface interface {
  // Admit makes an admission decision based on the request attributes
  // An error is returned if it denies the request.
  Admit(a Attributes) (err error)
}
```

A **plug-in** must be compiled with the binary, and is registered as an available option by providing a name, and implementation
of admission.Interface.

```go
func init() {
  admission.RegisterPlugin("AlwaysDeny", func(client client.Interface, config io.Reader) (admission.Interface, error) { return NewAlwaysDeny(), nil })
}
```

Invocation of admission control is handled by the **APIServer** and not individual **RESTStorage** implementations.

This design assumes that **Issue 297** is adopted, and as a consequence, the general framework of the APIServer request/response flow
will ensure the following:

1. Incoming request
2. Authenticate user
3. Authorize user
4. If operation=create|update, then validate(object)
5. If operation=create|update|delete, then admission.Admit(requestAttributes)
  a. invoke each admission.Interface object in sequence
6. Object is persisted

If at any step, there is an error, the request is canceled.


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/design/admission_control.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
