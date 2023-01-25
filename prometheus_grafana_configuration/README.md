## helm 으로 prometheus & grafana를 설치
https://github.com/prometheus-community/helm-charts

### helm repo add
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/giltab-server-install-doc/cpc-app-web   release/release-20221028  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
```

### helm repo update
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/giltab-server-install-doc/cpc-app-web   release/release-20221028  helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "eks" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```
### values.yaml
https://github.com/grafana/helm-charts/blob/main/charts/grafana/values.yaml
을 다운로드 받는다.
```shell
 ✘ jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/prometheus_grafana_configuration   master ±✚  kubectl create ns monitoring
namespace/monitoring created

 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/prometheus_grafana_configuration   master ±✚  helm install prometheus prometheus-community/kube-prometheus-stack -f "values.yaml" --namespace monitoring
NAME: prometheus
LAST DEPLOYED: Fri Jan 20 10:53:37 2023
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=prometheus"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

### 정상 확인
```shell
 ✘ jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/prometheus_grafana_configuration   master ±✚  kubectl --namespace  monitoring get pods -l "release=prometheus"
NAME                                                   READY   STATUS    RESTARTS   AGE
prometheus-kube-prometheus-operator-799f78fdf5-4zvqr   1/1     Running   0          77s
prometheus-kube-state-metrics-564f699ff9-ss4kh         1/1     Running   0          77s
prometheus-prometheus-node-exporter-8sbn7              1/1     Running   0          77s
prometheus-prometheus-node-exporter-c6qhr              1/1     Running   0          77s
prometheus-prometheus-node-exporter-d7lbb              1/1     Running   0          77s
prometheus-prometheus-node-exporter-pnx8j              1/1     Running   0          77s
```

### Grafana 실행 확인
port를 80에서 local 컴퓨터 3000 프트로 변경
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/prometheus_grafana_configuration   master ±✚   kubectl port-forward service/prometheus-grafana 3000:80 --namespace monitoring

```
id는 admin

pw는 prom-operator