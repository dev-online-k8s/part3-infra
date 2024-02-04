# Infra Setting

## Disclaimer
- AWS에서 EKS, EC2(Node Group), RDS를 사용하는 과정은 사용한 시간에 따라 일정한 비용을 소모하게 됩니다. 따라서 인프라를 생성하여 실습하신 뒤 해당 인프라를 그대로 방치해두면 많은 비용이 청구될 수 있으므로 가급적이면 전체 실습 과정을 한 번에 수행하시고 실습에 사용했던 인프라를 모두 제거해주시기 바라겠습니다.
- 비용 청구가 부담스러우신 분은 기존 실습에 사용하셨던 kind를 기반으로 한 로컬 환경을 그대로 사용하셔도 상관 없습니다.
  - 이 경우 저장소는 별도의 Storage Class 생성 없이 emptydir로 지정하여 사용하여 주세요. (image-server는 1개로 유지해주세요)
  - Database는 MySQL을 로컬 환경 혹은 Helm을 이용해 설치하여 사용해주세요.

## Chapter 1 : EKS Cluster 생성

### IAM Role 생성
- eks-cluster-role
  - IAM - Roles - Create Role
  - AWS Service에서 EKS - Cluster Use Case 선택
  - Add Permission에서 AmazonEKSClusterPolicy 추가되어 있는 것 확인하고 Next
  - Create Role 눌러서 생성 완료
- eks-node-role
  - IAM - Roles - Create Role
  - AWS Service에서 EC2 선택
  - Add Permission에서 다음 3가지 Permission 추가
    - AmazonEC2ContainerRegistryReadOnly
    - EKSWorkerNodePolicy
    - EKSCniPolicy
  - Create Role 눌러서 생성 완료

### Access Key 설정
- User - 현재 계정 선택
- Security Credentials - Access Key
- Create Access Key
- Use Case에서 CLI 선택
- Access Key / Secret Key 생성
- `aws configure`명령으로 Access Key , Secret Key 설정하여 CLI 설정

### EKS Cluster 생성
- Region 확인 (한국의 경우 ap-northeast-2 서울 리전 선택)
- EKS - Add Cluster - Create
- Cluster 이름 : sns-cluster
- Cluster Service Role : eks-cluster-role
- 다른 설정은 모두 기본값 사용하여 생성
  - 별도의 VPC 설정이 있는 경우 해당 VPC에 설정
- 클러스터 생성 후 Active 상태가 될때까지 기다리기

### EKS Node Group 생성
- 생성된 클러스터에서 Compute - Add node group
- Node group 이름 : sns-node
- Node IAM role : eks-node-role
- AMI type : Amazon Linux 2 (x86_64)
- Instance Type : t3.medium
- Desired, Minimum, Maximum size : 2
- 생성 후 노드 추가 완료될때까지 잠시 기다리기

### kubectl 컨텍스트 추가
- `aws eks update-kubeconfig --region ap-northeast-2 --name sns-cluster`
- `kubectl get nodes` 명령어로 실제 노드 목록 나오는지 확인

### 참고
[AWS EKS 설치 가이드](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/getting-started-console.html)

## EFS Storage 연결

## IAM Role 생성
- EKS - cluster에서 sns-cluster 선택
- OpenID Connect provider URL 복사
- IAM - Identity Providers tjsxor
- OpenID Connect 선택
- Provider URL에 EKS에서 복사한 OpenID URL 붙여넣고 Get Thumbprint 클릭
- Audience에 `sts.amazonaws.com` 입력
- Add Provider 눌러서 생성
- IAM - Roles - Create Role
- Web Identity 선택
- Identity Provider에서 OpenID URL 선택
- Audience에서 sts.amazonaws.com 선택 후 Next
- Permission에서 AmazonEFSCSIDriverPolicy 검색해서 선택 후 Next
- AmazonEKS_EFS_CSI_DriverRole으로 Role 이름 주고 생성
- IAM - Roles에서 AmazonEKS_EFS_CSI_DriverRole 선택하고 Trust Relationships 탭에서 Edit Trust Policy 서택
- Conditiond에서 `"oidc.eks.ap-northeast-2.amazonaws.com/id/OOOOOOOO:aud": "sts.amazonaws.com"`로 시작하는 한 줄 복사해서 붙여넣은 다음에 `"oidc.eks.ap-northeast-2.amazonaws.com/id/OOOOOOOO:sub": "system:serviceaccount:kube-system:efs-csi-*"`와 같은 형태로 변경
  - `aud`를 `sub`로,
  - `sts.amazonaws.com`을 `system:serviceaccount:kube-system:efs-csi-*` 으로 변경
  - 최종적으로 aud, sub 2개의 컨디션이 있어야 함
- Condition의 `StringEquals`를 `StringLike`으로 변경 후 저장

### VPC Security Group 수정
- VPC - Security Group
- 클러스터에서 사용하는 Security Group 선택
  - 별다른 설정을 하지 않았을 경우 Default Security Group
- Inbound Rules에서 Edit Inbound Rule선택
- 클러스터의 서브넷이 사용하는 대역에 대해 NFS(2049) 포트 추가 후 저장
  - 별다른 설정을 하지 않은 경우 172.31.0.0/16

### EFS Add-On 추가
- EKS - clusters - sns-cluster 선택
- Add-on 탭에서 Get more add-ons 클릭
- Amazon EFS CSI Driver 선택 후 설치
  - Role은 반드시 AmazonEKS_EFS_CSI_DriverRole 선택

### EFS FileSystem 생성
- EFS - File Systems - Create File System
  - 이름 : efs-volume
  - VPC : sns-cluster가 설치된 VPC
- 생성 후 File system ID 복사

### Storage 클래스 추가

- 다음과 같이 StorageClass 파일 생성
- fileSystemId는 EFS에서 복사한 ID로 추가

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: [EFS 파일시스템 ID]
  directoryPerms: "700"
```

- `kubectl apply -f efs-sc.yaml` 명령어로 StorageClass 생성

### 참고
[EFS CSI Driver 설치 가이드](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/efs-csi.html)

## MySQL DB 설정

### RDS Database 생성
- RDS - Create database
- Easy create - MySQL
- DB Instance size : Free Tier
- DB instance identifier : sns-db
- Master username : admin
- Master password : 임의의 비밀번호
- RDS 생성 기다린 뒤, Security Group 선택
- Inbound rules - Edit inbound rules
- MYSQL/Aurora 포트에 대해서 172.31.0.0/16 혹은 eks security group 추가 후 저장

### Schema 생성

- mysql 클라이언트 파드 생성
```
kubectl run mysql-client --image=mysql:8 -it --rm -- bash
```
- mysql 클라이언트 실행
```
mysql -h [RDS Endpoint] -u admin -p
(패스워드 입력)
```
- DDL 실행
```sql
create database sns;
use sns;

create user 'sns-server'@'%' identified by 'password!';
grant all privileges on sns.* to 'sns-server'@'%';

create table social_feed
(
    feed_id         int auto_increment
        primary key,
    image_id        varchar(255)                       not null,
    uploader_id     int                                not null,
    upload_datetime datetime default CURRENT_TIMESTAMP null,
    contents        text                               null
);

create table user
(
    user_id  int auto_increment
        primary key,
    username varchar(255) not null,
    email    varchar(255) not null,
    password varchar(255) not null
);

create table follow
(
    follow_id       int auto_increment
        primary key,
    user_id         int                                not null,
    follower_id     int                                not null,
    follow_datetime datetime default CURRENT_TIMESTAMP null
);
```

## Redis, Kafka, DB Service 설치
- Namespace
```sh
kubectl create namespace infra
```

- Redis
```sh
helm -n infra install redis oci://registry-1.docker.io/bitnamicharts/redis --set architecture=standalone --set auth.enabled=false --set master.persistence.enabled=false
```

- Kafka
```sh
helm -n infra install kafka oci://registry-1.docker.io/bitnamicharts/kafka --set controller.replicaCount=3  --set sasl.client.passwords=kafkakafka123! --set controller.persistence.enabled=false --set broker.persistence.enabled=false
```

### External Name Service 설정

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: infra
spec:
  type: ExternalName
  externalName: [RDS MySQL DB주소]
```

### ECR Repository 생성
- ECR - private registry - Repositories - Create Repository
- 다음과 같이 Private 저장소 생성
  - feed-server
  - user-server
  - image-server
  - notification-batch
  - timeline-server
  - sns-frontend
- 각 저장소에 대해 Create Repository 눌러서 생성 완료

## Chapter 6 : Monitoring

### metrics-server 설치

- https://github.com/kubernetes-sigs/metrics-server에서 설치 방법 확인 가능
```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Prometheus와 Grafana 설치

- Helm Repository 추가
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```

- Namespace 생성
```sh
kubectl create namespace monitoring
```

- Prometheus 설치
```sh
helm install prometheus prometheus-community/prometheus --namespace monitoring  --set server.persistentVolume.enabled=false --set alertmanager.persistence.enabled=false
```

- grafana.yaml 생성
```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local
      access: proxy
      isDefault: true
```
- Grafana 설치
```sh
helm install grafana grafana/grafana  --namespace monitoring --set persistence.enabled=false --set adminPassword="admin01" --values ./grafana.yaml
```

### OpenLens 설치
- https://github.com/MuhammedKalkan/OpenLens/ 에서 릴리즈 다운로드 

### KubeCost 설치
```sh
helm upgrade -i kubecost oci://public.ecr.aws/kubecost/cost-analyzer --version 1.108.1 \
    --namespace monitoring --set persistentVolume.enabled=false --set prometheus.server.persistentVolume.enabled=false \
    -f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/develop/cost-analyzer/values-eks-cost-monitoring.yaml
```
- port-fowarding
```yaml
kubectl port-forward --namespace monitoring deployment/kubecost-cost-analyzer 9090
```

## Chapter 7: 성능 테스트 및 테스트 데이터 추가

### k6 설치
- https://k6.io 에서 다운로드 후 설치

### 테스트 데이터 추가
```sh
git clone https://github.com/dev-online-k8s/part3-testdatagen.git
```

```sh
SNS_DATA_GENERATOR_TELEPRESENCE_ENABLED=true java -jar TestDataGen.jar
```

### Frontend Deploy 배포

```sh
docker pull jheo/sns-frontend:1.0.0
docker tag jheo/sns-frontend:1.0.0 {ecr주소}/sns-frontend:1.0.0
docker push {ecr주소}/sns-frontend:1.0.0
```
- push에서 인증 오류 발생 시 ECR의 View push command 버튼 눌러서 로그인 방법 수행
  - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin {account id}.dkr.ecr.ap-northeast-2.amazonaws.com

- Deployment와 Service 생성 및 배포
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sns-frontend
  namespace: sns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sns-frontend
  template:
    metadata:
      labels:
        app: sns-frontend
    spec:
      containers:
        - name: sns-frontend-container
          image: {ecr주소}/sns-frontend:1.0.0
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sns-frontend-service
  namespace: sns
spec:
  selector:
    app: sns-frontend
  ports:
    - protocol: TCP
      port: 3000
```

### Ingress Controller 설치
- https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.html
- https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html



