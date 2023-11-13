# 파드 내 프로세스에서 cgroups 사용하기

## 이번 장에서 다룰 내용
* cgroups 기초
* 쿠버네티스 프로세스 식별
* cgroup의 생성과 관리
* cgroup 계층 구조를 살펴보기 위한 리눅스 명령어
* cgroup v2와 cgroup v1의 차이점
* 프로메테우스 설치와 파드의 리소스 사용 현황 확인

`kubelet`은 노드의 전체 용량을 결정해 계산한 다음 기본 노드와 `kubelet`에 필요한 CPU 대역폭을 차감하고 컨테이너의 할당 가능한 리소스 양에서 이를 뺀다.

    할당 가능 용량 = 노드 용량 - kube-reserved - system-reserved


이걸 알아야 할까..?