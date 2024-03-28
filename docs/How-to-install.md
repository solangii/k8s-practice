## How to install kubernetes?

> k8s설치 과정을 공유합니다.

### What is k8s cluster?

Kubernetes(k8s) 클러스터는 컨테이너 오케스트레이션 시스템으로, 컨테이너화된 워크로드와 서비스를 관리하는 데 사용됩니다. 구성요소는 아래와 같습니다. 설치하시면서 설치 중인 내용이 어떤 구성요소에 해당하는지 인지하면서 진행하시면 좋을 듯 합니다.

1. Master Node
   - 클러스터의 제어 플레인을 담당합니다.
   - 주요 구성 요소로는 API 서버, 스케줄러, 컨트롤 매니저 등이 있습니다.
   - 클러스터 상태를 감시하고 클러스터의 다른 구성 요소를 관리합니다.
2. Worker Node
   - 컨테이너가 실행되는 노드입니다.
   - Pod(컨테이너의 그룹)를 실행하고 관리합니다.
   - kubelet이라는 에이전트를 실행하여 마스터 노드와 상호 작용합니다.
3. Container Runtime
   - 컨테이너를 실행하는 데 사용되는 소프트웨어입니다. 대표적으로 Docker, containerd, cri-o 등이 있고, 저는 containerd를 사용하고자 합니다.
   - Kubernetes는 이러한 컨테이너 런타임과 상호 작용하여 컨테이너를 실행하고 관리합니다.
4. Network Plugin
   - 클러스터 내의 Pod 간 통신을 담당하는 플러그인입니다. 대표적으로 Flannel, Calico등이 있고, 저는 Flannel을 사용하고자 합니다.
   - 이 플러그인은 Pod에 IP 주소를 할당하고, 클러스터 내의 다른 Pod와 통신할 수 있도록 네트워크를 구성합니다.

---

### Getting Started!

0. **[AWS Lightsail](https://lightsail.aws.amazon.com) 사용하여 인스턴스 생성** (최소 2개, 권장 3개) 

   > 저는 AWS Lightsail을 사용했습니다. 다른 여러 대의 컴퓨터 시스템(호스트)가 있다면 다른 것으로 진행해도 좋습니다.

   - master node용 instance 1개 생성 : Linux / ubuntu22.04 LTS / Dual stack (network) / Memory 2GB이상 필요

   - worker node용 instance 2개 생성 : Linux / ubuntu22.04 LTS / Dual stack (network) / Memory 1GB이상 필요

   - ssh key 다운받고, chmod 400 권한 부여.

      `ssh -i [인증키 PATH] ubuntu@ip` 으로 접속 가능

1. **ContainerD 설치 - 모든 node에 대해 진행**

   - containerd란? container 실행 엔진. 이에 대한 설명과 이해를 돕는 좋은 글 - [참고](https://www.linkedin.com/pulse/containerd는-무엇이고-왜-중요할까-sean-lee/?originalSubdomain=kr)

   1. `k8s.conf` 파일에 overlay와 br_netfilter 두 개의 커널 모듈을 로드해야하는 목록에 추가함.

      - `overlay`모듈은 containerd와 같은 container runtime에서 continer file system을 생성하는데 사용하는 overlayFS 파일시스템을 제공

      - `br_netfilter` 모듈은 linux 브리지 네트워킹 스택에서 iptables를 사용하여 네트워크 필터링을 지원함. 
        이는 Flannel, Calico와 같은 Kubernetes 네트워킹 플러그인에 필요한데, 이 플러그인들이 해당 모듈을 이용해서 네트워크 브리지에서 패킷 필터링 수행하기 때문임.

      ```shell
      cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
      overlay
      br_netfilter
      EOF
      ```

   2. overlay와 br_netfilter 모듈 로드

      - `modprobe` 명령은 지정된 커널 모듈 로드하기 위함. 

      ```shell
      sudo modprobe overlay
      sudo modprobe br_netfilter
      ```

   3. `k8s.conf` 파일에 커널 파라미터를 설정한다. 필요한 sysctl파라미터를 설정하면 재부팅 후에도 값이 유지된다.

      ```shell
      cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1
      EOF
      ```

   4. 재부팅 하지 않고 sysctl 파라미터 적용

      ```shell
      sudo sysctl --system
      ```

   5. https관련 패키지 설치

      ```shell
      sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
      ```

   6. docker의 공식 GPG(GNU Provacy Guard)키 추가

      - GPG(GNU Privacy Guard) 키는 안전한 통신과 디지털 콘텐츠의 진위를 확인하는데 사용되는 암호화 키
      -  Docker 패키지 저장소의 공개 GPG 키를 시스템 키 링에 추가하면서 패키지 관리자가 해당 저장소의 서명된 패키지를 신뢰할 수 있게 됨.

      ```shell
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
      ```

   7. Docker를 stable버전으로 설치

      ```shell
      echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
      ```

   8. containerd.io 설치 및 설정 

      ```shell
      sudo apt-get update && sudo apt-get install -y containerd.io
      ```

      - containerd를 실행하기 전 기본 설정을 정의하기 위해, `continaerd config default`를 수행한 다음, `tee`명령을 사용해서 앞의 명령의 출력을 파일에 쓰는 `config.toml` 에 쓰는 작업.

      ```shell 
      sudo mkdir -p /etc/containerd
      containerd config default | sudo tee /etc/containerd/config.toml
      ```

   9. systemd를 cgroup driver로 사용하기위한 설정

      ```shell
      sudo vi /etc/containerd/config.toml
      
      # 125번 줄 false -> true
      SystemdCgroup = true # SystemdCgroup : Systemd를 사용하여 cgroup 계층분리
      ```

   10. containerd 재시작 및 실행 확인

       ```shell
       sudo systemctl restart containerd
       systemctl status containerd
       ```

       active (running)이 뜨면 완료

2. **k8s설치 하기 for Master Node**

   > master node에 대해서만 진행합니다.

   1. 메모리 스왑 비활성

      - **비활성 하는 이유?** 컨테이너는 스스로 메모리를 관리하고 할당하므로, 스왑 공간을 사용하는 것은 일반적으로 권장되지 않는다. 그 외에도 스왑 영역이 비활성화되면 시스템이 메모리를 효율적으로 사용할 수 있게 되어 리소스 효율적 관리도 가능해짐. **k8s cluster에서는 스왑을 사용하지 않는 것이 일반적인 관행.**

      ```shell
      sudo swapoff -a && sudo sed -i '/swap/s/^/#/' /etc/fstab
      ```

   2. k8s APT(Apache Package Tool) 저장소가 필요하는 패키지 설치

      ```shell
      sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
      ```

   3. k8s APT 저장소의 GCP public signing key 다운

      ```shell
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      ```

   4. k8s APT 저장소 등록

      ```shell
      echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
      ```

   5. kublet, kubeadm, kubectl을 설치

      - **kublet**:  k8s 클러스터의 각 노드에서 실행되는 에이전트 (컨테이너화된 애플리케이션을 실행하고 관리함)
      - **kubeadm**: k8s 클러스터를 부트스트랩하고 관리하는 도구. (클러스터를 구성하는 데 필요한 모든 작업을 자동화하여 새로운 k8s 클러스터를 쉽게 설정할 수 있음)
      - **kubectl**: k8s 클러스터와 상호 작용하는 커맨드 라인 인터페이스(CLI) 도구. (k8s 클러스터의 모든 관리 작업을 수행 - 컨테이너화된 어플리케이션 배포, 관리, 리소스 생성, 수정, 삭제, ...)

      ```shell
      sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl && sudo apt-mark hold kubelet kubeadm kubectl
      ```

   6. kubeadmin 초기화 

      **k8s 클러스터 초기화 하는 이유?** 

      - 마스터 노드 설정: 이 명령은 k8s 클러스터의 첫 번째 노드인 마스터 노드를 설정한다. 
      - 네트워크 설정: ``--pod-network-cidr` 플래그는 Pod 네트워크의 CIDR(클러스터 내부의 Pod에 할당되는 IP 주소 범위)를 지정한다. 이 옵션은 클러스터 내의 Pod 간 통신을 위한 IP 주소 범위를 설정한다.
        - CIDR의 경우 K8S에서 주로 사용되는 것은 Calico, Flannel, Canal, Weave 등이 있다. k8s 클러스터 구축 이전에 정해놔야하는 이유 중 하나는 Kubeadm 명령어를 사용하여 k8s를 구축할 때 pod-network-cidr이라는 IP 주소 범위를 설정해야하는데 이 Default 값이 각 CIDR에 따라 다르기 때문이다. 가령 예를 들어 Calico의 경우 192.168.0.0/16 이고 **Flannel의 경우 10.244.0.0/16**이다. 

      ```shell
      sudo kubeadm init --pod-network-cidr=10.244.0.0/16
      ```

      !!!!!해당 명령 이후 출력 된` kubeadm join ~~ --token ~~~~`  토큰 값 저장해두기!!!!!!  - 이후 work node join시 사용됨.

   7. config 및 권한변경

      k8s 클러스터에 대한 로컬 사용자의 config파일 설정 진행

      ```shell
      mkdir .kube
      sudo cp /etc/kubernetes/admin.conf .kube/config
      ls -l .kube/config
      sudo chown ubuntu:ubuntu .kube/config
      export KUBECONFIG=/home/ubuntu/.kube/config
      kubectl get no
      ```

   8. Flannel CNI 설치

      ```shell
      kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
      ```

   9. 설치 확인

      ```shell
      kubectl get no
      kubectl get pods -A -w
      ```

3. **k8s설치 하기 for Worker Node**

   > worker node에 대해서만 진행합니다.

   1. 메모리 스왑 비활성

      ```shell
      sudo swapoff -a && sudo sed -i '/swap/s/^/#/' /etc/fstab
      ```

   2. k8s APT(Apache Package Tool) 저장소가 필요하는 패키지 설치

      ```shell
      sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
      ```

   3. k8s APT 저장소의 GCP public signing key 다운

      ```shell
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      ```

   4. k8s APT 저장소 등록

      ```shell
      echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
      ```

   5. kublet, kubeadm, kubectl을 설치

      ```shell
      sudo apt-get update
      sudo apt-get install -y kubelet kubeadm kubectl
      sudo apt-mark hold kubelet kubeadm kubectl
      ```

   6. Node Join

      master node에 k8s설치 하면서 kubeadm init과정에서 수행한 `sudo kubeadm init --pod-network-cidr=10.244.0.0/16` 이 명령어의 출력을 사용하시면 됩니다. 이때, sudo 권한으로 실행해야합니다. 저의 경우는 아래와 같으며, 이는 사용자마다 token, ip, port값이 달라지므로 ctrl C+V 하면 안되고, 변경된 값을 사용하셔야 합니다.

      ```shell
      sudo kubeadm join 172.26.3.177:6443 --token whjh4d.4wv4nyn0mxvzeaie --discovery-token-ca-cert-hash sha256:f08e49bca0979c3328738f726782a4e85a72ee2a88aacfab8c73584049a09239
      ```

   7. 설치 확인

      ```shell
      kubectl get pods -A -w
      kubectl get no
      ```

      