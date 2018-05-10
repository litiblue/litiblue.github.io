---
layout: post
title:  "AWS에서 Kubernetes클러스터 구축하기 (2) - NFS"
date:   2018-04-27 15:50:00 +0900
categories: dev
---

Kubernetes에서 NFS를 이용하여 컨테이너 간 파일 공유가 가능하도록 합니다.
이름이 `mynfs`인 PersistentVolume과 PersistentVolumeClaim을 각각 생성하고 동일한 볼륨을 사용하는 두 개의 컨테이너 `mypod`과 `mypod-2nd`를 생성합니다.
`mypod`에서 볼륨 디렉토리에 `test_file`이라는 파일을 만든 다음 `mypod-2nd`에서 해당 파일이 동일하게 생성되었는지 확인 합니다.
CMD-client에서도 NFS를 마운트 하여 `test_file`을 확인합니다.

### NFS서버 설치

AWS에서 제공하는 NFS 서버인 `EFS`를 설치하여 사용합니다.

#### 보안 그룹 추가

NFS를 위한 보안 그룹을 만들고 아래 목록에 해당하는 보안 그룹에 속한 EC2 인스턴스에서 접근 가능하도록 인바운드 규칙을 추가합니다.

- CMD-client 보안 그룹
- Kubernetes 마스터 노드 보안 그룹 (`masters.dev.example.com`)
- Kubernetes 나머지 노드 보안 그룹 (`nodes.dev.example.com`)

유형을 NFS로 하고 소스를 각 보안 그룹으로 설정하면 됩니다.

![](/assets/img/2018/05/efs-securitygroup.png)
*NFS 보안 그룹 추가*

#### EFS 설치

AWS 콘솔의 서비스 -> EFS -> `Create file system`으로 들어갑니다.

Security Groups에 위에서 만들어둔 NFS 보안 그룹을 추가합니다. 이를 통해 CMD-client와 Kubernetes 노드에서 NFS 서버에 접속 가능합니다.

![](/assets/img/2018/05/create_efs.png)
*EFS*

### Kubernetes 볼륨 및 컨테이너 생성

NFS서버의 `/`디렉토리를 PV에 등록한 뒤 각 Pod에서 `/mymnt`디렉토리에 볼륨을 마운트하여 `mypod`과 `mypod-2nd`가 같은 디렉토리를 공유합니다.

#### PersistentVolume `mynfs` 생성

```shell
$ cat pv-volume-nfs.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mynfs
spec:
  capacity:
    storage: 1Gi
  storageClassName: ""
  accessModes:
    - ReadWriteMany
  nfs:
    server: fs-xxxxxxxx.efs.us-east-x.amazonaws.com
    path: "/"
```

```
$ kubectl create -f pv-volume-nfs.yaml
persistentvolume "mynfs" created
```

#### PersistentVolumeClaim `mynfs` 생성

```shell
$ cat pvc-volume-nfs.yaml
```

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mynfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
  storageClassName: ""
```

```
$ kubectl create -f pvc-volume-nfs.yaml
persistentvolumeclaim "mynfs" created
```

#### `mypod` 생성

```shell
$ cat pv-pod.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  volumes:
    - name: myvol
      persistentVolumeClaim:
       claimName: mynfs
  containers:
    - name: my-container
      image: python
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/mymnt"
          name: myvol
```

```
$ kubectl create -f pv-pod.yaml
pod "mypod" created
```

#### mypod-2nd 생성

설정 파일 내용은 name을 제외한 모든 내용이 `mypod`과 동일합니다.

```shell
$ cat pv-pod-2nd.yaml
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod-2nd
spec:
  volumes:
    ...
    ...
```

```
$ kubectl create -f pv-pod-2nd.yaml
pod "mypod-2nd" created
```

### 파일 공유 테스트

`mypod`에서 생성한 파일이 `mypod-2nd`와 `CMD-client`에 공유 되는지 확인 합니다.

#### `mypod`에서 `test_file`파일 생성

```
$ kubectl exec -it mypod -- /bin/bash

root@mypod:/# cd /mymnt
root@mypod:/mymnt# touch test_file
```

#### `mypod-2nd`에서 `test_file`파일 확인
```
$ kubectl exec -it mypod-2nd -- /bin/bash

root@mypod-2nd:/# ls /mymnt/
test_file
```

#### `CMD-client`에서 `test_file`파일 확인

```
$ sudo mkdir efs
$ sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 fs-xxxxxxxx.efs.us-west-x.amazonaws.com:/ efs

$ ls ./efs/
test_file
```

### 마치며

지난번 포스트를 통해 AWS에서 Kubernetes를 설치하는 방법을 알아보았고 이번에는 컨테이너(Pod)와 볼륨(PV,PVC)이 제대로 생성되고 NFS를 통해 컨테이너 간 파일 공유가 정상적으로 이루어지는지 확인하였습니다. AWS 및 Kubernetes 사용에 참고 바랍니다.
