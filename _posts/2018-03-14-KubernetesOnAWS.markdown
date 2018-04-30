---
layout: post
title:  "AWS에서 Kubernetes클러스터 구축하기"
date:   2018-03-14 15:50:00 +0900
categories: dev
---

AWS에서 Kubernetes 클러스터를 구축합니다.
Hyperledger Fabric를 이용하여 블록체인 네트워크를 구축하던 중 Kubernetes 관련 부분을 따로 정리해두면 좋을 것 같아 포스팅합니다.

전체 설치 과정은 이 곳(<https://hackernoon.com/how-to-deploy-hyperledger-fabric-on-kubernetes-1-a2ceb3ada078>)에서 확인 가능합니다. 설치 파일 공유를 위해 NFS를 사용하는데 그 부분은 별도의 포스트에 정리할 계획입니다.

#### 서버 구성
- 명령어 실행을 위한 CMD-client 서버 1개
- 총 5개 노드(Master 1개, Worker 4개)로 구성된 Kubernetes클러스터
- 파일 공유를 위한 NFS 서버 1개

CMD-client, Kubernetes 클러스터, NFS 서버가 모두 하나의 VPC안에 있어야합니다. VPC가 다른 경우 서로 연결하기 힘들 수도 있으니 주의하길 바랍니다.

#### CMD-client 생성
먼저 명령어 실행을 위한 EC2 인스턴스를 생성 합니다. 많은 리소스가 필요 없으므로 CPU 1개, 메모리 2GB의 t2.small EC2 인스턴스를 사용합니다. 설치 과정은 쉬우므로 생략합니다.

![](/assets/img/cmd_client.png)
*명령어 실행용 EC2 인스턴스*

#### 관리 tool 설치

CMD-client에서 다음과 같이 kubectl과 kops를 설치합니다.

kubectl 설치 (<https://kubernetes.io/docs/tasks/tools/install-kubectl/>)
```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

kops 설치 (<https://kubernetes.io/docs/getting-started-guides/kops/>)
```shell
wget https://github.com/kubernetes/kops/releases/download/1.8.0/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

#### DNS 설정

`example.com` 도메인을 가지고있는 경우 Route53 설정을 통해 `dev.example.com`을 사용할 수 있습니다.
서비스 -> Route53 -> `Hosted zones`으로 들어가 새로운 HostedZone을 생성합니다.

![](/assets/img/CreateHostedZone.png)
*dev.example.com 생성*

자동으로 생성된 2개의 Record Set중에 NS타입의 value를 복사합니다.
![](/assets/img/GetValues.png)
*Record Set 복사*

부모 도메인 `example.com`의 설정으로 들어가 `dev.example.com`의 Record Set을 등록해야 합니다.
Type을 NS로 지정한 뒤 복사했던 값을 Value에 붙여넣으면 됩니다.
부모 도메인을 AWS에서 구매한 경우에는 아래와 같이 설정할 수 있습니다.
비슷한 화면이지만 아래는 `example.com`도메인을 구입했던 다른 AWS계정의 Route53 설정으로 만약 AWS가 아닌 다른 곳에서 도메인을 구입했다면 그에 맞는 방법으로 등록해야 합니다.

![](/assets/img/CreateRecordSetOrigin.png)
*Record Set 등록*

`dig NS dev.example.com`명령을 실행했을 때 다음과 같이 4개의 NS가 나오면 됩니다.

```shell
$ dig NS dev.xiilabcoin.com

;; ANSWER SECTION:
dev.example.com.	60	IN	NS	ns-xxxx.awsdns-xx.co.uk.
dev.example.com.	60	IN	NS	ns-xxx.awsdns-xx.com.
dev.example.com.	60	IN	NS	ns-xxx.awsdns-xx.net.
dev.example.com.	60	IN	NS	ns-xxxx.awsdns-xx.org.
```

#### S3버킷 생성

CLI를 통해 AWS에 접근하기 위해서는 액세스 키가 필요합니다.
내 보안 자격 증명 -> 사용자 -> 아이디 클릭 -> 액세스 키 만들기 클릭을 통해 생성합니다.

![](/assets/img/SecurityCredentials.png)
*액세스 키 생성*

생성된 액세스 키를 다음과 같이 등록하여 사용합니다.

```shell
ubuntu@ip-172-xx-xx-xx:~$ aws configure
AWS Access Key ID [****************LPDQ]: xxxxxxxxxxxx
AWS Secret Access Key [****************UIXT]: xxxxxxxxxxxx
Default region name [us-east-x]:
Default output format [json]:
```

클러스터 설정 정보 저장을 위한 S3버킷을 생성합니다.

```shell
ubuntu@ip-172-xx-xx-xx:~$ aws s3 mb s3://dev.example.com
make_bucket: dev.example.com
```

#### kops 설정 및 실행

키교환을 위한 ssh키를 생성합니다.

```shell
ubuntu@ip-172-xx-xx-xx:~/.ssh$ ssh-keygen
Generating public/private rsa key pair.

Your identification has been saved in /home/ubuntu/.ssh/id_rsa.
Your public key has been saved in /home/ubuntu/.ssh/id_rsa.pub.
```

kops를 사용하기 위해 환경 변수를 설정합니다. VPC_ID와 NETWORK_CIDR은 AWS 콘솔의 `VPC대시보드`에서 확인할 수 있습니다.

```shell
$ export KOPS_STATE_STORE=s3://dev.example.com
$ export CLUSTER_NAME=dev.example.com
$ export VPC_ID=vpc-xxxxxxxx
$ export NETWORK_CIDR=172.xx.x.x/16
```

```shell
$ kops create cluster --zones=us-east-xx --name=${CLUSTER_NAME} --vpc=${VPC_ID}
```

```shell
$ kops edit cluster ${CLUSTER_NAME}
```

설정 파일에서 다음 부분을 찾아서 cidr을 AWS VPC의 서브넷 리스트에 없는 IP로 수정 합니다.

```yml
subnets:
- cidr: 172.xx.xxx.0/19
  name: us-east-xx
  type: Public
  zone: us-east-xx
```

```shell
$ kops edit ig --name=dev.example.com nodes
```

설정 파일이 열리면 4개가 생성 되도록 maxSize: 4, minSize: 4 로 설정합니다.

```shell
$ kops edit ig --name=dev.example.com master-us-east-xx
```

설정 파일이 열리면 원하는 부분을 수정합니다. 수정 사항이 없는 경우는 그대로 종료합니다.

```shell
$ kops update cluster dev.example.com --yes
```

#### 클러스터 생성 확인

클러스터가 제대로 생성되었는지 다음과 같이 확인 합니다.
master 노드 하나와 다른 노드 4개로 성공적으로 설치되었음을 알 수 있습니다.

```shell
$ kops get ${CLUSTER_NAME}

Cluster
NAME			CLOUD	ZONES
dev.example.com	aws	us-east-xx

Instance Groups
NAME			ROLE	MACHINETYPE	MIN	MAX	ZONES
master-us-east-xx	Master	m3.medium	1	1	us-east-xx
nodes			Node	t2.medium	4	4	us-east-xx
```

아래는 동일한 내용을 AWS 콘솔에서 확인한 것으로 EC2 인스턴스 5개가 성공적으로 실행 중 입니다.

![](/assets/img/ec2_instances.png)
*EC2 인스턴스 생성 확인*

### 마치며

Kubernetes는 성공적으로 설치되었으니 실제 도커 컨테이너를 실행하여 잘 동작하는지 확인하는 일만 남았습니다.
다음 포스트에서는 NFS로 PersistentVolume을 구축하여 PersistentVolumeClaim을 통한 파일 공유가 제대로 이루어지는지를 확인하겠습니다.
