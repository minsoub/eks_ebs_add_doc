### PV 확인
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Grafana_configuration   master ±✚  kubectl get pv -n systems-dev-ns       
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                    STORAGECLASS   REASON   AGE
pvc-370418a3-1830-4f12-a452-80f1619e58c1   1Gi        RWO            Delete           Bound    systems-dev-ns/config-path-elasticsearch-node-0          ebs-sc                  86d
pvc-4985a7bb-48af-4eac-b6bf-233e203231a0   1Gi        RWO            Delete           Bound    systems-dev-ns/config-path-elasticsearch-node-1          ebs-sc                  86d
pvc-93fe8c30-b9c1-411a-b2a2-1cdd90692e29   1Gi        RWO            Delete           Bound    systems-dev-ns/config-path-elasticsearch-node-2          ebs-sc                  86d
pvc-ba277c5d-aac8-4fbe-be2f-8a142e0f5180   5Gi        RWO            Delete           Bound    systems-dev-ns/persistent-storage-elasticsearch-node-1   ebs-sc                  93d
pvc-bc31ac74-109f-4b1a-aebd-df78f9e3b075   10Gi       RWO            Delete           Bound    systems-dev-ns/grafana-pvc                               grafana-sc              2s
pvc-bde46146-4c46-49f5-8fab-9310c5992a4a   5Gi        RWO            Delete           Bound    systems-dev-ns/persistent-storage-elasticsearch-node-0   ebs-sc                  93d
pvc-ec13136a-9c60-4c8f-a5c2-e1f59a7eaa37   5Gi        RWO            Delete           Bound    systems-dev-ns/persistent-storage-elasticsearch-node-2   ebs-sc                  93d

```
### Ingress 확인
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Grafana_configuration   master ±✚  kubectl get ingress -n systems-dev-ns
NAME                               CLASS    HOSTS   ADDRESS                                                                        PORTS   AGE
cms-app-api-ingress                <none>   *       k8s-systemsd-cmsappap-eca615f605-1526138129.ap-northeast-2.elb.amazonaws.com   80      8d
cpc-app-web-ingress                <none>   *       k8s-systemsd-cpcappwe-c3110031ec-598124796.ap-northeast-2.elb.amazonaws.com    80      180d
external-gateway-api-ingress       <none>   *       k8s-systemsd-external-948a615e29-111252025.ap-northeast-2.elb.amazonaws.com    80      15d
grafana-ingress                    <none>   *       k8s-systemsd-grafanai-08240a1466-1183650272.ap-northeast-2.elb.amazonaws.com   80      10s
kibana-ingress                     <none>   *       k8s-systemsd-kibanain-51900853d4-2114978363.ap-northeast-2.elb.amazonaws.com   80      92d
lrc-app-api-ingress                <none>   *       k8s-systemsd-lrcappap-e6b386e4a5-1556147373.ap-northeast-2.elb.amazonaws.com   80      161d
lrc-app-web-ingress                <none>   *       k8s-systemsd-lrcappwe-3ba5072e02-1786558476.ap-northeast-2.elb.amazonaws.com   80      180d
smart-admin-listener-api-ingress   <none>   *       k8s-systemsd-smartadm-ea3afcf58c-1207627060.ap-northeast-2.elb.amazonaws.com   80      94d
systems-chat-api-ingress           <none>   *       k8s-systemsd-systemsc-383ed2dc5d-1210336174.ap-northeast-2.elb.amazonaws.com   80      149d
systems-gateway-api-ingress        <none>   *       k8s-systemsd-systemsg-ebcd4f0604-1552855188.ap-northeast-2.elb.amazonaws.com   80      182d
systems-mng-web-ingress            <none>   *       k8s-systemsd-systemsm-5b1ad0cbb7-1915523199.ap-northeast-2.elb.amazonaws.com   80      181d

```
### Grafana 접속
http://k8s-systemsd-grafanai-08240a1466-1183650272.ap-northeast-2.elb.amazonaws.com
ID : admin
Pass : admin => Wjdalstjq1!