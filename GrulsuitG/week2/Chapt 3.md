# Chapt 3. 파드 생성하기

파드가 처음 생성될 때와 실행 상태로 선언되는 시점사이의 간격이 크다.
이 때 일어나는 일에 대해 알아보자!

- kubelet이 컨테이너를 실행해야 한다는 사실을 안다.
- kubele이 Pause 컨테이너를 시작해 리눅스 OS가 컨테이너의 네트워크를 생성할 시간을 제공한다.
  > Pause 컨테이너? 새로운 컨테이너 네트워크 프로세스와 프로세스 ID를 부트스트래핑하기 위한 초기 홈 디렉토리를 만들기 위해 존재한다.
- CNI에서 Pause 컨테이너를 네임스페이스에 바인딩 한다.

이러한 일들이 일어날 때 쿠버네티스는 리눅스 기본 요소들을 활용한다.  
따라서 리눅스 기본 요소를 잘 이해하고 있는 것은 중요하다 !

## 쿠버네티스에서 리눅스 기본 요소 사용하기

다음은 리눅스 기본요소 들이 쿠버네티스에서 어떻게 사용되는지에 대한 설명이다.

- swapoff: 메모리 스와핑 비활서오하
- iptables: 파드로 보내는 iptable 규칙 생성
- mount: 특정 위치에 리소스 연결
- systemd: 핵심 프로세스인 Kublet 실행
- socat: 양방향 정보 스트림 설정
- nsenter: 프로세스의 네임스페이스로 들어가 진행상태 확인
- unshare: 프로세스가 격리되어 실행되는 하위 프로세스를 만들 수 있게 한다.
- ps: 실행중인 프로그램을 나열하는 프로그램

### 파드 생성하기

파드의 생성을 정의하는 yaml 파일을 작성했다면 다음과 같이 간단한 명령어로 파드를 만들 수 있다.

```
kubectl create -f <작성한 yaml 파일>
```

파드를 생성했다면 OS에서 실행중인 파드를 계산하는 명령어로 해당 파드에 대한 프로세스가 생성되었음을 확인할 수 있다.

```
ps -ax | wc -l
```

### 파드의 리눅스 의존성 탐색하기

파드는 다음과 같은 관점에서 일반적인 프로그램과 차이가 없다

- 공유 라이브러리, OS에 특화된 저수준 유틸리티 사용 가능
- 시스템 호출 가능
- 독립적인 메모리 주소 공간 필요

이러한 파드를 만들기 위해 Kubelet 은 다음과 같은 일들을 수행한다.

- 프로그램 실행되기 위한 격리된 홈 생성
- 이더넷 연결 확인
- DNS 확인 및 스토리지 액세스 권한 부여
- 프로그램에게 파드로 이동해서 시작해도 됨을 알려줌
- 프로그램 종료 대기
- 프로그램이 사용한 자원 정리

리눅스 시스템관리자가 해야할 루틴적인 작업들을 kubelet이 대신해준다!
이러한 일련의 과정을 `수명주기`라고 한다.

파드가 실행되고 나면 Pod 객체에 새로운 상태 정보가 있는 것을 알 수 있다.  
이러한 정보를 확인하는 명령어는 다음과 같다.

```
kubectl get pods -o yaml
```

세부 정보를 찾기 위해서는 jsonpath를 이용할 수 있다

```
ex
$ kubectl get pods -o=jsonpath'{.itmes[0].status.phase}'
$ kubectl get pods -o=jsonpath'{.itmes[0].status.podIP}'
$ kubectl get pods -o=jsonpath'{.itmes[0].status.hostIP}'
```

## 처음부터 파드 만들기

만약 쿠버네티스가 없던 시절에 컨테이너 관리 시스템을 만들려고 한다면 어떻게 해야할까?

### 1. chroot를 사용해 격리프로세스 만들기

가장 먼저 해야할 일은 정제된 컨테이너를 만드는 것이다.

- 실행할 프로그램과 프로그램이 실행되야 하는 파일 시스템의 위치를 결정한다.
- 프로세스가 실행될 환경을 만든다.(여러 리눅스 프로그램을 새로운 루트로 불러온다)
- 실행하고 싶은 프로그램을 Chroot 처리된 위치로 복사한다.

### 2. 마운트를 사용해 작업을 위한 프로세스 데이터 제공하기

컨테이너는 다른 곳에 있는 스토리지(ex 클라우드)에 액세스 해야 한다.  
`mount` 명령어를 사용하면 특정 디렉토리를 노출시킬 수 있다.  
이를 통해 chroot 로 생성된 격리 공간에 우리가 원하는 스토리지를 노출할 수 있다.

### 3. unshare를 통한 프로세스 보안

이렇게만 한다면 내부 컨테이너에서 외부 프로세스를 접근해 중단시킬 수 있는 치명적인 문제를 가지고 있다.  
이러한 문제를 예방하기 위해 컨테이너는 호스트에 대해 **격리(Isolation)** 되어야 한다.  
unshare 를 통한다면 프로세스 공간이 완전 격리된 환경에서 실행할 수 있다.  
해당 명령어를 사용하면 PID 가 1부터 시작되어 새로운 환경이 구축된 것처럼 보인다.

### 4. 네트워크 네임스페이스 생성하기

여기까지 한다면 해당 프로세스는 다른 프로세스와 격리되어있지만 아직 같은 네트워크를 사용하는 상태이다.  
새로운 네트워크를 갖는 프로그램을 실행시키려면 역시 unshare 명령어를 사용할 수 있다.

이런 과정들을 통해 컨테이너와 유사한 환경을 만들 수 있다.  
한 가지 차이가 있다면 격리된 네트워크에서 실행된 컨테이너는 cURL이 외부 주소로부터 정보를 가져오는 동작을 하지 않는다.  
왜냐하면 새로운 네임스페이스를 생성할 때 컨테이너에 대한 라우팅과 IP 정보를 잃어버렸기 때문이다.

### cgroup을 통한 CPU 조정하기

쿠버네티스에서 리소스양을 조절하기 위해 `cgroup`을 활용한다.

## 현실에서 파드 사용하기

컨테이너가 클러스터 내의 다른 컨테이너와 통신하기 위해서는 IP 주소가 필요하다.

### 네트워킹 문제

모든 쿠버네티스 컨테이너는 다음 사항이 필요하다.

- 클러스터 내부나 파드에서 파드 사이의 연결으 위해 직접 라우팅된 트래픽
- 또 다른 파드나 인터넷에 액세스하기 위해 라우팅된 트래픽
- 정적 IP 주소를 사용하는 서비스 뒤에서 엔드포인트 역할을 위한 로드 밸런싱된 트래픽

이러한 작업을 위해서는

- 파드의 메타데이터가 필요 -> 쿠버네티스 API 서버 수행
- 상태를 모니터링 -> kubelet 이 수행

이러한 목적을 달성하기 위해 파드는 레이블(label, 상태가 게시되는 방법)과 잘 정의된 명세를 가져야 한다.

### iptables

쿠버네티스 서비스는 '특정 IP -> 엔드포인트 중 서비스가 가능한 하나'로 전달되는 API를 정의한다.  
이러한 네트워킹 규칙은 iptables을 사용하는 kube-proxy를 통해 구현한다.  
iptables는 커널에 규칙을 추가하여 이러한 목적을 달성한다.
