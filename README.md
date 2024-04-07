# SRE: Practicing with Prometheus alerts

## Alerts implemented

1. Node Down Alert Group
- This alert should be triggered when a Kubernetes node is unreachable for more than 5 minutes.
- Alert Name: InstanceDown
- Expression: up{job="kubernetes-nodes"} == 0
- Trigger Duration: 2 minutes
- Labels:
  - severity: page
- Annotations:
  - host: "{{ $labels.kubernetes_io_hostname }}"
  - summary: "Instance down"
  - description: "Node {{ $labels.kubernetes_io_hostname }} has been down for more than 5 minutes."

2. Low Memory Alert
- This alert should be triggered when a Kubernetes node's available memory falls below 85%.
- Alert Name: LowMemory
- Expression: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 85
- Trigger Duration: 2 minutes
- Labels:
  - severity: warning
- Annotations:
  - host: "{{ $labels.kubernetes_node }}"
  - summary: "{{ $labels.kubernetes_node }} Host is low on memory. Only {{ $value }}% left"
  - description: "{{ $labels.kubernetes_node }} node is low on memory. Only {{ $value }}% left"

3. Kube Persistent Volume Errors Alert
This alert should be triggered if any persistent volume has a status of "Failed" or "Pending".
- Alert Name: KubePersistentVolumeErrors
- Expression: kube_persistentvolume_status_phase{job="kubernetes-service-endpoints",phase=~"Failed|Pending"} > 0
- Trigger Duration: 2 minutes
- Labels:
  - severity: critical
- Annotations:
  - description: The persistent volume {{ $labels.persistentvolume }} has status {{ $labels.phase }}.
  - summary: PersistentVolume is having issues with provisioning.

4. Kube Pod Crash Looping Alert
- This alert should be triggered if any Kubernetes pod is restarting more than once every 5 minutes.
- Alert Name: KubePodCrashLooping
- Expression: rate(kube_pod_container_status_restarts_total{job="kubernetes-service-endpoints",namespace=~".*"}[5m]) * 𝟼𝟶 * 𝟻 > 𝟶
- Trigger Duration: 2 minutes
- Labels:
 - severity: warning
- Annotations:
    - description: Pod {{ $labels.namespace }}/{{ $labels.pod }} ({{ $labels.container }}) is restarting {{ printf "%.2f" $value }} times / 5 minutes.
    - summary: Pod is crash looping.

5. Kube Pod Not Ready Alert
- This alert should be triggered if any Kubernetes pod remains in a non-ready state for longer than 2 minutes.
- Alert Name: KubePodNotReady
- Expression: sum by(namespace, pod) (max by(namespace, pod) (kube_pod_status_phase{job="kubernetes-service-endpoints",namespace=~".*",phase=~"Pending|Unknown"}) * on(namespace, pod)    group_left(owner_kind) topk by(namespace, pod) (1, max by(namespace, pod, owner_kind) (kube_pod_owner{owner_kind!="Job"}))) > 0
- Trigger Duration: 2 minutes
- Labels:
  - severity: warning
- Annotations:
  - description: Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready state for longer than 5 minutes.
  - summary: Pod has been in a non-ready state for more than 2 minutes."

## How To

1. Create a kubernetes cluster
```sh
minikube start
```
2. Create a namespace
```
kubectl create namespace sre
```
3. Add prometheus helm repository
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
4. Add grafana helm repository
```sh
helm repo add grafana https://grafana.github.io/helm-charts
```
5. Update helm local cache
```sh
helm repo update
```
6. Install prometheus
```sh
helm install prometheus prometheus-community/prometheus -f prometheus.yml --namespace sre
```
7. Install grafana
```sh
helm install grafana grafana/grafana --set adminPassword="admin" --namespace sre
```
8. Create deployment
```sh
kubectl apply -f deployment.yml -n sre
kubectl apply -f service.yml -n sre
```
9. Run service
```sh
minikube service upcommerce-service -n sre
```
10. Setup port forwarding
  a. Alertmanager
```sh
export POD_NAME=$(kubectl get pods --namespace sre -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace sre port-forward $POD_NAME 9093
```
  b. Prometheus
```sh
export POD_NAME=$(kubectl get pods --namespace sre -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace sre port-forward $POD_NAME 9090
```
  c. Application (replace port with actual value)
```sh
minikube service upcommerce-service -n sre --url

kubectl port-forward service/upcommerce-service -n sre 30768:5000
```



