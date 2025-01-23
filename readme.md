# 📢ⓒⓘⓒⓓ-ⓟⓞⓒ
<pre>
# # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
#  _____  _____  _____ ______      ______  _____  _____   #
# /  __ \|_   _|/  __ \|  _  \     | ___ \|  _  |/  __ \  #
# | /  \/  | |  | /  \/| | | | ___ | |_/ /| | | || /  \/  #
# | |      | |  | |    | | | ||___||  __/ | | | || |      #
# | \__/\ _| |_ | \__/\| |/ /      | |    \ \_/ /| \__/\  #
#  \____/ \___/  \____/|___/       \_|     \___/  \____/  #
#                                                         #
# # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
</pre>
> S●●P 기반 CICD의 간단한 버전을 Protytype of concept 개발 하기위한 과정.<br/>
> 각종 ECO-SYSTEM 설치시의 히스토리 및 팁을 남긴다. <br/>
> 이 이상 친절할 수 있을까?
> > <b>설치제원</b> <br/>
> > Windows 11 home & VirtualBox 7.1.4 <br/>
> > ```
> > vm cicd-m1 (마스터)
> > vm cicd-r1 (레파지토리)
> > vm cicd-w1 (워커1)
> > vm cicd-w2 (워커2)
> > vm cicd-w3 (워커3)
> > ```
> > - Ubuntu 20.04 LTS
> > - Docker 20.10.24
> > - kubekey v3.0.13
> > - kubernetes v1.26.5
> > - kubesphere v3.4.1
> > - GitLab 13.12.15
> > - HARBOR 2.5.3
> > - ArgoCD 2.10.6


## 1. 최초 VM 설치
### 1) Ubuntu 20.04 설치
#### ○ 대략적인 절차
1. ubuntu 20.04 iso를 미리 다운받자.
2. virtualbox network를 구성해둔다.
3. 가상머신을 알맞게 만들고
    + 2 Core / 2.5GB Memory / 60GB Disk.<br/>
    + 노트북 메모리 16G 기준, VM3대가 한계인듯.<br/>
    + POC구성에선 노트북2대를 이용하여 VM5대로 구성할 예정이다.
4. 네트워크 설정을 맞춰준 뒤
5. CD롬 드라이브에 다운받은  ubuntu-20.04 iso 파일 삽입
6. 시작하기 누르면 가상머신이 뜨면서 ubuntu 설치가 진행된다.
7. 아래 URL을 참고하여 설치마무리, 고정IP세팅, hostname, hosts, sshd 설정 등을 한다


##### ● VM 설치 참고 URL : [링크](https://proj.pe.kr/04dd6f45c25748538ef2e0a6e8902e50/50010d7ef42044aa9224db65fb36f45e.html)


### 2) 도커설치 (root로 진행)
#### ○ Install Docker Engine
```bash
apt-get update
apt-get install ca-certificates gnupg lsb-release
```
```bash
# Add Docker’s official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```bash
# Set up the repository:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```bash
apt-get update

# 도커 설치가능버전 확인
apt-cache madison docker-ce

# 20.10의 마지막버전으로 해보자 (최신버전에서 뭐가 잘 안되던 기억이 있어서)
apt-get install docker-ce=5:20.10.24~3-0~ubuntu-focal docker-ce-cli=5:20.10.24~3-0~ubuntu-focal containerd.io docker-compose-plugin

# 설치 버전 확인
docker --version
```
### 3) kubekey pre 설치 (root로 진행)
> kubekey 세팅까지 끝낸 뒤, VM 복제를 하려고 한다.
```bash
apt-get update
apt-get install -y socat conntrack ebtables ipset

# root로 swap disabled
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
```

## 2. VM 복제
### 1) VirtualBox에서 복제
1. VM을 기동중지 한 상태
2. VM리스트에서 오른쪽버튼 > 복제
3. 이름을 cicd-xx로 변경
4. 완전한 복제
5. 스냅샷은 Current Machine State
6. 맥주소정책 : 모든 네트워크 어댑터의 새 MAC주소 생성
7. 완료

### 2) 복제완료 후
1. 네트워크를 cicdNetwork로 세팅 하고<br/>
 <i>(추후에 어댑터에 브리지로 변경하게 된다. 이후에 설명)</i>
2. cpu, memory 설정 확인하고
3. cicd-xx 구동
4. 머신에 guru 로그인 한다
5. root로 hostname 변경
6. vi /etc/hostname 하고 cicd-xx로 변경 
7. GUI모드로 변경
<code>systemctl isolate graphical.target</code>
8. GUI에서 네트워크 변경<br/> [링크](https://proj.pe.kr/04dd6f45c25748538ef2e0a6e8902e50/35086cc805e948958288e086bab178ab.html) <-- 링크 초반부분 참고하여 네트워크 변경
9. 터미널모드로 변경 <code>systemctl isolate multi-user.target</code>
10. 스냅샷 저장<br>
외부 SSH 접속되는지 보고, 문제없다면 뜨자.

### 3) 모든 노드의 복제를 끝냈으나
#### 💥 모든 노드의 복제가 끝난 뒤, IP를 NAT Network 방식에서 Bridge 방식으로 변경이슈 발생
```plaintext
노트북 2대에 VM을 나눠서 설치하다보니
노트북 A의 서브 네트워크대역대와
노트북 B의 서브 네트워크대역대에 대해서
서로 긴밀한 연결이 어려운 문제가 발생했다.

┌── 노트북A 10.10.5.123
│   ├── cicd-m1 10.100.0.101
│   └── cicd-r1 10.100.0.121
│
└── 노트북B 10.10.5.124
    ├── cicd-w1 10.100.0.111
    ├── cicd-w2 10.100.0.112
    └── cicd-w3 10.100.0.113

NatNetwork의 portforwarding으로는 한계가 생겼다.
예1) 각각의 Node에서 다른 Node로 ssh가 열려있어야 하는데, 서브네트워크끼리 어떻게 연결 시킬 것인가?
     cicd-m1에서 cicd-w2로 ssh 연결, cicd-w1에서 cicd-r1로 ssh 연결
예2) kubernetes에서 master와 worker3대를 어떻게 네트워크 연결 할 것인가?
     k8s에서 사용하는 기본 6443 port
     kubelet이 사용하는 port
     CNI에서 사용하는 port
     kubesphere에서 사용하는 port
     등등

이슈를 해결하기 위해선 VM 5대가 모두 같은 네트워크 대역대여야만 한다.
그런데 노트북엔 VM 5대를 동시에 설치할 수가 없다. (사양딸림)

VirtualBox에서 제공하는 "어댑터 브리지"를 이용하기로 한다.
참고 URL : https://hahaite.tistory.com/322

┌── 노트북A 10.10.5.123
├── cicd-m1 10.10.5.121
├── cicd-r1 10.10.5.136
│
├── 노트북B 10.10.5.124
├── cicd-w1 10.10.5.132
├── cicd-w2 10.10.5.153
└── cicd-w3 10.10.5.139

IP  10.10.5.XXX
SN  255.255.255.0
GW  10.10.5.1
DNS 219.250.36.130
    168.126.63.1

이와 같이 모든 노드들에 대해 같은 네트워크로 구성하였다.
```
#### 노드마다 hosts 변경
```bash
# vi /etc/hosts
10.10.5.121 cicd-m1
10.10.5.136 cicd-r1
10.10.5.132 cicd-w1 
10.10.5.153 cicd-w2 
10.10.5.139 cicd-w3
```

### 4) ssh-keygen (각 노드마다 root로)
#### ○ ssh key를 등록
```bash
ssh-keygen -t rsa
touch ~/.ssh/authorized_keys
chmod 755 ~/.ssh/authorized_keys

# 각 서버의 공개키를 authorized_keys 에 Add한다
```
#### ○ alias 설정
```bash
# .profile 수정
alias m1='ssh -i ~/.ssh/id_rsa cicd-m1'
alias r1='ssh -i ~/.ssh/id_rsa cicd-r1'
alias w1='ssh -i ~/.ssh/id_rsa cicd-w1'
alias w2='ssh -i ~/.ssh/id_rsa cicd-w2'
alias w3='ssh -i ~/.ssh/id_rsa cicd-w3'
```

#### ○ ssh 접속 테스트
> m1, r1, w1, w2, w3 예약어로 패스워드 없이 모든 서버에 SSH 접속되는지 확인

### 5) 최종 노드 현황
> 현재까지 버전을 노드마다 스냅샷 저장한다

구분|OS|Hostname|IP
---|---|---|---
노트북A-VM|Ubuntu 20.04 LTS|cicd-m1|10.10.5.121
노트북A-VM|Ubuntu 20.04 LTS|cicd-r1|10.10.5.136
노트북B-VM|Ubuntu 20.04 LTS|cicd-w1|10.10.5.132
노트북B-VM|Ubuntu 20.04 LTS|cicd-w2|10.10.5.153
노트북B-VM|Ubuntu 20.04 LTS|cicd-w3|10.10.5.139


## 3. kubekey & Kubesphere (root 진행)
### 1) 다운로드 kubekey  
```bash
mkdir kubekey
cd kubekey

curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.13 sh -
chmod +x kk
```

### 2) kk 가능버전 확인
```bash
./kk version --show-supported-k8s
~~
~~
v1.26.0
v1.26.1
v1.26.2
v1.26.3
v1.26.4
v1.26.5
v1.27.0
v1.27.1
v1.27.2
~~
~~
```

> kubekey로 kubernetes와 kubesphere를 동시에 설치할 수 있다. <br/>
> 아래 버전으로 설치 <br/>
> + kubekey v3.0.13 <br/>
> + kubernetes v1.26.5 <br/>
> + kubesphere v3.4.1

### 3) create config 파일 생성
```bash
./kk create config --with-kubernetes v1.26.5 --with-kubesphere v3.4.1
mv config-sample.yaml config-install.yaml
```

### 4) create config 파일 수정
vi config-install.yaml
```bash
metadata:
  name: cicd-poc
spec:
  hosts:
  - {name: cicd-m1, address: 10.10.5.121, privateKeyPath: "~/.ssh/id_rsa"}
  - {name: cicd-w1, address: 10.10.5.132, privateKeyPath: "~/.ssh/id_rsa"}
  - {name: cicd-w2, address: 10.10.5.153, privateKeyPath: "~/.ssh/id_rsa"}
  - {name: cicd-w3, address: 10.10.5.139, privateKeyPath: "~/.ssh/id_rsa"}
  roleGroups:
    etcd:
    - cicd-m1
    control-plane:
    - cicd-m1
    worker:
    - cicd-w1
    - cicd-w2
    - cicd-w3
```

### 5) Kubesphere 설치
#### ○ 설치 ㄱㄱ
```bash
./kk create cluster -f config-install.yaml 
```


#### ○ 에러발생
```log
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
ERRO[0000] unable to determine image API version: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial unix /var/run/dockershim.sock: connect: no such file or directory" 
E1209 10:23:16.239911    1684 remote_image.go:218] "PullImage from image service failed" err="rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.ImageService" image="kubesphere/pause:3.8"
FATA[0000] pulling image: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.ImageService
```
> k8s 1.24 이후부턴 docker가 containerd로 변경되는데,<br/>
> kubekey쪽에서 버그가 있는지, endpoint에 docker가 남아있어서 발생한 문제

#### ○ 에러수정
```bash
# 단순히 프로세스만 조회해도 같은 에러가 발생할 것이다.
crictl ps

# containerd의 toml을 새로 교체해준다.
root@cicd-m1:/etc/containerd$ mv config.toml config.20241209
root@cicd-m1:/etc/containerd$ containerd config default > /etc/containerd/config.toml
root@cicd-m1:/etc/containerd$ systemctl restart containerd
```
```bash
# containerd의 runtime endpoint와 image endpoint를 수동으로 지정하여 확인(일회성)
crictl -r unix:///run/containerd/containerd.sock -i unix:///run/containerd/containerd.sock ps

# 지속적용을 위해 crictl config 명령어를 사용하여 runtime endpoint와 image endpoint를 설정
crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock --set image-endpoint=unix:///run/containerd/containerd.sock

# 위 명령어를 실행하고 나면 /etc/crictl.yaml 파일이 생성되며 내용은 아래와 같음
> runtime-endpoint: "unix:///run/containerd/containerd.sock"
> image-endpoint: "unix:///run/containerd/containerd.sock"
> timeout: 0
> debug: false
> pull-image-on-create: false
> disable-pull-on-run: false

# containerd restart
systemctl restart containerd 후

# 프로세스 조회시 에러없어졌는지 확인
crictl ps
```

#### ○ 재설치 ㄱㄱ
```bash
./kk create cluster -f config-install.yaml 
```
- k8s가 먼저 설치되고, 이후에 Kubesphere가 설치된다.<br/>
  원래는 15분~20분 사이에 설치완료 되는데, 여기선 좀 더 오래걸림..50분?<br/>
  - VM 이슈가 좀 있는듯 하다. <br/>
    네트워크 bridge방식에 문제가 있는건지, 윈도우11과 virtualbox가 안맞는건지 <br/>
    자꾸 노드들의 CPU가 비 정기적으로 Stuck 된다.<br/>
- 여하튼 설치 완료되면 아래와 같은 최종 로그가 보인다.
  ```plaintext
  #####################################################
  ###              Welcome to KubeSphere!           ###
  #####################################################

  Console: http://10.10.5.121:30880
  Account: admin
  Password: P@88w0rd
  NOTES：
    1. After you log into the console, please check the
      monitoring status of service components in
      "Cluster Management". If any service is not
      ready, please wait patiently until all components 
      are up and running.
    2. Please change the default password after login.

  #####################################################
  https://kubesphere.io             2024-12-09 11:54:47
  #####################################################
  11:54:52 KST success: [cicd-m1]
  11:54:52 KST Pipeline[CreateClusterPipeline] execute successfully
  Installation is complete.

  Please check the result using the command:

    kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
  ```

#### ○ 설치 후
```bash
# kubectl get nodes 되는지 확인
NAME      STATUS   ROLES           AGE   VERSION
cicd-m1   Ready    control-plane   15d   v1.26.5
cicd-w1   Ready    worker          15d   v1.26.5
cicd-w2   Ready    worker          15d   v1.26.5
cicd-w3   Ready    worker          15d   v1.26.5

# 다른 유저에게 kubectl을 주고 싶을 경우
mkdir -p ~/.kube
sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

#kubectl 자동 완성 활성화
source <(kubectl completion bash) 
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### ○ Kubesphere Admin 접속
> `10.10.5.121:30880` <br/>
> nginx proxy_pass를 이용하여 https://k8s.proj.pe.kr 로 접속하게 처리하였다.<br/>
> nginx 관련사항은 아래 항목에서 다시 설명한다.

#### ○ Api 서버 접근을 위해 ks-api-np service를 Nodeport로 오픈
```yaml
# http://10.10.5.121:30890 로 API URI를 획득한다
kind: Service
apiVersion: v1
metadata:
  name: ks-api-np
  namespace: kubesphere-system
  labels:
    app: ks-apiserver
    app.kubernetes.io/managed-by: Helm
    tier: backend
    version: v3.3.2
  annotations:
    kubernetes.io/created-by: kubesphere.io/ks-apiserver
    kubesphere.io/creator: admin
    meta.helm.sh/release-name: ks-core
    meta.helm.sh/release-namespace: kubesphere-system
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9090
      nodePort: 30890
  selector:
    app: ks-apiserver
    tier: backend
  type: NodePort
```

#### ● kubesphere admin page에서 pod접속 터미널이 안열리는 경우
`Could not connect to the container. Do you have sufficient privileges?`

nginx forwarding 시에 웹소켓 관련한 설정을 추가 해줘야 한다.(nginx의 설정파일)
```bash
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
server {
       listen 443 ssl;
       server_tokens off;
       server_name k8s.proj.pe.kr;
       ssl_certificate      /appdata/cert/k8s/fullchain.pem;
       ssl_certificate_key  /appdata/cert/k8s/privkey.pem;
       ssl_protocols TLSv1.1 TLSv1.2;

       location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_pass http://10.10.5.121:30880;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
       }
}
```


## 4. nginx 설치
### cicd-~~r1~~에 nginx 설치
#### ○  현재 구성하려는 클러스터 내에 몇몇 도메인을 사용할 예정
  - k8s.proj.pe.kr (Kubesphere Admin)
  - cr.proj.pe.kr (Harbor)
  - gitlab.proj.pe.kr 
  - argocd.proj.pe.kr 
  - home.proj.pe.kr (CICD API page)
#### ○ LB가 없는 채로 구성하기때문에, LB역할을 대신 해줄 gateway가 필요.
- 간단하게 nginx를 대용품으로 사용하기로 한다.
#### ○ ~~cicd-r1에 nginx를 apt 설치~~ <strong>cicd-m1에 nginx를 apt 설치</strong>
  > 나중에 알게 된 사실인데<br/>cicd-r1에 Harbor를 설치하니 Harbor에서 사용하는 nginx가 자동으로 설치되면서,<br/>
  > 기존에 cicd-r1에 설치했던 nginx와 충돌을 일으킴<br/>
  > cicd-m1으로 gateway를 옮기게 되었다.

### nginx conf file 예시 (proxy 관련)
```bash
# 이하 구문은 최종세팅 이후의 conf 파일이다. conf 수정후엔 nginx를 재기동해야 적용
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
server {
        return 301 https://$host$request_uri;
        listen 80;
        server_name k8s.proj.pe.kr gitlab.proj.pe.kr home.proj.pe.kr argocd.proj.pe.kr;
        return 404;
}
server {
       listen 443 ssl;
       server_tokens off;
       server_name k8s.proj.pe.kr;
       ssl_certificate      /appdata/cert/k8s/fullchain.pem;
       ssl_certificate_key  /appdata/cert/k8s/privkey.pem;
       ssl_protocols TLSv1.1 TLSv1.2;

       location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_pass http://10.10.5.121:30880;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
       }
}
server {
       listen 443 ssl;
       server_tokens off;
       server_name gitlab.proj.pe.kr;
       ssl_certificate      /appdata/cert/gitlab/fullchain.pem;
       ssl_certificate_key  /appdata/cert/gitlab/privkey.pem;
       ssl_protocols TLSv1.1 TLSv1.2;

       location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            #proxy_pass http://10.10.5.136:1980;
            proxy_pass https://10.10.5.136:19443;
       }
}
server {
       listen 443 ssl;
       server_tokens off;
       server_name home.proj.pe.kr;
       ssl_certificate      /appdata/cert/home/fullchain.pem;
       ssl_certificate_key  /appdata/cert/home/privkey.pem;
       ssl_protocols TLSv1.1 TLSv1.2;
       root /appdata/www/home;
}
server {
       listen 443 ssl;
       server_tokens off;
       server_name argocd.proj.pe.kr;
       ssl_certificate      /appdata/cert/argocd/fullchain.pem;
       ssl_certificate_key  /appdata/cert/argocd/privkey.pem;
       ssl_protocols TLSv1.1 TLSv1.2;

       location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_pass https://10.10.5.121:31443;
       }
}
```


## 5. NFS관련 설정
### 1) NFS Server 설치
#### ○ cicd-r1 에서 진행 (root)
```bash
# NFS 서버 설치
apt update
apt install nfs-common nfs-kernel-server portmap

mkdir -p /appdata/nfsdisk
```

```bash
# vi /etc/exports 해서 아래 내용 추가기입 저장
  # for kubernetes
  /appdata/nfsdisk *(rw,sync,root_squash)
```
```
exportfs
  /sharedir/k8s     <world>
```

#### 확인
```bash
# 설정 적용을 위해 서비스 재기동
service nfs-server restart

# 재시작에도 항상 자동시작을 위해 enable
systemctl enable nfs-server

# 서버 상태 확인
systemctl status nfs-server.service

# mount 목록을 확인하여 정상 반영되었는지 체크
showmount -e 127.0.0.1

# 최종적으로 공유폴더를 /mnt 경로에 마운트.
mount -t nfs 10.10.5.136:/appdata/nfsdisk /mnt  
```

### 2) NFS Client 설치
#### ○ cicd-m1, cicd-w1, cicd-w2, cicd-w3 각각 설치 (root)
```bash
apt update
apt install nfs-common

# 각 노드에서 정상반영 확인
showmount -e 10.10.5.136

# 마운트 설정
mount -t nfs 10.10.5.136:/appdata/nfsdisk /mnt\

# 마운트 해제
umount /mnt
```

#### ○ ubuntu 부팅시 /etc/fstab에 nfs를 등록하여 자동마운트 처리
vi /et/fstab
```bash
# 아래항목 추가
10.10.5.136:/appdata/nfsdisk        /mnt nfs defaults 0 0
```

#### ○ 수동마운트용 shell
vi mount-nfs.sh
```
#!/bin/sh
showmount -e 10.10.5.136
mount -t nfs 10.10.5.136:/appdata/nfsdisk /mnt
```
chmod +x mount-nfs.sh


## 6. Git-Lab 설치
### 1) 개요
> 여기서는 도커로 설치한다. <br/>
> 🔎 : 구글검색 "gitlab docker 설치" <br/>
> dockerhub의 gitlab/gitlab-ce:13.12.15-ce.0 이미지를 사용할 예정 <br/>
> (현 시점의 gitlab-ce 최신 버전은 17버전인거 같은데, 플젝에서 사용하던 버전에 맞춰서 사용해본다.)

- gitlab의 최소 설치사양의 메모리는 4G이다.
- cicd-r1의 cpu와 메모리를 4 core, 4 Gi로 변경하였다.

### 2) 설치 (cicd-r1에 설치, root로 진행)
#### ○ 데이터 저장 디렉토리 생성
```
mkdir -p /appdata/gitlab/config
mkdir -p /appdata/gitlab/data
mkdir -p /appdata/gitlab/logs
```
#### ○ Docker Run으로 Git-Lab 실행
```
docker run -d --name gitlab --ip 172.17.0.2 --restart=unless-stopped --hostname gitlab.proj.pe.kr \
 -p 1980:80 \
 -p 1922:22 \
 -p 19443:443 \
 -v /appdata/gitlab/config:/etc/gitlab \
 -v /appdata/gitlab/logs:/var/log/gitlab \
 -v /appdata/gitlab/data:/var/opt/gitlab \
 -e TZ=Asia/Seoul \
 gitlab/gitlab-ce:13.12.15-ce.0
```

#### ○ Nginx forwarding
1980은 잘 되는데, 19443은 왜 안되냐.. <br/>
<code>proxy_pass http://10.10.5.136:1980;</code> 로 일단 진행

#### ○ 접속
- https://gitlab.proj.pe.kr 접속 후
- root 패스워드 설정
- 유저생성
  - bluedove / bluedove97@acornsoft.io
  - islandkhj / islandkhj@acornsoft.io

### 3) 남아있는 문제점
- 표면적으로는 https://gitlab.proj.pe.kr 로 동작하고 있으나
- 내부적으로는 http://gitlab.proj.pe.kr 로 동작하다보니
- 내부 Clone 링크라던지, gitlab-runner 설정의 도메인이 http로 되어있는 문제가 있다.
- 추후에 다시 처리




## 7. Docker-compose 설치
### Harbor 설치를 위해 Docker compose를 먼저 설치한다. (root로 진행)
```bash
curl -SL https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose

ls -l /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

```bash
# 버전확인
docker-compose version
```

#### 참고URL : https://seosh817.tistory.com/387



## 8. Harbor 설치
### 1) Harbor에서 사용할 CA Certificates 생성 (cicd-r1, root로 진행)
- 많은 예제는 사설인증서로 Root CA를 발급하는 것부터 설명하나,
- 여기서는 cr.proj.pe.kr을 사용할 예정이다. 
- 이미 아래와 같은 인증파일을 발급한 상태(let's encrypt 에서 발급한 신뢰 인증서)
  - cert.pem
  - chain.pem
  - fullchain.pem
  - privkey.pem
- let's encrypt에서 isrgrootx1.pem(root ca)를 다운받고, 이를 ca.crt로 변환하였다

### 2) ssl pem 인증서를 crt,key로 변환 (cicd-r1, root로 진행)
- /appdata/cert/harbor 에서 진행
- key 파일로 변환하기
  ```bash
  # key 파일로 변환하기
  openssl rsa -in privkey.pem -text > cr.proj.pe.kr.key

  ## crt 파일로 변환하기
  openssl x509 -inform PEM -in fullchain.pem -out cr.proj.pe.kr.crt

  # cert 파일 생성 (도커는 cert파일만 인식한다고 한다)
  openssl x509 -inform PEM -in cr.proj.pe.kr.crt -out cr.proj.pe.kr.cert

  ls /appdata/cert/harbor -l
  합계 40
  -rw-r--r-- 1 root root 1939 12월 11 14:12 ca.crt
  -rw-r--r-- 1 guru guru 1769 12월  6 16:03 cert.pem
  -rw-r--r-- 1 guru guru 1801 12월  6 16:03 chain.pem
  -rw-r--r-- 1 root root 1769 12월 11 10:48 cr.proj.pe.kr.cert
  -rw-r--r-- 1 root root 1769 12월 11 10:41 cr.proj.pe.kr.crt
  -rw-r--r-- 1 root root 5685 12월 11 10:39 cr.proj.pe.kr.key
  -rw-r--r-- 1 guru guru 3570 12월  6 16:04 fullchain.pem
  -rw-r--r-- 1 root root 1939 12월 11 14:18 isrgrootx1.pem
  -rw-r--r-- 1 guru guru 1704 12월  6 16:04 privkey.pem
  ```
- Docker 인증서 복사
  ```bash
  # 나중에 harbor에 'docker login'을 하기 위한 설정
  sudo mkdir -p /etc/docker/certs.d/cr.proj.pe.kr
  cp cr.proj.pe.kr.cert /etc/docker/certs.d/cr.proj.pe.kr/
  cp cr.proj.pe.kr.crt /etc/docker/certs.d/cr.proj.pe.kr/
  cp cr.proj.pe.kr.key /etc/docker/certs.d/cr.proj.pe.kr/
  cp ca.crt /etc/docker/certs.d/cr.proj.pe.kr/
  ```

- 도커 재시작
  ```bash
  systemctl restart docker
  ```

- 📌 참고 : 인증서 파일 내용 조회 
  - <code>openssl x509 -text -noout -in xxx.pem</code>

### 3) Harbor 설치 (cicd-r1, root로 진행)
 - /root home 에서 진행
 - https://github.com/goharbor/harbor/releases/tag/v2.5.3 접속하여 harbor-offline-installer-v2.5.3.tgz 다운로드
 - 압축 해제 <code>tar xzvf harbor-offline-installer-v2.5.3.tgz</code>
 - cd harbor
 - cp harbor.yml.tmpl harbor.yml
 - vi harbor.yml 중 일부 수정
    ```bash
    # 아래 항목에 해당하는 부분 찾아서 수정
    hostname: cr.proj.pe.kr
    https:
      port: 443
      certificate: /appdata/cert/harbor/cr.proj.pe.kr.crt
      private_key: /appdata/cert/harbor/cr.proj.pe.kr.key
    database:
      max_idle_conns: 33
      max_open_conns: 300
    data_volume: /appdata/harbor
    ```
 - Harbor 배포 또는 재구성
   - prepare스크립트를 실행하여 HTTPS를 활성화한다. <br/>
     <code>./prepare</code> <---- 이미지다운, docker-compose.yml 생성도 해주고, 이것저것 한다
   - 혹시 떠있던 harbor는 다운시키고 (docker-compose.yml 이 있는 위치에서 실행) <br/>
     <code>docker compose down -v</code>
   - 하버 기동 (docker-compose.yml 이 있는 위치에서 실행) <br/>
     <code>docker compose up -d</code><br/>

    > 처음 cicd-r1에 설치했던 nginx와 harbor와 같이 설치되는 nginx가 충돌발생<br/>
    > 그래서 cicd-m1에 nginx 설치하는 방식으로 변경한 것이다.<br/>
    > hosts에 cicd-r1 ip인 10.10.5.136은 cr.proj.pe.kr에서 사용

### 3) Harbor Web 관리자 접속
#### ○ https://cr.proj.pe.kr 접속
 - 최초 로그인 정보
   - admin / Harbor12345
   - 이후 비밀번호 변경절차를 수행한다.
#### ○ 프로젝트 샘플 생성
 - 퍼블릭 프로젝트 생성 : public-pj

### 4) Harbor CLI 접속 확인(docker)
#### ○ cicd-r1에서
```bash
# Harbor 로그인 확인
docker login cr.proj.pe.kr

docker pull nginx:1.14

# Harbor Push 확인
docker tag nginx:1.14 cr.proj.pe.kr/public-pj/nginx:1.14
docker push cr.proj.pe.kr/public-pj/nginx:1.14
```

### 5) Harbor CLI 접속 확인(containerd)
 > k8s가 설치되어있는 m1, w1, w2, w3는 docker뿐만 아니라 containerd로 harbor 접속이 원활해야 한다. <br/>
 > <code>crictl pull cr.proj.pe.kr/public-pj/nginx:1.14</code> <br/>
 > 아마 에러가 발생했을 것이다. 노드별 인증처리를 추가로 해줘야 겠다.

#### ○ cicd-m1에서
 - ca.crt, cr.proj.pe.kr.cert, cr.proj.pe.kr.crt, cr.proj.pe.kr.key 파일들 복사처리
    ```bash
    mkdir -p /etc/docker/certs.d/cr.proj.pe.kr
    mkdir -p /etc/containerd/certs.d/cr.proj.pe.kr

    # 각각의 디렉토리에
    # ca.crt, cr.proj.pe.kr.cert, cr.proj.pe.kr.crt, cr.proj.pe.kr.key 파일들 복사처리

    ```
 - config.toml 내용 중 일부 수정<br/>
   vi /etc/containerd/config.toml
    ```bash
        [plugins."io.containerd.grpc.v1.cri".registry]
          [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
            [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
              endpoint = ["https://registry-1.docker.io"]
            [plugins."io.containerd.grpc.v1.cri".registry.mirrors."cr.proj.pe.kr"]
              endpoint = ["https://cr.proj.pe.kr"]
          [plugins."io.containerd.grpc.v1.cri".registry.configs]
            [plugins."io.containerd.grpc.v1.cri".registry.configs."cr.proj.pe.kr".auth]
              username = "admin"
              password = "P@88w0rd"
            [plugins."io.containerd.grpc.v1.cri".registry.configs."cr.proj.pe.kr".tls]
              insecure_skip_verify = true
    ```
 - containerd 재기동 <code>systemctl restart containerd.service</code>
 - pull test <br/> <code>crictl pull cr.proj.pe.kr/public-pj/nginx:1.14</code> <br/> 된다!

#### ○ w1, w2, w3도 같은 m1과 같은 처리를 해준다.


## 9. Argo-CD 설치 
### 1) argocd namespace 생성 (cicd-m1, root로 진행)
```bash
kubectl create namespace argocd
```

### 2) install.yaml 파일 다운로드
```bash
mkdir ~/argo
cd ~/argo
#curl https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml > install.yaml
curl -O https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.6/manifests/install.yaml
mv install.yaml argocd.yaml
```

### 3) 2.10.6으로 설치 진행
```bash
kubectl apply -f argocd.yaml -n argocd
```
#### 설치되고 배포되는 현황을 살펴본 뒤 <code>kubectl get all -n argocd</code>
> argocd-server service를 nodeport로 변경하자
```yaml
# argocd-server service 중 일부 수정
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 31080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8080
      nodePort: 31443
  type: NodePort
```
 - <code>http://10.10.5.121:31080</code>
 - <code>https://10.10.5.121:31443</code>

### 4) ArgoCD 초기 비밀번호 확인
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 5) nginx forwarding 처리
```
server {
       listen 443 ssl;
       server_tokens off;
       server_name argocd.proj.pe.kr;
       ssl_certificate      /appdata/cert/argocd/fullchain.pem;
       ssl_certificate_key  /appdata/cert/argocd/privkey.pem;
       ssl_protocols TLSv1.1 TLSv1.2;

       location / {
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_pass https://10.10.5.121:31443;
       }
}
```
#### https://argocd.proj.pe.kr 접속
 - admin 로그인 후 비밀번호 변경

### 6) argocd cli 사용하기
> argocd server 내부 터미널 접속하여 사용한다
```
kubectl exec -it -n argocd deployment/argocd-server -- /bin/bash 
argocd login localhost:8080
argocd account list
```
#### ○ 계정 생성
- argocd namespace에 있는 configmap 인 argocd-cm을 수정한다.
  ```
  data:
    accounts.bluedove: apiKey, login
    accounts.islandkhj: apiKey, login
  ```
- argocd cli에서
  ```
  argocd account list
  argocd account update-password --account 'bluedove' --new-password 'P@88w0rd'
  argocd account update-password --account 'islandkhj' --new-password 'P@88w0rd'
  ```
#### ○ 계정 권한 설정
> argocd namespace에 있는 configmap 인 argocd-rbac-cm을 수정한다.
```
data:
  policy.default: role:readonly	# 기본권한으로 readonly 설정
  policy.csv: |
    p, role:qa,  applications, *, */*, allow    # qa 역할은 applications 에 모든 권한 허용
    p, role:dev, applications, *, */*, allow
    p, role:dev, projects, *, *, allow
    p, role:dev, repositories, *, *, allow
    
    g, islandkhj, role:admin         # islandkhj 에게 admin 역할 부여
    g, bluedove, role:dev             # bluedove 에게 dev 역할 부여
    g, qauser, role:qa               # qauser 에게 qa 역할 부여
```


## 10. Git-Lab SSL 적용
### 1) SSL 미적용시 문제점 발생
 - gitlab Clone url이 http로 나오는 문제
 - argocd repository 와 gitlab repo의 연결 문제
 - gitlab runner 설치시 연결문제

### 2) 해결해보자 (cicd-r1, root로 진행)
#### ○ ssl pem 인증서를 crt,key로 변환
 - /appdata/cert/gitlab 에서 진행 (여기에 gitlab.proj.pe.kr의 인증서파일을 탑재한다)
    ```bash
    cd /appdata/cert/gitlab

    # key file로 변환하기
    openssl rsa -in privkey.pem -text > gitlab.key

    # crt 파일로 변환하기
    openssl x509 -inform PEM -in fullchain.pem -out gitlab.crt

    # 도커 볼륨마운트 /etc/gitlab/ssl 에 인증서 파일 복사
    cp -rp gitlab.crt /appdata/gitlab/config/ssl/
    cp -rp gitlab.key /appdata/gitlab/config/ssl/
    ```
#### ○ 도커 컨테이너 내부에서 작업
```bash
# 도커 컨테이너 내부로 진입
docker exec -it gitlab bash

# vi /etc/gitlab/gitlab.rb 파일 일부 수정
external_url 'https://gitlab.proj.pe.kr'   # <-- 32 line 쯤
nginx['redirect_http_to_https'] = true     # <-- 1326 line쯤
nginx['redirect_http_to_https_port'] = 80

nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.key"


# 도커 컨테이너 내부에서 아래구문 실행
gitlab-ctl reconfigure


# 로그가 쭉쭉 나오면서 처리된다. 
  ~~
  ~~
  Running handlers:
  Running handlers complete
  Chef Infra Client finished, 9/762 resources updated in 01 minutes 29 seconds
  gitlab Reconfigured! 

# 컨테이너에서 exit
```
#### ○ nginx의 conf를 수정 후 nginx 재기동
<code>proxy_pass https://10.10.5.136:19443;</code>

#### ○ 테스트
 - https://gitlab.proj.pe.kr 접속후
 - 내부 url 조사해보면, https로 변경된 것을 확인 할 수 있다.



## 11. Gitlab runner 설치
> Gitlab Runner는 3가지가 있는데 <br/>
> > Shared Runner - Gitlab 관리자가 구성하고 관리하며, 모든 그룹/프로젝트에서 사용할 수 있다.<br/>
> > Group Runner - Group관리자가 구성하고 관리할 수 있으며, 특정그룹/특정프로젝트에서 사용할 수 있다.<br/>
> > Specific Runner - 단일 프로젝트 전용 Runner. 다른 프로젝트와 공유해서는 안되는 요구사항에서 사용<br/>
> 
> S●●P에서는 Shared Runner를 사용했었기에, 여기서도 Shared Runner를 목표로 설치한다.
### 1) Helm으로 설치. (cicd-m1, root로 진행)
#### ○ 준비
```bash
mkdir ~/gitlab
cd ~/gitlab

# 헬름리포 등록
helm repo add gitlab https://charts.gitlab.io

# 리포 버전 찾기
helm search repo -l gitlab/gitlab-runner
~~
~~
gitlab/gitlab-runner	0.30.0       	14.0.0     	GitLab Runner
gitlab/gitlab-runner	0.29.0       	13.12.0    	GitLab Runner
gitlab/gitlab-runner	0.28.0       	13.11.0    	GitLab Runner
gitlab/gitlab-runner	0.27.0       	13.10.0    	GitLab Runner
gitlab/gitlab-runner	0.26.0       	13.9.0     	GitLab Runner
gitlab/gitlab-runner	0.25.0       	13.8.0     	GitLab Runner
~~
~~
helm search repo -l gitlab/gitlab-runner --version 0.29.0
```
> gitlab이 13.12.15 를 쓰고 있으니 0.29.0 차트를 이용하자

#### ○ 헬름 차트 다운
```bash
helm pull gitlab/gitlab-runner --version 0.29.0
tar -xvzf gitlab-runner-0.29.0.tgz
```
- values.yaml을 myvalues.yaml로 복사해서 사용
- vi myvalues.yaml 중 일부 수정
```bash
gitlabUrl: http://gitlab.proj.pe.kr/  #gitlab의 Admin Area > Runners 에 보이는 Register URL을 입력한다.
                                      #여러번 실패 후에, http가 아니라 https로 해야되더라

runnerRegistrationToken: "okxrT~~~~" ##gitlab의 Admin Area > Runners 에서 확인할 수 있는 Token을 입력한다.

rbac:
  create: true
  rules:
    - apiGroups: ["extensions", "apps"]
      resources: ["deployments"]
      verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
    - apiGroups: [""]
      resources: ["pods", "services", "secrets", "pods/exec", "serviceaccounts"]
      verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:16.04"
  tags: "build-image"
  privileged: true
```
#### ○ 폴더구조
<pre>
gitlab
├── gitlab-runner
│   ├── CHANGELOG.md
│   ├── CONTRIBUTING.md
│   ├── Chart.yaml
│   ├── LICENSE
│   ├── Makefile
│   ├── NOTICE
│   ├── README.md
│   ├── templates
│   │   ├── NOTES.txt
│   │   ├── _cache.tpl
│   │   ├── _env_vars.tpl
│   │   ├── _helpers.tpl
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── hpa.yaml
│   │   ├── role-binding.yaml
│   │   ├── role.yaml
│   │   ├── secrets.yaml
│   │   └── service-account.yaml
│   └── values.yaml
├── gitlab-runner-0.29.0.tgz
└── myvalues.yaml
</pre>

### 2) 차트 설치 Try 1
```bash
helm upgrade --install gitlab-runner -n gitlab-runner --create-namespace gitlab/gitlab-runner --version 0.29.0 -f myvalues.yaml

  Release "gitlab-runner" does not exist. Installing it now.
  NAME: gitlab-runner
  LAST DEPLOYED: Tue Dec 17 11:33:32 2024
  NAMESPACE: gitlab-runner
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  NOTES:
  Your GitLab Runner should now be registered against the GitLab instance reachable at: "http://gitlab.proj.pe.kr/"
```
#### ○ 실패
> gitlab-runner가 배포되었으나, POD가 올라오지 않는다.<br/>
> gitlab.proj.pe.kr 도메인의 세팅문제로 보임

- gitlab.proj.pe.kr의 A레코드 주소 변경 (10.10.0.121)
  - gitlab-runner pod가 구동될때, 외부 DNS의 gitlab.proj.pe.kr 의 도메인을 찾더라.
  - Node의 hosts를 수정해뒀기때문에, POD도 같은 hosts를 쓸거라 생각했는데, 아니었다.
  - 하긴 pod 생성시의 timezone도 node를 따라가지 않고 pod 개별적으로 처리했어야 했지
  - pod 내부 hosts 파일을 수정하기보단, gitlab.proj.pe.kr 도메인의 주소를 내부IP로 변경처리
  - (나중에 알게 됨) cr.proj.pe.kr과 argocd.proj.pe.kr의 A레코드도 IP변경해야겠더라
- helm uninstall은 또 왜 안되는것인지.. helm list 했는데 아무것도 등록된게 없..!! ㄴ더ㅐㅑ러디냐ㅓ3ㅓ
  - kubectl delete ns gitlab-runner 로 날렸다. 깔끔하게 지워졌으려나 에혀

### 3) 차트 설치 Try 2
```bash
kubectl create ns gitlab-runner
# helm install -f ./values.yaml <Release 이름> <차트경로>
# helm install -f ./values.yaml --namespace gitlab-runner ci <Chart 경로>
helm install -f ./values.yaml --namespace gitlab-runner ci gitlab/gitlab-runner --version 0.29.0

  NAME: ci
  LAST DEPLOYED: Tue Dec 17 16:51:56 2024
  NAMESPACE: gitlab-runner
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  NOTES:
  Your GitLab Runner should now be registered against the GitLab instance reachable at: "http://gitlab.proj.pe.kr/"
```
#### ○ 실패
 - POD 로그 확인해보니 여전히 에러 발생
    ```
    ERROR: Registering runner... failed                 runner=okxrTg6P status=401 Unauthorized
    PANIC: Failed to register the runner. You may be having network problems.
    ```
 - 실패. namespace 날리고 재시도 <code>kubectl delete ns gitlab-runner</code>


### 4) 차트 설치 Try 3
```bash
kubectl create ns gitlab-runner

# myvalues.yaml에서 아래부분 수정
gitlabUrl: https://gitlab.proj.pe.kr/ # 기존 http://gitlab.proj.pe.kr/ 
requestConcurrency: 1 # 이건 그냥 해봄
locked: false         # shared runner가 기본적으로 locked 되어있어서 뭐가 잘 안됨
runUntagged: true     # "This job is stuck because the project doesn't have any runners online assigned to it" 에러해결을 위해

helm install -f myvalues.yaml --namespace gitlab-runner ci ./gitlab-runner

  NAME: ci
  LAST DEPLOYED: Wed Dec 18 08:17:40 2024
  NAMESPACE: gitlab-runner
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  NOTES:
  Your GitLab Runner should now be registered against the GitLab instance reachable at: "https://gitlab.proj.pe.kr/"
```
#### ○ 성공
 - 파드도 제대로 떴고
 - 파드 로그도 에러가 발생하지 않음
 - Gitlab Admin Area > Overview > Runner에 Shared Runner가 등록된 것이 보인다!

## 14. Gitlab runner 연동 with gitlab-ci.yml 
### 1) java용 dockerfile 예시
```dockerfile
# 소스 빌드를 위한 base image
FROM maven:3.3-jdk-8 as build

# 빌드 구성
WORKDIR /app
COPY pom.xml .

# 소스코드 복사
COPY src/ /app/src/
RUN mvn -f pom.xml clean package -Dmaven.test.skip=true

# Service를 위한 base image
FROM openjdk:8
USER root
WORKDIR /app

RUN mkdir -p /app/work
COPY --from=build /app/target/app.jar /app.jar
#EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app.jar"]
```
### 2) .gitlab-ci.yml 예시
```yml
# gitlab에 java로 프로젝트를 하나 만들고 dockerfile까지 세팅된 상태에서
# .gitlab-ci.yml을 생성한다.
docker-build:
  image: docker:cli
  stage: build
  services:
    - docker:dind
  variables:
    DOCKER_IMAGE_NAME: cr.proj.pe.kr/test-pj/test-java
    DOCKER_IMAGE_TAG: "1.0"
  before_script:
    - docker login -u "admin" -p "P@88w0rd" cr.proj.pe.kr
  script:
    - docker build --no-cache -t "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG" .
    - docker push "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG"
  when: manual
```

```yml
docker-build:
  image: docker:24.0.5
  stage: build
  services:
    - name: docker:24.0.5-dind
      alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2376
    #DOCKER_TLS_CERTDIR: /certs
    DOCKER_TLS_VERIFY: "1"
    DOCKER_CERT_PATH: $DOCKER_TLS_CERTDIR/client
    DOCKER_IMAGE_NAME: cr.proj.pe.kr/test-pj/test-java
    DOCKER_IMAGE_TAG: "1.0"
    CI_DOCKER_CONFIG: "ewoJImF1dGhzIjogewoJCSJjci5wcm9qLnBlLmtyIjogewoJCQkiYXV0aCI6ICJZV1J0YVc0NlYyaG5aRzF6Wkd0aFoyZ3dNQ0U9IgoJCX0KCX0KfQ=="
  before_script:
    - echo "Waiting for Docker to launch..."; sleep 1;
    - echo "run.. CI_DOCKER_CONFIG = $CI_DOCKER_CONFIG"
  #  - docker login -u "admin" -p "P@88w0rd" cr.proj.pe.kr
  script:
    - mkdir ~/.docker
    - echo "$CI_DOCKER_CONFIG" | base64 -d > ~/.docker/config.json
    - cat ~/.docker/config.json
    - docker build --no-cache -t "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG" .
    - docker push "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG"
  when: manual
```
#### ○ 파이프라인 CI 구동시 에러발생
 - Docker client 19.0x 부터 클라이언트에 ca 인증서가 강제되고 있다고 함
 - unable to resolve docker endpoint: open /certs/client/ca.pem: no such file or directory 에러 발생.
 - 그리고 도커인도커에서 도커가 올라올때 인증서를 생성해서 올라오는데 시간이 걸려서 sleep을 주고, ca.pem 등등의 인증서 파일이 올라오는 시간을 벌어준다.
 - 참고 : https://stackoverflow.com/questions/74958598/not-able-to-build-docker-image-in-gitlab-ci/74958689#74958689
 - gitlab-ci.yaml 도 수정하고, config.toml도 수정하라고 한다.
 - gitlab-runner chart 배포시 configmap이 등록되어있는데, 여기를 수정해서 config.toml을 수정할 수 있다.(config.template.toml 부분)
 - 아예 헬름차트를 수정해서 다시 배포해보자
 - namespace 날리고 <code>kubectl delete ns gitlab-runner</code>,
 - myvalues.yaml 수정 후 헬름으로 gitlab-runner 재배포 처리하자
    ```yaml
    # myvalues.yaml 수정
    runners:
      config: |
        [[runners]]
          [runners.kubernetes]
            namespace = "{{.Release.Namespace}}"
            image = "ubuntu:16.04"
          [[runners.kubernetes.volumes.empty_dir]]
            name = "docker-certs"
            mount_path = "/certs/client"
            medium = "Memory"
    ```
### 3) .gitlab-ci.yml 수정
```yml
docker-build:
  image: docker:20.10.16
  stage: build
  services:
    - name: docker:20.10.16-dind
      alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: /certs
    DOCKER_TLS_VERIFY: "1"
    DOCKER_CERT_PATH: $DOCKER_TLS_CERTDIR/client
    DOCKER_IMAGE_NAME: cr.proj.pe.kr/test-pj/test-java
    DOCKER_IMAGE_TAG: "1.0"
    CI_DOCKER_CONFIG: "ewoJImF1dGhzIjogewoJCSJjci5wcm9qLnBlLmtyIjogewoJCQkiYXV0aCI6ICJZV1J0YVc0NlYyaG5aRzF6Wkd0aFoyZ3dNQ0U9IgoJCX0KCX0KfQ=="
  before_script:
    - sleep 1
    - ls -R /certs/client
    - sleep 2
    - ls -R /certs/client
    - sleep 3
    - ls -R /certs/client
    - echo "Waiting for Docker to launch..."
    - until docker info; do sleep 1; done
    - echo "run.. CI_DOCKER_CONFIG = $CI_DOCKER_CONFIG"
  script:
    - mkdir ~/.docker
    - echo "$CI_DOCKER_CONFIG" | base64 -d > ~/.docker/config.json
    - cat ~/.docker/config.json
    - docker build --no-cache -t "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG" .
    - docker push "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG"
  when: manual
```
#### ○ 파이프라인 CI 구동시 에러발생2
 - 도커인도커에서 도커가 올라올때 sleep과 until을 줘서 인증서 파일이 올라올 수 있는 시간을 줬고
 - dockerfile build까진 진행이 되었으나
 - docker push "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG" 에서 에러 발생
    ```
    The push refers to repository [cr.proj.pe.kr/test-pj/test-java]
    Get "https://cr.proj.pe.kr/v2/": x509: certificate signed by unknown authority
    ```

### 4) Troubleshooting
> S●●P에선 sf●●●●m/docker-c●●p-gitrunner:dind 이런식으로 custom된 이미지를 쓴거같다는 생각이 든다. <br/>
> 참고 : https://gitlab.com/gitlab-org/gitlab-runner/-/issues/1842<br/>
> 로컬용 docker:dind 이미지를 만들고, cert 정보가 있을때와 없을때의 push상황을 비교해보자
#### ○ 임시 dockerfile
 - docker:20.10.16-dind 이미지를 From으로 하여
    ```dockerfile
    FROM docker:20.10.16-dind 
    VOLUME /var/lib/docker
    EXPOSE 2375 2376
    #ENTRYPOINT ["dockerd-entrypoint.sh"]
    CMD ["sleep","3600"]
    ```
 - 빌드하고 `docker build -t cr.proj.pe.kr/public-pj/docker-cicd-gitrunner:test .`
 - 실행시키고 `docker run -d --privileged --name dindd cr.proj.pe.kr/public-pj/docker-cicd-gitrunner:test `
 - 컨테이너에 접속해본다 `docker exec -it dindd sh`
 - ~/docker/config.json 만 생성해봤는데, cr.proj.pe.kr에 로그인이 불가
 - /etc/docker/cert.d/cr.proj.pe.kr/ 에 cr.proj.pe.kr.cert, cr.proj.pe.kr.crt, cr.proj.pe.kr.key, ca.crt 파일까지 넣어주니
 - 그제서야 로그인이 가능
 - OK 그렇다면 커스텀된 dind 이미지가 올라올때, 인증서도 같이 처리해야한다.

### 5) Custom docker in docker image 생성
#### ○ 파일 확인
```
root@cicd-r1:~/docker-dind# ls
  ca.crt  cr.proj.pe.kr.cert  cr.proj.pe.kr.crt  cr.proj.pe.kr.key  dockerfile
```

#### ○ cat dockerfile
```dockerfile
FROM docker:20.10.16-dind 
RUN mkdir -p /etc/docker/certs.d/cr.proj.pe.kr
COPY ca.crt  /etc/docker/certs.d/cr.proj.pe.kr/
COPY cr.proj.pe.kr.cert  /etc/docker/certs.d/cr.proj.pe.kr/
COPY cr.proj.pe.kr.crt  /etc/docker/certs.d/cr.proj.pe.kr/
COPY cr.proj.pe.kr.key  /etc/docker/certs.d/cr.proj.pe.kr/

VOLUME /var/lib/docker
EXPOSE 2375 2376

ENTRYPOINT ["dockerd-entrypoint.sh"]
```

#### Image build 및 push
 - `docker build -t cr.proj.pe.kr/public-pj/docker-cicd-gitrunner:dind .`
 - `docker push cr.proj.pe.kr/public-pj/docker-cicd-gitrunner:dind`

#### ○ 파이프라인 CI 구동시 에러발생3
 - 이것만으론 파이프라인 실행시 /etc/docker/cert.d/cr.proj.pe.kr 라는 디렉토리를 찾지 못했다는 에러가 발생
 - 아마 entrypoint가 상위의 dockerd-entrypoint.sh 가 실행되면서
 - 내가 커스텀한 dockerfile에 작성한 mkdir, COPY 등은 실행조차 안된 걸로 추측
 - 그렇다면 Gitlab Adminarea의 Shared Runner의 variable 설정에서 인증서 4종세트를 모두 환경변수화 하여
 - gitlab-ci.yaml에서 강제로 세팅하는 것까지 추가해서 해보겠다.

### 6) Git-lab Admin area > Settings > CI/CD > Variables > Expand
 > CA_CRT, CR_PROJ_PE_KR_CRT, CR_PROJ_PE_KR_KEY 값을 base64로 encoding 하여 저장

### 7) .gitlab-ci.yml 최종
```yml
docker-build:
  image: docker:20.10.16
  stage: build
  services:
    - name: cr.proj.pe.kr/public-pj/docker-cicd-gitrunner:dind
      alias: docker
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: /certs
    DOCKER_TLS_VERIFY: "1"
    DOCKER_CERT_PATH: $DOCKER_TLS_CERTDIR/client
    DOCKER_IMAGE_NAME: cr.proj.pe.kr/public-pj/test-java
    DOCKER_IMAGE_TAG: "1.0"
    CI_DOCKER_CONFIG: "ewoJImF1dGhzIjogewoJCSJjci5wcm9qLnBlLmtyIjogewoJCQkiYXV0aCI6ICJZV1J0YVc0NlYyaG5aRzF6Wkd0aFoyZ3dNQ0U9IgoJCX0KCX0KfQ=="
  before_script:
    - sleep 10
    - ls -R /certs/client
    - echo "Waiting for Docker to launch..."
    - until docker info; do sleep 1; done
    - echo "run.. CI_DOCKER_CONFIG = $CI_DOCKER_CONFIG"
    - mkdir ~/.docker
    - echo "$CI_DOCKER_CONFIG" | base64 -d > ~/.docker/config.json
    - cat ~/.docker/config.json
    - mkdir -p /etc/docker/certs.d/cr.proj.pe.kr
    - echo "$CA_CRT" | base64 -d > /etc/docker/certs.d/cr.proj.pe.kr/ca.crt
    - echo "$CR_PROJ_PE_KR_CRT" | base64 -d > /etc/docker/certs.d/cr.proj.pe.kr/cr.proj.pe.kr.cert
    - echo "$CR_PROJ_PE_KR_CRT" | base64 -d > /etc/docker/certs.d/cr.proj.pe.kr/cr.proj.pe.kr.crt
    - echo "$CR_PROJ_PE_KR_KEY" | base64 -d > /etc/docker/certs.d/cr.proj.pe.kr/cr.proj.pe.kr.key
    - ls -R /etc/docker/certs.d/cr.proj.pe.kr
  script:
    - docker build --no-cache -t "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG" .
    - echo "run.. docker images"
    - docker images
    - echo "run.. docker push $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_TAG"
    - docker push "$DOCKER_IMAGE_NAME":"$DOCKER_IMAGE_TAG"
  when: manual
```

#### ○ 파이프라인 CI 성공!!
 - gitlab-runner 의 job이 뜨고
 - docker in docker 컨테이너가 정상적으로 구동되고
 - dockerfile을 기반으로 이미지 빌드를 하고
 - 최종적으로 Harbor에 Push까지 되었다.
