+++
date = "2018-09-10T12:00:00-07:00"
title = "Experimental: Automatic Proxy Injection"
[menu.l5d2docs]
  name = "Experimental: Automatic Proxy Injection"
  weight = 13
+++

Linkerd can be configured to automatically inject the data plane proxy into your
service. This is an alternative to needing to run the `linkerd inject` command.

This feature is **experimental**, since it only supports Deployment-type
workloads. It will be removed from experimental once all workloads are supported
([tracked here](https://github.com/linkerd/linkerd2/issues/1751)).

FIXME Feature highlights:

* Implemented via a proxy injector webhook.
* All required permissions are defined in the new `linkerd-linkerd-proxy-injector` cluster role, separated from the cluster role of the control plane.
* When a new deployment is created, the Linkerd proxy will be automatically injected if:
  * the namespace has the `linkerd.io/auto-inject: enabled` label or
  * the namespace isn't labeled with the `linkerd.io/auto-inject` label.
* Namespaces that are labeled with the `linkerd.io/auto-inject: disabled` are ignored.
* Within "enabled" namespaces, deployments with pods that are labeled with `linkerd.io/auto-inject: disabled` or `linkerd.io/auto-inject: completed` are ignored.

## Installation

Automatic proxy injection is implemented with an
[admission webhook)(https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks),
which requires your cluster to have the `admissionregistration.k8s.io/v1beta1`
API enabled. To verify, run:

```bash
kubectl api-versions | grep admissionregistration
admissionregistration.k8s.io/v1beta1
```

Automatic proxy injection is disabled by default when installing the Linkerd
control plane. To enable it, set the `--proxy-auto-inject` flag, as follows:

```bash
$ linkerd install --proxy-auto-inject | kubectl apply -f -
```

This will add a new `linkerd-proxy-injector` service and deployment to the
control plane, to serve the admission webhook.

```bash
$ kubectl -n linkerd get deploy/linkerd-proxy-injector
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
linkerd-proxy-injector   1         1         1            1           1m
```

## Configuration

By default, automatic proxy injection is "opt-in", meaning that injection will
only be performed on pods with the `linkerd.io/inject: enabled` annotation, or
on pods in namespaces with the `linkerd.io/inject: enabled` annotation. See
the [Opt-In](#opt-in) section below for more information about opting in.

It's also possible to configure automatic proxy inject to be "opt-out", meaning
that injection will be performed on all pods in all namespaces, unless the pod
or namespace have the `linkerd.io/inject: disabled` annotation set. To install
in opt-out mode, set the `--proxy-auto-inject-mode` flag, as follows:

```bash
$ linkerd install --proxy-auto-inject --proxy-auto-inject-mode=opt-out | kubectl apply -f -
```

See the [Opt-Out](#opt-out) section below for more information about opting out.

### Opt-In

In opt-in mode, either the pod or namespace must include the
`linkerd.io/inject: enabled` annotation in order for it to be injected.

For example, to opt-in all pods in a namespace, modify the namespace to include
the correct annotation, as follows:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sample-inject-enabled-namespace
  annotations:
    linkerd.io/inject: enabled
```

Whereas to opt-in all pods in a deployment, modify the deployment's pod spec to
include the correct annotation, as follows:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sample-inject-enabled-deployment
  labels:
    app: sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
      labels:
        app: sample
    spec:
      containers:
      - name: sample
        image: buoyantio/helloworld
```

It's also possible to opt-out all pods in a deployment in a namespace that has
been opted-in via the `linkerd.io/inject: enabled` annotation, by setting the
`linkerd.io/inject: disabled` annotation on the deployment's pod spec, as
follows:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sample-inject-disabled-deployment
  namespace: sample-inject-enabled-namespace
  labels:
    app: sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      annotations:
        linkerd.io/inject: disabled
      labels:
        app: sample
    spec:
      containers:
      - name: sample
        image: buoyantio/helloworld:0.1.6
```

### Opt-Out

In opt-out mode, pods will be automatically injected unless the the pod or
namespace include the `linkerd.io/inject: disabled` annotation.

For example, to opt-out all pods in a namespace, modify the namespace to include
the correct annotation, as follows:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sample-inject-disabled-namespace
  annotations:
    linkerd.io/inject: disabled
```

Whereas to opt-out all pods in a deployment, modify the deployment's pod spec to
include the correct annotation, as follows:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sample-inject-enabled-deployment
  labels:
    app: sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      annotations:
        linkerd.io/inject: disabled
      labels:
        app: sample
    spec:
      containers:
      - name: sample
        image: buoyantio/helloworld
```

It's also possible to opt-in all pods in a deployment in a namespace that has
been opted-out via the `linkerd.io/inject: disabled` annotation, by setting the
`linkerd.io/inject: enabled` annotation on the deployment's pod spec, as
follows:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sample-inject-enabled-deployment
  namespace: sample-inject-disabled-namespace
  labels:
    app: sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
      labels:
        app: sample
    spec:
      containers:
      - name: sample
        image: buoyantio/helloworld:0.1.6
```
