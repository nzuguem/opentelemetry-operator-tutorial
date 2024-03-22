# [OpenTelemetry Operator][otel-operator-gh]

It is an operator in the Kubernetes sense, facilitating the adoption and implementation of OpenTelemetry in an application cluster.

It allows you to do 2 great things :
- [OpenTelemetry Collector][otel-collector-gh]
- Auto-instrumentation of the workloads using OpenTelemetry instrumentation libraries

## Install Operator on Minikube

### Helm

```bash
helm repo add otel https://open-telemetry.github.io/opentelemetry-helm-charts

helm repo update

helm install --create-namespace --namespace otel-system otel otel/opentelemetry-operator --set admissionWebhooks.certManager.enabled=false --set admissionWebhooks.certManager.autoGenerateCert=true

# Check the correct installation of the chart
helm status -n otel-system otel

#NAME: otel
#LAST DEPLOYED: Wed Feb 28 06:52:42 2024
#NAMESPACE: otel-system
#STATUS: deployed
#REVISION: 1
#NOTES:
#opentelemetry-operator has been installed. Check its status #by running:
#  kubectl --namespace otel-system get pods -l "release=otel"
```

## Usage

Operator extends Kubernetes API with new [CRDs][otel-operator-api-doc-gh]

### Create OpenTelemetry Collector

> ⚠️ The OpenTelemetry operator does not ***semantically*** validate the contents of the configuration in the manifest: if the configuration is ***semantically*** invalid, the instance will still be created, but the underlying OpenTelemetry collector may crash.

The OpenTelemetry operator analyzes the **recievers** and their ports in the configuration and creates corresponding services (standard and headless).

Example :
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: collector
spec:
  # Automatic resource update in the event of an operator update
  # Be careful with the "none" value, as there is a risk of version dephasing between OTel Operator and Collector
  upgradeStrategy: automatic
  mode: <deployment | statefulset | daemonset | sidecar>
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        send_batch_size: 10000
        timeout: 10s

    exporters:
      debug:

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [debug]
```

- Mode ***Deployment*** : OTelcol collects telemetry data for the cluster as a whole

```bash
kubectl create -f manifests/collectors/collector.yml
```

- Mode ***StatefulSet***

```bash
kubectl create -f manifests/collectors/collector-sts.yml
```

- Mode ***DaemonSet*** : OTelcol collects telemetry data about a node and the workloads hosted on it.

```bash
kubectl create -f manifests/collectors/collector-ds.yml
```

- Mode ***Sidecar***

```bash
kubectl create -f manifests/collectors/collector-sidecar.yml
```

### Focus on Sidecar injection
After defining the `OpenTelemetryCollector` resource in sidecar mode, simply inject it into the Pods (**NOT Deployment**) using the annotation :
`sidecar.opentelemetry.io/inject: "true"`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    sidecar.opentelemetry.io/inject: "true"
spec:
  containers:
  - name: myapp
    image: jaegertracing/vertx-create-span:operator-e2e-tests
    ports:
      - containerPort: 8080
```
```bash
kubectl create -f manifests/workloads/deploy-java.yml
```

This annotation (`sidecar.opentelemetry.io/inject`) can be valued in several ways : 

- `true` : inject the only `OpenTelemetryCollector` resource in the namespace
- `OpenTelemetryCollector Sidecar Name` : name of `OpenTelemetryCollector` CR instance in the current namespace
- `namespace/OpenTelemetryCollector Sidecar Name` : name and namespace of OpenTelemetryCollector CR instance in another namespace
- `false` : do not inject

> 📌 The sidecar injection annotation can be defined at the namespace level

When using sidecar mode the OpenTelemetry collector container will have the environment variable `OTEL_RESOURCE_ATTRIBUTES` set with Kubernetes resource attributes, ready to be consumed by the [resourcedetection][otelcol-processor-resourcedetection-gh] processor

### Using `imagePullSecrets`
> `imagePullSecrets` is a ServiceAccount and Pod property that references Docker Registry Secrets. This property can only be accessed by the kubelet to pull images from the Container Private Registry

In the case of OTelcol, the image can be stored in a Container Private Registry. The image can also be a custom distribution built using the [OpenTelemetry Collector Builder (OCB)][ocb-doc]

```bash
kubectl create secret docker-registry nzuguem-cr-secret --docker-server=https://index.docker.io/v1/ \
        --docker-username=nzuguem --docker-password=DUMMY_DOCKER_PASSWORD \
        --docker-email=nzuguemkevine@gmail.com
```
```bash
kubectl create -f manifests/collectors/collector-sidecar-ocb-sa.yml
```
```bash
kubectl create -f manifests/collectors/collector-sidecar-ocb.yml
```

### OpenTelemetry auto-instrumentation injection
Installing OTelcol in your cluster is not enough to get telemetry data. Applications or subjects must send or expose signals to OTelcol.

To this end, applications or subjects must be instrumented: the OpenTelemetry operator can be used to.

This Instrumentation resource lets you configure the OpenTelemetry SDK by language. SDK, which will then be automatically injected into applications or subjects.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: instrumentation
spec:
  exporter:
    endpoint: http://localhost:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
```

To create an instrumentation resource : 
```bash
kubectl create -f manifests/instrumentations/instrumentation.yml
```
After resource instrumentation, the workloads (Pod, PodSpec) to be instrumented must be annotated: `instrumentation.opentelemetry.io/inject-java: "true"`
Like the `sidecar.opentelemetry.io/inject` annotation, this one can be valued in several ways :
- `true` : inject and Instrumentation resource from the namespace
- `Instrumentation Name` - name of Instrumentation CR instance in the current namespace
- `namespace/Instrumentation Name` : name and namespace of Instrumentation CR instance in another namespace
- `false` : do not inject

To inject only OTel SDK environment variables at workload level (without instrumentation itself): `instrumentation.opentelemetry.io/inject-sdk="true"`

> 📌 This annotation can be defined at the namespace level

#### Multi-container pods with single instrumentation
By default, instrumentation is performed on the first Pod container. This can cause problems with certain k8s operators (Istio, for example).

It is therefore necessary to designate (*pod.spec.containers.name*) the containers to be instrumented: `instrumentation.opentelemetry.io/container-names: "myapp,myapp2"`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-multiple-containers
spec:
  selector:
    matchLabels:
      app: my-pod-with-multiple-containers
  replicas: 1
  template:
    metadata:
      labels:
        app: my-pod-with-multiple-containers
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
        instrumentation.opentelemetry.io/container-names: "myapp,myapp2"
    spec:
      containers:
      - name: myapp
        image: myImage1
      - name: myapp2
        image: myImage2
      - name: myapp3 # This container will not be instrumented
        image: myImage3
```

When there are several instrumentations to be made on the Pod, we can be more precise about the containers to be instrumented :
```
instrumentation.opentelemetry.io/inject-java: "true"
instrumentation.opentelemetry.io/java-container-names: "myapp,myapp2"
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-multi-containers-multi-instrumentations
spec:
  selector:
    matchLabels:
      app: my-pod-with-multi-containers-multi-instrumentations
  replicas: 1
  template:
    metadata:
      labels:
        app: my-pod-with-multi-containers-multi-instrumentations
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
        instrumentation.opentelemetry.io/java-container-names: "myapp,myapp2"
        instrumentation.opentelemetry.io/inject-python: "true"
        instrumentation.opentelemetry.io/python-container-names: "myapp3"
    spec:
      containers:
      - name: myapp # Java Instrumentation
        image: myImage1
      - name: myapp2 # Java Instrumentation
        image: myImage2
      - name: myapp3 # Python Instrumentation
        image: myImage3
```
> 🚨 Go auto-instrumentation does not support multicontainer pods. When injecting Go auto-instrumentation the first pod should be the only pod you want instrumented

#### Use customized or vendor instrumentation
To instrument a workload, the OpenTelemetry operator injects an InitContainer (specifications can be found [here][auto-instrumentation-spec-gh]) containing the agent or component to be injected. This InitContainer will copy the component to a volume shared with the workload.

We can create our own container images for instrumentation :
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  java:
    image: your-customized-auto-instrumentation-image:java
  nodejs:
    image: your-customized-auto-instrumentation-image:nodejs
  python:
    image: your-customized-auto-instrumentation-image:python
  dotnet:
    image: your-customized-auto-instrumentation-image:dotnet
  go:
    image: your-customized-auto-instrumentation-image:go
  apacheHttpd:
    image: your-customized-auto-instrumentation-image:apache-httpd
  nginx:
    image: your-customized-auto-instrumentation-image:nginx
```

> f your custom images are in your container registries, then the workload to be instrumented needs to know the identification data (ImagePullSecrets, ServiceAccount) in the container Registry.
#### Controlling Instrumentation Capabilities
We can configure the instrumentation capability of our workloads (e.g. prohibit instrumentation of DotNet and activated instrumentation of Go workloads).

This is done via operator configuration (Chart Helm): ***feature gates***
 ```yaml
# values.yml
manager:
  # Feature Gates are a a comma-delimited list of feature gate identifiers.
  # Prefix a gate with '-' to disable support.
  # Prefixing a gate with '+' or no prefix will enable support.
  # A full list of valud identifiers can be found here: https://github.com/open-telemetry/opentelemetry-operator/blob/main/pkg/featuregate/featuregate.go
  featureGates: "-operator.autoinstrumentation.dotnet,+operator.autoinstrumentation.go"
```

## Compatibility matrix
### OTel Collector vs OTel Operator

The OpenTelemetry Operator follows the same versioning as the operand (OpenTelemetry Collector) up to the minor part of the version. For example, the OpenTelemetry Operator v0.18.1 tracks OpenTelemetry Collector 0.18.0. The patch part of the version indicates the patch level of the operator itself, not that of OpenTelemetry Collector.

When a custom `otelcol.spec.image` is used with an `OpenTelemetryCollector` resource, the **OpenTelemetry Operator will not manage this versioning and upgrading.**
In this scenario, it is best practice that the OpenTelemetry Operator version should match the underlying core version. 
Given a OpenTelemetryCollector resource with a `otelcol.spec.image` configured to a custom image based on underlying OpenTelemetry Collector at version 0.40.0, it is recommended that the OpenTelemetry Operator is kept at version 0.40.0.
<!-- Links -->
[otel-collector-gh]:https://github.com/open-telemetry/opentelemetry-collector
[otel-operator-gh]: https://github.com/open-telemetry/opentelemetry-operator
[otelcol-processor-resourcedetection-gh]:https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourcedetectionprocessor
[ocb-doc]: https://opentelemetry.io/docs/collector/custom-collector/
[auto-instrumentation-spec-gh]:https://github.com/open-telemetry/opentelemetry-operator/tree/main/autoinstrumentation
[otel-operator-api-doc-gh]: https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api.md