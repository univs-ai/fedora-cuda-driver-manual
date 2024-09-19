# Fedora Server configuration manual (Fedora 40 기준)
Fedora 서버 세팅 시 설치 요소와 Nvidia Driver를 인스톨하고, docker와 연결하는 부분을 설명한다.

1. dnf update
2. docker, docker-compose install
3. dnf repository 확장 (rpm fusion 사용)
4. nvidia driver 설치
5. cuda toolkit 설치
6. nvidia docker 설치
7. 방화벽 해제
8. IP 고정
   
### 1. dnf update
Nvidia driver는 최신커널에서 동작하기 때문에, 설치시 update를 한다.
```sh
sudo dnf update
```

### 2. docker, docker-compose install
```sh
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
wget https://github.com/docker/compose/releases/download/v2.29.3/docker-compose-linux-x86_64
sudo mv docker-compose-linux-x86_64 /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
sudo systemctl start docker
sudo systemctl enable docker
```

### 3. dnf update
Nvidia driver는 최신커널에서 동작하기 때문에, 설치시 update를 한다.
```sh
sudo dnf -y install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
sudo dnf -y install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf -y install xorg-x11-drv-nvidia akmod-nvidia xorg-x11-drv-nvidia-cuda
sudo akmods --force
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo yum install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
curl -s -L https://nvidia.github.io/libnvidia-container/centos8/libnvidia-container.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo dnf install nvidia-docker2
sudo nano /etc/nvidia-container-runtime/config.toml
# 반드시 false로 한다.
no-cgroups = false
sudo nmcli con mod  ens9f0 ipv4.address 192.168.79.26/24 \
		ipv4.gateway 192.168.79.1 \
		ipv4.dns 192.168.79.1 \
		ipv4.method manual connection.autoconnect yes
sudo nmcli con mod  ens9f0 ipv4.address 192.168.79.24/16 \
		ipv4.gateway 192.168.0.1 \
		ipv4.dns 192.168.0.1 \
		ipv4.method manual connection.autoconnect yes
sudo docker tag cfcde21dc5df dockerhub.visionlabs.ru/luna/facestream:v.5.2.0
```


- - -
## 예전 설치법 (Nvidia driver 수동 설치)


유니버스플랫폼에서 GPU 구동을 위해 nvidia 드라이버를 설치해야 하는데, 정확하게 정리된 글이 없어서 여기에 남긴다.
방법은 두 가지인데, 
 1. 호스트에 설치 + 도커 컨테이너 툴킷 설치 
 2. 도커 컨테이너 툴킷만 설치


### Fedora 호스트에 Nvidia 드라이버 직접 설치 방법

#### 드라이버 다운로드
전문가용 nvidia 드라이버 검색 사이트에서 일치하는 드라이버를 검색한다. [https://www.nvidia.com/Download/Find.aspx?lang=en-us]

검색 요령은 다음을 참고한다.
 - beta는 패스
 - 유니버스는 드라이버 버전 470, 쿠다버전 11.4만 동작함
 - 보통의 경우는 x86_64로 설치
 - 당연히 리눅스에서 쓰기때문에 리눅스용 드라이버


####
download 버튼에 우클릭을 하여 링크주소 복사를 하여 다운로드 주소를 주워온다.

설치할 유니버스 서버에서 해당명령을 실행한다.
```sh
# 470.239.06 (2024.04월에 릴리즈되어 여러번 실행해본 버전)
curl -o nvidia-driver.run https://us.download.nvidia.com/XFree86/Linux-x86_64/470.239.06/NVIDIA-Linux-x86_64-470.239.06.run
```

#### Fedora 서버 장치 업데이트
설치에 앞서 몇몇 커널과 장치 업데이트를 최신화 해야한다.
```sh
sudo dnf update
sudo dnf install kernel-devel kernel-headers gcc make dkms acpid libglvnd-glx libglvnd-opengl libglvnd-devel pkgconfig
# 안전하게 리붓한다.
sudo reboot
```

#### 기본 GPU 가상 드라이버 끄기 & Fedora 환경 설정
Fedora Server설치를 권장하지만, workstation 으로 설치해도 가능하다. 
다만 기본으로 동작하는 가상 그래픽 드라이버(nouveau)를 꺼줘야 한다.

Server로 설치해도 아래의 작업은 동일하게 수행하기를 권장한다.

##### 아래의 파일 생성
```sh
sudo nano /etc/modprobe.d/blacklist.conf
```

##### 생성한 파일에 아래의 내용 입력
```sh

blacklist nouveau
options nouveau modeset=0
```

##### GRUB 로더 설정파일 열기
```sh
sudo nano /etc/default/grub
```

##### 'GRUB_CMDLINE_LINUX' 이라는 부분 찾아서 다음과 같이 수정
```sh
GRUB_CMDLINE_LINUX="rhgb quiet rd.driver.blacklist=nouveau"
```

##### GRUB 부트 분기점 생성
```sh
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

##### 기본 그래픽 드라이버 삭제 (UI 있는버전만 해당)
```sh
sudo dnf remove xorg-x11-drv-nouveau
```

##### 신규 커널 적용
```sh
sudo dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
```

##### CLI상태 서비스 활성화, 재부팅 
nvidia-driver 설치를 위해서는 GUI가 아닌, CLI 상태로 진입해야 한다.

```sh
systemctl set-default multi-user.target

sudo reboot
```

재시작 하면 CLI 화면이 나온다.

#### Nvidia 드라이버 설치
아까 nvidia-driver.run을 다운받은곳으로 이동하여 다음을 실행한다.
```sh
sudo bash nvidia-driver.run 
```

##### DKMS 커널 등록이 어쩌구저쩌구 > 예스
##### Nvidia 32bit 드라이버가 어쩌구 저쩌구 > 예스
##### Xorg 오토매틱 빽업이 어쩌구저쩌구 > 예스

대부분에 대해 긍정적으로 답변해주면 설치프로세스가 진행된다.

#### 드라이버 설치 확인
별 에러없이 넘어갔다면 드라이버가 정상적으로 설치된 것이다.
GUI가 보고싶다면 다음과 같이 명령을 실행한 후 재부팅한다.

```sh
systemctl set-default graphical.target

sudo reboot
```

재부팅 후에 화면이 제대로 나오고, 다음의 명령어를 입력하여 드라이버 설치상태를 확인한다.
```sh
[universe@fedora ~]$ nvidia-smi
Tue Dec  5 14:56:57 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.223.02   Driver Version: 470.223.02   CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA RTX A4000    Off  | 00000000:2F:00.0 Off |                  Off |
| 41%   52C    P2    37W / 140W |   2658MiB / 16117MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A    883027      C   /usr/bin/python3                 2656MiB |
+-----------------------------------------------------------------------------+

```

#
#
### FEDORA Nvidia docker container toolkit 설치방법

#### 도커설치
```sh
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

#### nvidia docker2 설치
```sh
curl -s -L https://nvidia.github.io/libnvidia-container/centos8/libnvidia-container.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
sudo dnf install nvidia-docker2
```

#### /etc/nvidia-container-runtime/config.toml 수정
```sh
sudo nano /etc/nvidia-container-runtime/config.toml

# 반드시 false로 한다.
no-cgroups = false
```

#### docker 서비스 등록 reboot 그리고 설치 상태 확인
```sh
sudo systemctl enable docker.service

sudo reboot

sudo systemctl start docker.service

docker run --privileged --gpus all nvidia/cuda:11.4.3-base-ubuntu20.04 nvidia-smi
```

#### nvidia driver 로딩 성공 시
다음과 같이 메모리가 0MiB가 아니라 뭐가 하나 걸려있다.
```sh
[dkant@fedora ~]$ sudo docker run -it --gpus all nvidia/cuda:11.4.3-base-ubuntu20.04 nvidia-smi 
[sudo] dkant 암호: 
Tue Dec  5 06:27:47 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.223.02   Driver Version: 470.223.02   CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA RTX A4000    Off  | 00000000:2F:00.0 Off |                  Off |
| 41%   31C    P8     7W / 140W |   2658MiB / 16117MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```
