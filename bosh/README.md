# BOSH 배포

## 1. BOSH?

### 1)  BOSH 소개

*Chef, puppet, ansible과 비슷한 기능을 하는 대규모 분산 환경에서 사용하는 관리, 배포 도구*

BOSH는 어플리케이션를 릴리즈하고, VM을 생성하고 종료 하는 등 다양한 작업을 할 수 있지만. 주로 CF등 PaaS와 연계하여 사용하므로 타 배포 관리 도구와는 차이점이 있으며, CF 이외의 다른 플랫폼에서도 사용할 수 있지만 난이도가 높은 편이다. 기본적으로 다른 도구들과 다르게 OS 명령을 사용하지 않는다.

### 2) 내부용어

- release (https://bosh.io/releases/)

  소프트웨어를 구축하고 배포하는 데 필요한 configuration properties, 스크립트 파일, configuration 템플릿, 소스코드, 바이너리 파일들을 묶어 버전으로 관리

- stemcell (https://bosh.io/stemcells/)

   VM에 배포할 운영체제 이미지. 



## 2. BOSH Cli 설치

- bosh cli 다운로드

  *cli version은 https://github.com/cloudfoundry/bosh-cli/releases 통해 확인*

```
$ wget https://github.com/cloudfoundry/bosh-cli/releases/download/v5.4.0/bosh-cli-5.4.0-linux-amd64
$ sudo chmod +x ./bosh*
$ sudo mv ./bosh* /usr/local/bin/bosh
```

- bosh cli 설치 확인

```
$ bosh -v
version 5.4.0-891ff634-2018-11-14T00:22:02Z

Succeeded
```

- 추가 system package 설치

```
sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt-dev libxml2-dev libssl-dev libreadline6 libreadline6-dev libyaml-dev libsqlite3-dev sqlite3
```



## 3-1. BOSH Lite 설치

### 1) VirtualBox install

*VirtualBox 5.1.38 버전에서 테스트 하여, VirtualBox 5.1.38 버전을 설치하는 방법에 대해 소개하고 있습니다.*

```
$ wget https://download.virtualbox.org/virtualbox/5.1.38/virtualbox-5.1_5.1.38-122592~Ubuntu~xenial_amd64.deb

$ sudo dpkg -i virtualbox*.deb
```

### 2) Director VM

- bosh-deployment 내려받기

```
$ git clone https://github.com/cloudfoundry/bosh-deployment ~/workspace/bosh-deployment
$ mkdir -p ~/deployments/vbox
$ cd ~/deployments/vbox
```

- bosh-deployment 배포하기

```
$ bosh create-env ~/workspace/bosh-deployment/bosh.yml \ 
  --state./state.json \ 
  -o ~/workspace/bosh-deployment/virtualbox/cpi.yml \ 
  -o ~/workspace/bosh-deployment/virtualbox/outbound-network.yml \ 
  -o ~/workspace/bosh-deployment/bosh-lite.yml \ 
  -o ~/workspace/bosh-deployment/bosh-lite-runc.yml \ 
  -o ~/workspace/bosh-deployment/uaa.yml \ 
  -o ~/workspace/bosh-deployment/credhub.yml \ 
  -o ~/workspace/bosh-deployment/jumpbox-user.yml \ 
  --vars-store ./creds.yml \ 
  -v director_name = bosh-lite \ 
  -v internal_ip = 192.168.50.6 \ 
  -v internal_gw = 192.168.50.1 \ 
  -v internal_cidr = 192.168.50.0/24 \ 
  -v outbound_network_name = NatNetwork
```

위 명령은 vm에 자동으로 호스트 전용 모드로 192.168.50.1로 네트워크를 구성해 주며, NAT 네트워크 설정을 통해 외부 통신을 할 수 있도록 'NatNetwork'를 구성해 준다.

-v 옵션은 --var로 bosh.yml에 변수 처리 되어있는 부분의 값을 추가하며, -o 옵션은 --ops-file로 해당 yml의 기능을 manifest에 추가하여 bosh를 설치한다.

위 명령을 통해 cpi, outbound-network, bosh-lite, uaa, credhub와 jumpbox-user를 사용하게 되며, 각 기능에 대해서는 따로 문서로 안내하고 있음.

- Alias 설정

```
$ bosh alias-env vbox -e 192.168.50.6 --ca-cert <(bosh int ~/deployments/vbox/creds.yml --path /director_ssl/ca)
$ export BOSH_CLIENT=admin
$ export BOSH_CLIENT_SECRET= `bosh int ~/deployments/vbox/creds.yml --path /admin_password`
```

alias 설정은 bosh에서 해당 env director를 호출 시 director의 vm 아이피가 아닌 vbox라는 이름으로 쉽게 호출 할 수 있도록 설정하는 단계이며, 배포 완료 후 creds.yml과 state.json이 생성이 된다.

creds.yml은 bosh에서 사용하고 있는 각 컴포넌트의 인증서와 인증 정보를 관리하는 yml파일로 bosh 로그인 및 다양한 컴포넌트의 로그인 정보를 확인하려면 creds.yml를 참고하면 된다.

state.json은 배포하면서 생성한 VM 및 IaaS의 자원 상태를 관리하고 있으며, state.json을 통하여 추후 재사용 및 삭제를 할 수 있다.

- bosh 배포 확인

```
$ bosh -e vbox env

Using environment '192.168.50.6' as '?'

Name      vbox  
...
User      admin  

Succeeded
```

- route 추가

```
$ sudo route add -net 10.244.0.0/16     192.168.50.6 # Mac OS X
$ sudo ip route add   10.244.0.0/16 via 192.168.50.6 # Linux (using iproute2 suite)
$ sudo route add -net 10.244.0.0/16 gw  192.168.50.6 # Linux (using DEPRECATED route command)
$ route add           10.244.0.0/16     192.168.50.6 # Windows
```

호스트 전용 모드로 설치된 bosh와 통신하기 위해서 로컬에 route를 추가해야한다. 

위 순서대로 배포를 하게 되면 VM에 bosh-lite가 설치 된 것을 확인 할 수 있다.

- bosh 삭제

```
$ bosh delete-env ~/workspace/bosh-deployment/bosh.yml \ 
  --state./state.json \ 
  -o ~/workspace/bosh-deployment/virtualbox/cpi.yml \ 
  -o ~/workspace/bosh-deployment/virtualbox/outbound-network.yml \ 
  -o ~/workspace/bosh-deployment/bosh-lite.yml \ 
  -o ~/workspace/bosh-deployment/bosh-lite-runc.yml \ 
  -o ~/workspace/bosh-deployment/uaa.yml \ 
  -o ~/workspace/bosh-deployment/credhub.yml \ 
  -o ~/workspace/bosh-deployment/jumpbox-user.yml \ 
  --vars-store ./creds.yml \ 
  -v director_name = bosh-lite \ 
  -v internal_ip = 192.168.50.6 \ 
  -v internal_gw = 192.168.50.1 \ 
  -v internal_cidr = 192.168.50.0/24 \ 
  -v outbound_network_name = NatNetwork
```

가끔 bosh를 삭제해야할 경우도 생긴다.

해당 명령을 통하여  삭제가 가능하다.

