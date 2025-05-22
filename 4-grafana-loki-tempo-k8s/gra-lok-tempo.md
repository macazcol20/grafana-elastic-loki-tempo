https://github.com/npuichigo/istio-stack

https://medium.com/otomi-platform/debugging-microservices-on-k8s-with-istio-opentelemetry-and-tempo-4c36c97d6099
https://medium.com/otomi-platform/debugging-microservices-on-kubernetes-with-istio-opentelemetry-and-tempo-part-2-e10b951029a0


# Istio stack
<img src="stack.png" alt="istio stack" width="400"/>

Equip k8s cluster with the following components:
* Istio - Service Mesh --->  I already have
* OpenTelemetry - Monitoring
* Prometheus - Metrics
* Tempo - Tracing
* Loki - Logging
* Grafana - Visualization

## Getting started
### Istio
Install Istio with custom IstioOperator to use OpenTelemetry provider:

NOTE: Already installed

To enable the extensionProvider, we need to create a Telemetry resource:
```bash
kubectl apply -f istio/telemetry.yaml
```

### OpenTelemetry
Install cert-manager following https://cert-manager.io/docs/installation/helm/

Install `opentelemetry-operator` and related CRDs:
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

kubectl create ns otel
helm install opentelemetry-operator open-telemetry/opentelemetry-operator -n otel
```

Configure the OpenTelemetry collector to receive, process and export the collected telemetry to the desired backends (which will be deployed soon) with a `OpenTelemetryCollector` resource:
```bash
kubectl apply -f opentelemetry/otel-collector.yaml -n otel
k -n otel get po
```

### Prometheus
Install Prometheus with `kube-prometheus-stack` Helm chart:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create ns prometheus
helm install prom prometheus-community/kube-prometheus-stack -n prometheus -f prometheus/values.yaml
k -n prometheus get po
```

Configure the Prometheus Operator to scrape the Istio control plane and workloads
```bash
kubectl apply -f prometheus/service-monitor.yaml
kubectl apply -f prometheus/pod-monitor.yaml
```

### Tempo
Tempo supports S3, GCS, Azure or filesystem as storage backends. In this example, we will use Azure as the storage backend.
For this turorial. I wil use Minio:

```sh
### get the secrets to login
kubectl get secret minio-creds-secret -n minio -o jsonpath='{.data.accesskey}' | base64 --decode && echo
kubectl get secret minio-creds-secret -n minio -o jsonpath='{.data.secretkey}' | base64 --decode && echo


## if no access, deploy a minio pod for cli
## Deploy a Permanent Pod (Reusable)

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: minio-client
  namespace: minio
spec:
  containers:
  - name: minio-client
    image: minio/mc
    command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
EOF

## exec into the pod
kubectl exec -it minio-client -n minio -- /bin/sh

## signin and make a bucket
mc alias set minio http://minio-minio-instance-hl.minio.svc.cluster.local:9000 minio minio123

mc mb minio/tempo-traces
mc mb minio/loki
mc mb minio/loki-ruler
mc mb minio/loki-admin
```


Create a `Secret` with the Azure Storage Account credentials before installing Tempo:
```bash
kubectl create ns tempo
kubectl create secret generic tempo-minio-creds -n tempo \
  --from-literal=access_key_id=minio \
  --from-literal=secret_access_key=minio123

```

Install Tempo with `tempo-distributed` Helm chart:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install -f tempo/values.yaml tempo grafana/tempo-distributed -n tempo
```

You need to check if `OpenTelemetryCollector` is correctly sending traces to Tempo.

### Loki
Same as Tempo, Loki needs a storage backend too. In this example, we will use Azure as the storage backend.

Create a `Secret` with the Azure Storage Account credentials before installing Loki:
```bash
kubectl create ns loki

kubectl create secret generic loki-minio-creds -n loki \
  --from-literal=access_key_id=minio \
  --from-literal=secret_access_key=minio123

```

Install Loki with `loki` Helm chart:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install -f loki/values.yaml -n loki loki grafana/loki
```

<!-- Export log from `OpenTelemetryCollector` to Loki is still experimental. Here we configure the `log` components installed on each node to send logs to Loki with a `PodLogs` resource:
```bash
kubectl apply -f loki/podlogs.yaml
``` -->

### setup fluentbit 
What This DaemonSet Will Do:
Collect logs from container files in /var/log/containers/*.log
Enrich with Kubernetes metadata (e.g. namespace, pod, container name)
Forward logs to Loki, which stores and indexes them: http://loki-gateway.loki:3100/loki/api/v1/push

```sh
## Create Namespace (if not already)
kubectl create namespace observability
k -n observability apply -f fluentbit/cm.yaml
k -n observability apply -f fluentbit/ds.yaml

kubectl get pods -n observability -l app=fluent-bit
kubectl logs -n observability -l app=fluent-bit
```
### Grafana
Grafana may have been installed with `kube-prometheus-stack` Helm chart. If not, install it with helm chart:
Yes in my  case. I have grafana already:
- get the credentials:
Username: admin
Password: prom-operator

```sh
kubectl port-forward -n prometheus svc/prom-grafana 3000:80
## you can get secret
kubectl get secret -n prometheus prom-grafana -o jsonpath="{.data.admin-password}" | base64 -d && echo

```

#### Data source
So in order to add the data source, we need to make this update under the grafana section, in the already installed prometheus/values.yaml
search grafa in the helm chart and update this section    additionalDataSources: []

## previously  ####################
additionalDataSources: []
##### #### ############# ##########


## Update to   ####################
```sh
additionalDataSources:
- name: Tempo
type: tempo
uid: tempo
access: proxy
url: http://tempo-query-frontend.tempo:3100
jsonData:
nodeGraph:
    enabled: true

- name: Loki
editable: false
type: loki
uid: loki
access: proxy
url: http://loki-gateway.loki
jsonData:
    derivedFields:
    - datasourceName: Tempo
    matcherRegex: "traceID=00-([^\\-]+)-"
    name: traceID
    url: "$${__value.raw}"
    datasourceUid: tempo
##### #### ############# ##########

## Then upgrade the helm deployment

helm upgrade prom prometheus-community/kube-prometheus-stack \
  -n prometheus \
  -f prometheus/values.yaml



```


Tempo:
```yaml
grafana:
  additionalDataSources:
  - name: Tempo
    type: tempo
    uid: tempo
    access: proxy
    url: http://tempo-query-frontend.tempo:3100
    jsonData:
      nodeGraph:
        enabled: true
```

Loki:
```yaml
grafana:
  additionalDataSources:
  - name: Loki
    editable: false
    type: loki
    uid: loki
    access: proxy
    url: http://loki-gateway.loki
    jsonData:
      derivedFields:
      - datasourceName: Tempo
        matcherRegex: "traceID=00-([^\\-]+)-"
        name: traceID
        url: "$${__value.raw}"
        datasourceUid: tempo
```


## portforward grafana
kubectl port-forward -n prometheus svc/prom-grafana 3000:80

#### Dashboards
Istio dashboards can be found [here](https://grafana.com/orgs/istio/dashboards) and installed as `ConfigMap` resources.


###### ########### ###################### ############################################################################################


## Deleting everything
```sh
# Delete Loki
helm uninstall loki -n loki

# Delete Tempo
helm uninstall tempo -n tempo

# Delete Prometheus stack (includes Grafana)
helm uninstall prom -n prometheus

# Delete OpenTelemetry Operator
helm uninstall opentelemetry-operator -n otel

# If you used cert-manager
helm uninstall cert-manager -n cert-manager

kubectl delete ns loki tempo prometheus otel observability 

# Delete OpenTelemetry CRDs
kubectl get crds | grep opentelemetry | awk '{print $1}' | xargs kubectl delete crd

# Delete Prometheus CRDs
kubectl get crds | grep monitoring.coreos.com | awk '{print $1}' | xargs kubectl delete crd

# Delete Loki/Tempo CRDs (if present)
kubectl get crds | grep loki | awk '{print $1}' | xargs kubectl delete crd || true
kubectl get crds | grep tempo | awk '{print $1}' | xargs kubectl delete crd || true

kubectl get cm -n prometheus | grep grafana-dashboard | awk '{print $1}' | xargs -I{} kubectl delete cm {} -n prometheus


## References
[Debugging microservices on Kubernetes with Istio, OpenTelemetry and Tempo — Part 1](https://medium.com/otomi-platform/debugging-microservices-on-k8s-with-istio-opentelemetry-and-tempo-4c36c97d6099)

[Debugging microservices on Kubernetes with Istio, OpenTelemetry and Tempo — Part 2](https://medium.com/otomi-platform/debugging-microservices-on-kubernetes-with-istio-opentelemetry-and-tempo-part-2-e10b951029a0)