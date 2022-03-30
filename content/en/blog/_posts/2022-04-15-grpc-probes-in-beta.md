---
layout: blog
title: "Try the new gRPC probes in 1.24: now in beta."
date: 2022-04-15
slug: grpc-probes-now-in-beta
---

With Kubernetes 1.24 the gRPC probes functionality entered beta and is available by default.
Now you can configure startup, liveness, and readiness probes for your gRPC app
without exposing http endpoint or using an extra executable. All built-in to the Kubernetes.

## Some history

Kubernetes allows to control the container lifecycle from inside the app
by checking http response, running executable, or simply connecting to a tcp port.

Those checks are often enough for the most of apps. For gRPC apps, it is easy
to repurpose the `exec` probe to use it for gRPC health checking. This solution
was described in the blog post "[Health checking gRPC servers on Kubernetes](https://kubernetes.io/blog/2018/10/01/health-checking-grpc-servers-on-kubernetes/)"
and used for a long time. Commonly used tool to enable this was created on [Aug 21, 2018](https://github.com/grpc-ecosystem/grpc-health-probe/commit/2df4478982e95c9a57d5fe3f555667f4365c025d) with
the first release at [Sep 19, 2018](https://github.com/grpc-ecosystem/grpc-health-probe/releases/tag/v0.1.0-alpha.1).

This approach for gRPC apps health checking is very popular. There are [3,626 Dockerfiles](https://github.com/search?l=Dockerfile&q=grpc_health_probe&type=code)
with the `grpc_health_probe` and [6,621 yaml](https://github.com/search?l=YAML&q=grpc_health_probe&type=Code) files that are discovered with the
basic search on GitHub (at the moment of writing). This is good indication of the tool popularity
and the need to support this natively.

With the Kubernetes 1.23, native support for gRPC health probing was introduced as alpha feature.
The feature is not at beta and enabled by default.

## Using the feature

The gRPC health checking is [easy to use](/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe).
The natively supported health probe has many benefits over the workaround involving `grpc_health_probe` executable.

With the native gRPC support you don't need to download and carry `10MB` of an additional executable with your image.
Exec probes are generally slower than a gRPC call as they require instantiating a new process to run an executable.
It also makes the checks less sensible for edge cases when the pod is running at maximum resources and has troubles
instantiating new processes.

There are a few limitations though. Since configuring a client certificate for probes is hard,
services that require client authentication are not supported. The built-in probes are also
not checking the server certificates and ignore related problems.

Built-in checks also cannot be configured to ignore certain types of errors
(`grpc_health_probe` returns different exit codes for different errors),
and cannot be "chained" to run the health check on multiple services in a single probe.

But all these limitations are quite standard for gRPC and there are easy workarounds
for those.

## Try it for yourself

### Getting cluster

You can try this feature today. To try the feature you can spin up the kubernetes cluster
yourself with the `GRPCContainerProbe` feature gate enabled.

Alternatively, at the moment of writing, you can spin up the cluster on GKE
using the following command (note, version is `1.23` and `enable-kubernetes-alpha` are specified):

``` bsh
$ gcloud container clusters create test-grpc \
    --enable-kubernetes-alpha \
    --no-enable-autorepair \
    --no-enable-autoupgrade \
    --release-channel=rapid \
    --cluster-version=1.23
```

You will also need to configure `kubectl` to access the cluster:

``` bsh
$ gcloud container clusters get-credentials test-grpc
```

### Trying the feature out

Lets create the pod to test how gRPC probes work. The `agnhost` image has
a useful [grpc-health-checking](https://github.com/kubernetes/kubernetes/blob/b2c5bd2a278288b5ef19e25bf7413ecb872577a4/test/images/agnhost/README.md#grpc-health-checking) module
that exposes two ports - one is serving health checking service,
another - http port to react on commands `make-serving` and `make-not-serving`.

Here is an example pod definition. It starts the `grpc-health-checking` module,
exposes ports `5000` and `8080`, and configures gRPC readiness probe:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-grpc
spec:
  containers:
  - name: agnhost
    image: k8s.gcr.io/e2e-test-images/agnhost:2.35
    command: ["/agnhost", "grpc-health-checking"]
    ports:
    - containerPort: 5000
    - containerPort: 8080
    readinessProbe:
      grpc:
        port: 5000
```

If the file called `test.yaml`, you can create the pod and check it's status.
The pod will be in ready state as indicated by the snippet of the output.

``` bsh
$ kubectl apply -f test.yaml
$ kubectl describe test-grpc

...
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
...
```

Now let's change the health checking endpoint status to NOT_SERVING.
In order to call the http port of the Pod, let's create a port forward:

``` bsh
$ kubectl port-forward test-grpc 8080:8080
```

You can `curl` to call the command...

``` bsh
$ curl http://localhost:8080/make-not-serving
```

... and in a few seconds the port status will switch to not ready.

``` bsh
$ kubectl describe test-grpc

Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True

...

  Warning  Unhealthy  2s (x6 over 42s)  kubelet            Readiness probe failed: service unhealthy (responded with "NOT_SERVING")
```

Once it is switched back, in ~1s the port will get back to ready status:

``` bsh
$ curl http://localhost:8080/make-serving
$ kubectl describe test-grpc

...

Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
```

Using gRPC health probing on Kubernetes is easy and straightforward. Follow the [documentation to learn more](/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe)
and provide feedback before the feature will be promoted to GA.

# Summary

Kubernetes is a popular workload orchestration platform and we add features based on feedback and demand.
Features like gRPC probes support is a minor improvement that will make life of many app developers
easier and apps more resilient. Try it today and give feedback, before the feature went into GA.
