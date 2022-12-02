# Amazon EKS에서 영구 스토리지 사용

## Amazon EBS CSI 드라이버 배포 및 테스트

### 작업자 노드가 Amazon EBS 볼륨을 생성 및 수정할 수 있도록 허용하는 권한이 있는 IAM 정책 다운로드

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.9.0/docs/example-iam-policy.json
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  mv example-iam-policy.json ebs-csi-driver-iam-policy.json
```

- 내용을 확인한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications",
        "ec2:DetachVolume",
        "ec2:ModifyVolume"
      ],
      "Resource": "*"
    }
  ]
```

- Amazon_EBS_CSI_Driver라는 IAM 정책을 생성한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://ebs-c
si-driver-iam-policy.json
{
    "Policy": {
        "PolicyName": "AmazonEKS_EBS_CSI_Driver_Policy",
        "PolicyId": "ANPA3X64ZUYGTSF5P4AFZ",
        "Arn": "arn:aws:iam::807380035085:policy/AmazonEKS_EBS_CSI_Driver_Policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-08-30T02:35:45+00:00",
        "UpdateDate": "2022-08-30T02:35:45+00:00"
    }
}
```

- 클러스터의 OIDC 공급자 URL을 확인한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  aws eks describe-cluster --name systems-eks-cluster --query "cluster.identity.oidc.issuer" --outpu
t text
https://oidc.eks.ap-northeast-2.amazonaws.com/id/50BD6A5EF52FA9721B3A780D19AF9C49
```

- IAM 신뢰 정책 파일을 생성한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  vi trust-policy.json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
         "Federated": "arn:aws:iam::807380035085:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/50BD6A5EF52FA9721B3A780D19AF9C49"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
         "StringEquals": {
            "oidc.eks.ap-northeast-2.amazonaws.com/id/50BD6A5EF52FA9721B3A780D19AF9C49:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
         }
      }
    }
  ]
}

```

- 생성된 파일을 가지고 IAM 역할을 생성한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  aws iam create-role --role-name AmazonEKS_EBS_CSI_DriverRole --assume-role-policy-document file://
"trust-policy.json"

{
    "Role": {
        "Path": "/",
        "RoleName": "AmazonEKS_EBS_CSI_DriverRole",
        "RoleId": "AROA3X64ZUYG5SSGV3EG4",
        "Arn": "arn:aws:iam::807380035085:role/AmazonEKS_EBS_CSI_DriverRole",
        "CreateDate": "2022-08-30T02:44:34+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Federated": "arn:aws:iam::807380035085:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/50BD6A5EF52FA9721B3A780D19AF9C49"
                    },
                    "Action": "sts:AssumeRoleWithWebIdentity",
                    "Condition": {
                        "StringEquals": {
                            "oidc.eks.ap-northeast-2.amazonaws.com/id/50BD6A5EF52FA9721B3A780D19AF9C49:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
                        }
                    }
                }
            ]
        }
    }
}
```

- 새 IAM 정책을 역할에 연결한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  aws iam attach-role-policy --policy-arn arn:aws:iam::807380035085:policy/AmazonEKS_EBS_CSI_Driver_Policy --role-name AmazonEKS_EBS_CSI_DriverRole
```

- Amazon EBS CSI 드라이버를 EKS에 배포한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
serviceaccount/ebs-csi-controller-sa created
serviceaccount/ebs-csi-node-sa created
clusterrole.rbac.authorization.k8s.io/ebs-csi-node-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-attacher-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-provisioner-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-resizer-role created
clusterrole.rbac.authorization.k8s.io/ebs-external-snapshotter-role created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-attacher-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-node-getter-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-provisioner-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-resizer-binding created
clusterrolebinding.rbac.authorization.k8s.io/ebs-csi-snapshotter-binding created
deployment.apps/ebs-csi-controller created
poddisruptionbudget.policy/ebs-csi-controller created
daemonset.apps/ebs-csi-node created
csidriver.storage.k8s.io/ebs.csi.aws.com created
```

- ebs-csi-controller-sa Kubernetes 서비스 계정에 이전에 생성한 IAM 역할의 Amazon 리소스 이름(ARN)으로 주석을 단다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  kubectl annotate serviceaccount ebs-csi-controller-sa -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::807380035085:role/AmazonEKS_EBS_CSI_DriverRole

serviceaccount/ebs-csi-controller-sa annotated
```

- 드라이버 Pod를 삭제한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  kubectl delete pods -n kube-system -l=app=ebs-csi-controller
pod "ebs-csi-controller-6b5749787-7xq8h" deleted
pod "ebs-csi-controller-6b5749787-ncs5m" deleted
```

역할에 할당된 IAM 정책의 IAM 권한으로 드라이버 파드가 자동으로 재배포된다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  kubectl get pods -n kube-system
NAME                                 READY   STATUS    RESTARTS   AGE
aws-node-7jtcw                       1/1     Running   0          98d
aws-node-g8gmp                       1/1     Running   0          98d
aws-node-jx2hx                       1/1     Running   0          98d
aws-node-twxl2                       1/1     Running   0          98d
aws-node-v55cn                       1/1     Running   0          98d
aws-node-v7d5d                       1/1     Running   0          98d
aws-node-zjqd6                       1/1     Running   0          98d
aws-node-zvz54                       1/1     Running   0          98d
coredns-6dbb778559-5rkhf             1/1     Running   0          98d
coredns-6dbb778559-6tbpq             1/1     Running   0          90d
ebs-csi-controller-6b5749787-4c8fb   6/6     Running   0          36s
ebs-csi-controller-6b5749787-9z5q9   6/6     Running   0          36s
ebs-csi-node-7nmpb                   3/3     Running   0          6m48s
ebs-csi-node-jbbqd                   3/3     Running   0          6m48s
ebs-csi-node-kcz2s                   3/3     Running   0          6m48s
ebs-csi-node-n6zm6                   3/3     Running   0          6m48s
ebs-csi-node-rx9zx                   3/3     Running   0          6m48s
ebs-csi-node-s9jd9                   3/3     Running   0          6m48s
ebs-csi-node-t79sx                   3/3     Running   0          6m48s
ebs-csi-node-wp2nc                   3/3     Running   0          6m48s
kube-proxy-8495h                     1/1     Running   0          98d
kube-proxy-b4222                     1/1     Running   0          98d
kube-proxy-fg6cn                     1/1     Running   0          98d
kube-proxy-ftfbh                     1/1     Running   0          98d
kube-proxy-k5h6z                     1/1     Running   0          98d
kube-proxy-kvs4x                     1/1     Running   0          98d
kube-proxy-x8nm5                     1/1     Running   0          98d
kube-proxy-zrtz4                     1/1     Running   0          98d
```

### 드라이버 테스트

동적 프로비저닝을 사용하는 어플리케이션으로 Amazon EBS CSI 드라이버를 테스트할 수 있다. Amazon EBS 볼륨은 온디맨드로 프로비저닝된다.

- AWS GitHub에서 aws-ebs-csi-driver 리포지토리를 복제한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
```

- 작업 디렉토리를 Amazon EBS 드라이버 테스트 파일이 포함된 폴더로 변경한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc  cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning
```

- 테스트에 필요한 Kubernetes 리소스를 생성한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl apply -f manifests
persistentvolumeclaim/ebs-claim created
pod/app created
storageclass.storage.k8s.io/ebs-sc created
```

위의 명령은 StorageClass, PersistentVolumeClaim(PVC) 및 Pod를 생성한다. Pod는 PVC를 참조한다.  
Amazon EBS 볼륨은 Pod가 생성될 때만 프로비저닝된다.

- ebs-sc 스토리지 클래스를 조회한다.

```
storageclass.storage.k8s.io/ebs-sc created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl describe storagecla
ss ebs-sc
Name:            ebs-sc
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           ebs.csi.aws.com
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

- 기본 네임 스페이스 Pod를 보고 app Pod의 상태가 Running 으로 변경될 때가지 기다린다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl get pods --watch
NAME                             READY   STATUS    RESTARTS   AGE
app                              1/1     Running   0          3m35s
elasticsearch-54bc8d7c9d-x8hm8   1/1     Running   0          56d
```

- PVC를 참조하는 Pod로 인해 생성된 영구 볼륨을 확인한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-6acf5765-6b8f-48d3-ba0a-e0967daa6ca5   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  4m20s
```

- 영구 볼륨에 대한 정보를 확인한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl describe pv pvc-6acf5765-6b8f-48d3-ba0a-e0967daa6ca5
Name:              pvc-6acf5765-6b8f-48d3-ba0a-e0967daa6ca5
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: ebs.csi.aws.com
Finalizers:        [kubernetes.io/pv-protection external-attacher/ebs-csi-aws-com]
StorageClass:      ebs-sc
Status:            Bound
Claim:             default/ebs-claim
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          4Gi
Node Affinity:
  Required Terms:
    Term 0:        topology.ebs.csi.aws.com/zone in [ap-northeast-2c]
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            ebs.csi.aws.com
    FSType:            ext4
    VolumeHandle:      vol-0fcea2473d799c81c
    ReadOnly:          false
    VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1661828025957-8081-ebs.csi.aws.com
Events:                <none>
```

- Pod가 볼륨에 데이터를 쓰고 있는지 확인한다.

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl exec -it app -- c
at /data/out.txt
Tue Aug 30 02:58:25 UTC 2022
Tue Aug 30 02:58:30 UTC 2022
```

- 테스트 Pod 삭제

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning   master  kubectl delete -f manifests
persistentvolumeclaim
pod "app" deleted
storageclass.storage.k8s.io "ebs-sc" deleted
```

### Pod와 Volume에서 사용된 샘플 Yaml

- pod에 배포 할 때 volumes 선언 부분

```
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-claim
```

- StorageClass

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

- PersistentVolumeClaim : 용량 선언

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
```

- systems-dev-ns에 적용된 스토리지

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
  namespace: systems-dev-ns
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
  namespace: systems-dev-ns
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 40Gi
```

# Elasticsearch EKS 구성하기

- Elasticsearch : docker.elastic.co/elasticsearch/leasticsearch:7.12.0
- Kibana : docker.elastic.co/kibana/kibana:7.12.0

## EKS 구성시 필요한 항목

- Ingress
  인그래스는 클러스터 외부에서 클러스터 내부 서비스로 HTTP와 HTTPS 경로를 노출한다. 즉 Pod를 호출 할 수 있는 도메인 주소.
- StateFulSet
  어플리케이션의 stateFul을 관리하는데 사용하는 워크로드 API 오브젝트이다. Pods 집합의 Deployment와 Scaling을 관리하며, Pods의 순서 및 고유성을 보장한다.  
  즉, ES의 내부 클러스터링과 Kibana의 클러스터링이 연결이 되려면 해당 방식을 사용해야 한다.
- ConfigMap
  Key-Value로 기밀이 아닌 데이터를 저장하는데 사용하는 API 오브젝트이다. Pods는 볼륨에서 환경 변수, 커맨드-라인 인수 또는 구성파일로 컨피그맵을 사용할 수 있다.  
  컨테이너 이미지에서 환경별 구성을 분리하여, 어플리케이션에 이식할 수 있다.
- HPA (Horizontal Pod Autoscaler)
  CPU 사용량을 관찰하여 replication controller, deployment, replicaSet, statefulSet의 Pods 개수를 자동으로 스케일링 한다.

- Service
  Pods 집합에서 실행중인 어플리케이션을 네트워크 서비스로 노출하는 추상화 방법.  
  즉, 외부 DNS를 지정하고 로드밸런스를 여기서 수행한다.

- PV (Persistent Volumes)
  노드가 클러스터 리소스인 것처럼 PV는 클러스터 리소스. 즉, Pods의 외부 적재 스토리지.

## [Elasticsearch] ConfigMap 구성하기

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: systems-dev-ns
  name: elasticsearch-config
  labels:
    app: elasticsearch-node
    role: master
data:
  elasticsearch.yml: |-
    cluster.name: ${cluster.name}
    node.name: ${node.name}
    discovery.seed_hosts: ${discovery.seed_hosts}
    cluster.initial_master_nodes: ${cluster.initial_master_nodes}
    network.host: 0.0.0.0
    xpack.security.enabled: false
    xpack.monitoring.enabled: true
    xpack.monitoring.collection.enabled: true
```

- metadata.labels.role
  라벨롤이 master이 라벨끼리 엮는다.
- ${변수명}
  deployment에서 지정한 변수를 지정한다는 뜻이다.
- xpack.security.enabled
  Elasticsearch security 설정 여부  
  EKS 자체에서 외부 IP 접근이 안되므로 현재는 false로 지정.
- xpack.monitoring.enabled, xpack.monitoring.collection.enabled
  Kibana에서 Stack monitoring이 되어야 하므로 true

## [Elasticsearch] HPA 구성하기

Elasticsearch에서 클러스터링 구성시, 1대는 마스터 노드여야 하고, 나머지 2대는 이중화 구성이 되어야 하므로 권장하는 최소 클러스터링 구성은 3이다.  
최소 3대에서 최대 5대까지 클러스터링이 구성 된다.

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-hap
  namespace: systems-dev-ns
spec:
  scaleTargetRef:
    apiVersion: app/v1
    kind: Deployment
    name: elasticsearch-node
  minReplicas: 3
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

- elasticsearch-node
  HPA 구성이 적용되는 Pods의 이름을 명시한다.

## [Elasticsearch] Service 구성하기

```
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-node
  namespace: systems-dev-ns
spec:
  selector:
    app: elasticsearch-node
  type: NodePort
  ports:
    - name: serving
      protocol: TCP
      port: 9200
    - name: node-to-node
      protocol: TCP
      port: 9300
```

- elasticsearch-node
  여기서 명시한 이름으로 추후 Kibana에서 내부 통신을 할 때 IP 주소 값대신 사용한다 ( 서비스명:9200 포트 )

## [Elasticsearch] Deployment(StatefullSet) 구성하기

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-node
  labels:
    app: elasticsearch-node
    role: master
  namespace: systems-dev-ns
spec:
  serviceName: elasticsearch-node
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch-node
      role: master
  template:
    metadata:
      labels:
        app: elasticsearch-node
        role: master
    spec:
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:7.12.0
          imagePullPolicy: Always
          resources:
            limits:
              cpu: 2000m
              memory: 8Gi
            requests:
              cpu: 1000m
              memory: 4Gi
          ports:
            - containerPort: 9200
              name: rest
              protocol: TCP
            - containerPort: 9300
              name: inter-node
              protocol: TCP
          env:
            - name: cluster.name
              value: elasticsearch-cluster
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: discovery.seed_hosts
              value: "elasticsearch-node"
            - name: cluster.initial_master_nodes
              value: "elasticsearch-node-0,\
                elasticsearch-node-1,\
                elasticsearch-node-2"
            - name: ES_JAVA_OPTS
              value: "-Xms2048m -Xmx2048m"
          volumeMounts:
            - name: persistent-storage
              mountPath: "/usr/share/elasticsearch/data"
      initContainers:
        - name: fix-permissions
          image: busybox:1.32.0-musl
          imagePullPolicy: Always
          command:
            ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          securityContext:
            privileged: true
          volumeMounts:
            - name: persistent-storage
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: busybox:1.32.0-musl
          imagePullPolicy: Always
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: busybox:1.32.0-musl
          imagePullPolicy: Always
          command: ["sh", "-c", "ulimit -n 65536"]
          securityContext:
            privileged: true
  volumeClaimTemplates:
    - metadata:
        name: persistent-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ebs-sc
        resources:
          requests:
            storage: 5Gi

```

- discovery.seed_hosts
  클러스터링을 구성하기 위해 명시된 name을 지정한다.  
  statefulset의 대표이름을 명시 해야 한다.
- cluster.initial_master_nodes
  master_node가 될 수 있는 후보군을 명시한다.  
  statefulset에 의해 부여 받은 이름을 명시한다.
  hpa 지정을 최소 3개로 하였으니 후보군을 고정적인 0 ~ 2 범위로 잡는다.
- volumes
  외부 볼륨으로 스토리지 사용

## [Kibana] ConfigMap 구성하기

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: systems-dev-ns
  name: kibana-config
  labels:
    app: kibana
    role: master
data:
  kibana.yml: |-
    elasticsearch:
      hosts: ${ELASTICSEARCH_HOSTS}
      username: minsoub@bithumbsystems.com
      password: wjdalstjq1!
```

- ELASTICSEARCH_HOST
  deployment의 env에 정의되어 있다.

## [Kibana] HPA 구성하기

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: kibana-hpa
  namespace: systems-dev-ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kibana-node
  minReplicas: 1
  maxReplicas: 1
  targetCPUUtilizationPercentage: 50
```

## [Kibana] Service 구성하기

```
apiVersion: v1
kind: Service
metadata:
  name: kibana-service
  namespace: systems-dev-ns
spec:
  selector:
    app: kibana
  type: NodePort
  ports:
    - protocol: TCP
      port: 5601
      targetPort: 5601
```

## [Kibana] Deployment 구성하기

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: systems-dev-ns
  labels:
    app: kibana
    role: master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
      role: master
  template:
    metadata:
      labels:
        app: kibana
        role: master
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:7.12.0
          imagePullPolicy: Always
          resources:
            limits:
              cpu: 3000m
              memory: 6Gi
            requests:
              cpu: 2000m
              memory: 4Gi
          env:
            - name: ELASTICSEARCH_HOSTS
              value: http://elasticsearch-node:9200
          ports:
            - containerPort: 5601

```

- ELASTICSEARCH_HOSTS
  ES의 Service에서 명시한 내부 이름을 해당 부분에다가 명시한다.  
  Kibana와 ES의 통신은 9200 (만약 NodePort를 다르게 작성하였다면 변경)

## fluentd Daemonset

fluentd를 쿠버네티스 환경에서 실행할 때는 DaemonSet 형식으로 등록하여 실행한다.

- Daemonset 이란
  데몬셋은 특정 파드를 모든 노드에서 실행되도록 하는 워크로드이다. 노두가 추가되면 파드가 자동으로 등록되어 데몬셋을 제거하면 데몬셋이 생성한 파드들도 삭제된다.

데몬셋을 사용하면 모든 노드에서 fluentd 파드의 복사본을 사용할 수 있다. fluentd는 로그 수집기 역할을 하면서 로그를 데이터 저장소로 옮기는 역할도 한다.

### fluentd Daemonset.yaml

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      initContainers:
        - name: config-fluentd
          image: busybox
          imagePullPolicy: Always
          command: ["/bin/sh","-c"]
          args:
            - cp /fluentd/etc2/fluent.conf /fluentd/etc/fluent.conf;
              cp /fluentd/etc2/kubernetes.conf /fluentd/etc/kubernetes.conf;
          volumeMounts:
            - name: config-path
              mountPath: /fluentd/etc
            - name: config-source
              mountPath: /fluentd/etc2
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.3.0-debian-elasticsearch
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch-node.systems-dev-ns.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
            - name: FLUENT_UID
              value: "0"
          resources:
            limits:
              memory: 2048Mi
            requests:
              cpu: 1024Mm
              memory: 2048Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: dockercontainerlogdirectory
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config-path
              mountPath: /fluentd/etc
      terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: dockercontainerlogdirectory
          hostPath:
            path: /var/lib/docker/containers
        - name: config-source
          configMap:
            name: fluentd-config
        - name: config-path
          emptyDir: {}
```

## fluentd - ConfigMap

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
data:
  fluent.conf: |
    #@include systemd.conf
    @include kubernetes.conf
    @include kubernetes2.conf
    <match **>
      @type elasticsearch
      @id out_es
      @log_level info
      include_tag_key true
      host elasticsearch-node.systems-dev-ns.svc.cluster.local
      port 9200
      scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
      ssl_verify "#{ENV['FLUENT_ELASTICSEARCH_SSL_VERIFY'} || 'true'}"
      reoload_connections "#{ENV['FLUENT_ELASTICSEARCH_RELOAD_CONNECTIONS'] || 'true'}"
      logstash_prefix "#{ENV['FLUENT_ELASTICSEARCH_LOGSTASH_PREFIX'] || 'logstash'}"
      logstash_format true
      type_name fluentd
      buffer_chunk_limit 2M
      buffer_queue_limit 32
      flush_interval 5s
      max_retry_wait 30
      disable_retry_limit
      num_threads 8
    </match>
  kubernetes.conf: |
    <match fluent.**>
      @type null
    </match>
    <source>
      @type tail
      @id in_tail_container_logs
      path "/app/log/spring.log"
      pos_file "/app/log/fluentd-containers.log.pos"
      tag "kubernetes.*"
      read_from_head true
      <parse>
         @type "regexp"
         expression ^(?<logtime>[^\]*) (?<loglevel>\w+) (?<process_id>\d+) --- \[(?<thread_name>[^\]]*)\] (?<logger_name>[^\]]*)  : (?<log_text>[^\]]*)
         time_format "%Y-%m-%dT%H:%M:%S.% NZ"
         time_type string
      </parse>
    </source>
```

## fluentd - Cluster Role, ClusterRoleBinding, ServiceAccount

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: kube-system
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - namespaces
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: fluentd
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system

```

## EFK 생성 절차

### 영구 스토리지 생성 (PV)

```
ms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f ebs_volume_create.yaml
storageclass.storage.k8s.io/ebs-sc created
persistentvolumeclaim/ebs-claim created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get storageclass
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ebs-sc          ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  2m13s
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl describe storageclass ebs-sc
Name:            ebs-sc
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           ebs.csi.aws.com
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

### ElastichSearch

- ConfigMap 생성

```
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f elasticsearch_configmap.yaml
configmap/elasticsearch-config created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get configmap -n systems-dev-ns
NAME                                  DATA   AGE
aws-load-balancer-controller-leader   0      91d
elasticsearch-config                  1      80s
ingress-controller-leader-nginx       0      33d
kube-root-ca.crt                      1      99d
```

- HPA (Horizontal Pods AutoScaler)

```
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f elasticsearch_hpa.yaml
horizontalpodautoscaler.autoscaling/elasticsearch-hap created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get hpa -n systems-dev-ns
NAME                REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
elasticsearch-hap   Deployment/elasticsearch-node   <unknown>/50%   3         5         0          29s
```

- Service 생성

```
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f elasticsearch_service.yaml
service/elasticsearch-node created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get svc  -n systems-dev-ns
elasticsearch-node                  NodePort       172.20.49.58     <none>                  9200:30822/TCP,9300:30004/TCP   101s
```

- StatefullSet 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f elasticsearch_statefulset.yaml
statefulset.apps/elasticsearch-node created

jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get pods -w -l app=elasticsearch-node -n systems-dev-ns
NAME                   READY   STATUS     RESTARTS   AGE
elasticsearch-node-0   1/1     Running    0          65s
elasticsearch-node-1   0/1     Init:2/3   0          25s
elasticsearch-node-1   0/1     PodInitializing   0          25s
elasticsearch-node-1   1/1     Running           0          38s
elasticsearch-node-2   0/1     Pending           0          0s
elasticsearch-node-2   0/1     Pending           0          4s
elasticsearch-node-2   0/1     Init:0/3          0          4s
elasticsearch-node-2   0/1     Init:1/3          0          26s
elasticsearch-node-2   0/1     Init:2/3          0          29s
elasticsearch-node-2   0/1     PodInitializing   0          32s
elasticsearch-node-2   1/1     Running           0          45s

jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get statefulset -n systems-dev-ns
NAME                 READY   AGE
elasticsearch-node   3/3     41m
```

위에서 생성된 스토피지 내역 확인

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get pv -n systems-dev-ns
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                    STORAGECLASS   REASON   AGE
pvc-ba277c5d-aac8-4fbe-be2f-8a142e0f5180   5Gi        RWO            Delete           Bound    systems-dev-ns/persistent-storage-elasticsearch-node-1   ebs-sc                  41m
pvc-bde46146-4c46-49f5-8fab-9310c5992a4a   5Gi        RWO            Delete           Bound    systems-dev-ns/persistent-storage-elasticsearch-node-0   ebs-sc                  41m
pvc-ec13136a-9c60-4c8f-a5c2-e1f59a7eaa37   5Gi        RWO            Delete           Bound    systems-dev-ns/persistent-storage-elasticsearch-node-2   ebs-sc                  40m
```

### Kibana

- ConfigMap 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f kibana_configmap.yaml
configmap/kibana-config created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get configmap -n systems-dev-ns
NAME                                  DATA   AGE
aws-load-balancer-controller-leader   0      92d
elasticsearch-config                  1      5h54m
ingress-controller-leader-nginx       0      34d
kibana-config                         1      10s
kube-root-ca.crt                      1      99d
```

- HPA 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f kibana_hpa.yaml
horizontalpodautoscaler.autoscaling/kibana-hpa created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get hpa -n systems-dev-ns
NAME                REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
elasticsearch-hap   Deployment/elasticsearch-node   <unknown>/50%   3         5         0          5h53m
kibana-hpa          Deployment/kibana-node          <unknown>/60%   1         1         0          11s
```

- Service 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f kibana_service.yaml
service/kibana-service created
```

- Deployment 생성

```
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f kibana_deployment.yaml
deployment.apps/kibana created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get pods -n systems-dev-ns -l app=kibana
NAME                      READY   STATUS    RESTARTS   AGE
kibana-54bbf69758-blm6j   1/1     Running   0          38s
```

application 명으로 찾고자 할 때 -l app=kibana 형식을 취한다.

### Fluentd 구성

- ServiceAccount 생성, ClusterRole 및 ClusterRoleBinding 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f fluentd_clusterrole.yaml
clusterrole.rbac.authorization.k8s.io/fluentd created
clusterrolebinding.rbac.authorization.k8s.io/fluentd created
serviceaccount/fluentd unchanged

 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get sa -n kube-system
  jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get cr -n kube-system
```

- ConfigMap 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f fluentd_configmap.yaml
configmap/fluentd-config created
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get configmap -n kube-system
NAME                                      DATA   AGE
aws-auth                                  2      100d
cattle-controllers                        0      35d
cert-manager-cainjector-leader-election   0      29d
cert-manager-controller                   0      36d
coredns                                   1      100d
cp-vpc-resource-controller                0      100d
eks-certificates-controller               0      100d
extension-apiserver-authentication        6      100d
fluentd-config                            2      57s
kube-proxy                                1      100d
kube-proxy-config                         1      100d
kube-root-ca.crt                          1      100d
```

- DaemonSet 생성

```
jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl apply -f fluentd_daemonset.yaml
daemonset.apps/fluentd created

jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/EFK_configuration/deploy  kubectl get daemonset -n kube-system
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
aws-node       8         8         8       8            8           <none>                   100d
ebs-csi-node   8         8         8       8            8           kubernetes.io/os=linux   34h
fluentd        8         8         0       8            0           <none>                   46s
kube-proxy     8         8         8       8            8           <none>                   100d
```
