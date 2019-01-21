# BOSH Components

*BOSH 환경을 구성하기 전에 설치, 사용 및 구성할 주요 구성요소에 대한 정보*

![bosh](https://bosh.io/docs/images/bosh-architecture.png) 

*BOSH 아키텍처*

## 1. Command Line Interface (CLI)

Command Line Interface (CLI)는 BOSH에 대한 명령 인터페이스. 운영자는 CLI를 사용하여 Director와 상호 작용하고 Cloud에 대한 작업을 수행한다.

CLI는 Director의 API와 직접 통신할 수 있는 시스템에 설치된다.



## 2. Director

Director는 BOSH의 핵심 구성요소이다. Director는 VM 생성 및 배포 뿐만 아니라 다른 소프트웨어 및 서비스 lifecycle 이벤트를 제어한다.

- ### Task Queue

  Director와 Workers가 작업을 관리하는 데에는 비동기 식 대기열 방식을 사용한다. Task Queue는 데이터 베이스에 저장되어 있다.

- ### Workers

  Task Queue에 등록되어 있는 작업을 실행한다.

- ### CPI

  CPI는 (Cloud Provider Interface)는 Director가 IaaS와 통신하며, stemcells , VM 및 disks를 생성 및 관리하는데 사용되는 API.



## 3. Health Monitor

Health Monitor는 에이전트에서 lifecycle event를 사용하여 VM의 상태를 모니터링 한다. Health Monitor 가 VM 문제를 감지하면, 알림 Plugin을 통하여 알람을 보내거나 Resurrector를 통하여 복구를 시도한다.

- ### Resurrector

  사용하도록 설정된 Resurrector Plugin은 Health Monitor에서 내려가거나 오류가 있는 것으로 식별된 VM을 자동으로 재생성을 시도한다. CLI에서 사용하는 것과 동일한 Director API 를 사용한다.



## 4. DNS Server

​	BOSH는 PowerDNS를 사용하여 배포 된 VM간에 DNS 정보 제공 서비스를 제공한다.



## 5. Director 저장 요소

- ### Database

  Director는 Postgres 데이터베이스를 사용하여 배포 상태에 대한 정보를 저장한다. Database에는 Stemcells, 릴리즈 정보 및 배포 정보가 저장됨.

- ### Blobstore

  Blobstore는 어플리케이션의 소스와 어플리케이션을 컴파일 된 이미지를 저장한다. 사용자는 CLI를 사용하여 어플리케이션을 업로드를 하고 Director는 어플리케이션을 Blobstore에 저장한다. 어플리케이션을 배포할 때 BOSH는 패키지 컴파일을 조정하고 결과를 Blobstore에 저장한다.



## 6. Agent

BOSH에서 배포하는 모든 VM에는 Agent가 포함되어 있다. Agent는 Director으로부터 configure 받아 VM에서 그 configure를 실행시킨다.

예를 들면, MySQL 실행 작업을 VM에 할당하기 위하여 Director는 VM의 Agent에 configure를 지시한다. 이러한 configure에는 설치 패키지 및 이러한 패키지를 구성하는 방법이 포함되어있으며, Agent는 이 configure를 사용하여 VM에 MySQL를 설치하고 구성한다.



## 7. 컴포넌트 간 통신 요소

- ### Message Bus (NATS)

  Director와 Agents는 NATS라는 메시징 시스템을 통해 통신한다. 이 메시지는 VM에 대한 Provisioning 지침을 수행하고 모니터링되는 프로세스의 상태 변화에 대해 Health Monitor에게 알려주는 역할을 한다.

- ### Registry

  Director는 VM을 생성하거나 업데이트할 때 VM의 부트스트래핑을 단계에서 사용 할 수 있도록 VM에 대한 구성 정보를 Registry에 저장한다.

