---
title: "CPU 경합 검증"
date: 2020-08-21 08:26:28 +0800
categories: ["kubernetes"]
---

# Kubernetes의 고부하 환경에서의 CPU 경합 검증

## 검증 배경

Kubernetes에서 특정 노드의 고부하 시
노드에 동작하는 POD의 CPU사용량이 요청한 수량 만큼 할당되는지 검증

## 검증 환경
- 서버 사양:
  - CPU: AMD FX8120(3140MHz, 8Core)
  - 메모리: 8GB
- 시스템 구성:
  - Hypervisor: KVM
  - Kubernetes: 1MasterNode, 2WorkerNode

## 검증 방안

- 워커노드는 1 CPU할당
- 동일노드에 2개의 POD 생성(loadtest1, loadtest2)
- 각 POD의 CPU는 0.5 vcpu를 요청
- 1번 POD(loadtest1)에서 고부하 명령어 실행(cat /dev/urandom > /dev/null)
- 2번 POD(loadtest1)에서 상기 고부하 명령어를 점진적(1회, 5회, 10회)으로 증가 하여 부하 수준 증가
- 2번 POD의 명령어의 증감에 따라 1번 POD의 CPU사용량에 영향을 미치는지 검증

### POD1 사양(Loadtest1)

    [root@master1 k8s]# vi loadtest1.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: loadtest1
      name: loadtest1
    spec:
      containers:
      - args:
        - sleep
        - 9999999999d
        image: busybox
        name: loadtest
        resources:
          limits:
            cpu: "0.5"
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      nodeName: worker01

    status: {}

### POD2 사양(Loadtest2)

    [root@master1 k8s]# cat loadtest2.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: loadtest2
      name: loadtest2
    spec:
      containers:
      - args:
        - sleep
        - 9999999999d
        image: busybox
        name: loadtest
        resources:
          limits:
            cpu: "0.5"
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      nodeName: worker01

    status: {}

### 실행 명령어

    nohup cat /dev/urandom > /dev/null &

### 성능 측정

    # top명령어를 활용하여 CPU사용률 확인(첫번째 수집 값 삭제)
    top -b -n 7 -d 10.0 -p $PID |grep $PID

## 검증 실행

- 상기 POD 사양으로 POD1(Loadtest1), POD2(Loadtest2) 생성
- 단독실행: POD1에서 실행 명령어 실행 및 CPU 사용률 측정
- 부하수준1: POD2에서 실행 명령어 1회 실행 및 CPU 사용률 측정
- 부하수준5: POD2에서 실행 명령어 4회 추가 실행 및 CPU 사용률 측정(총 5개 명령어 실행)
- 부하수준10: POD2에서 실행 명령어 5회 추가 실행 및 CPU 사용률 측정(총 10개 명령어 실행)

## 검증 결과

|부하수준|1회|2회|3회|4회|5회|평균|편차|CPU 사용률(단독실행 대비)|
|------|---|---|---|---|---|---|---|---|
|단독실행|49.98|49.97|49.98|50.00|50.00|49.99|0.014|100.00%|
|부하수준1|48.77|48.77|49.00|48.78|49.03|48.87|0.13|97.77%|
|부하수준5|48.82|48.87|48.73|48.90|48.73|48.81|0.076|97.65%|
|부하수준10|48.82|48.82|48.88|48.70|48.83|48.81|0.07|97.65%|

검증결과 Pod2의 부하에 따라서 Pod1의 CPU사용률에 2%가량 영향을 미치는 것으로 확인

관련 링크: [소스코드](https://github.com/jeonwoosung/k8s-install/tree/master/k8s_test/cpu-isolation/half_core)

## 추가검증

Worker노드에 2CPU할당 후 POD에 1CPU 제한 시 검증 수행

|부하수준|1회|2회|3회|4회|5회|평균|편차|CPU 사용률(단독실행 대비)|
|------|---|---|---|---|---|---|---|---|
|단독실행|99.93|99.92|99.92|99.93|99.9|99.92|0.01|100.00%|
|부하수준1|98.77|98.83|98.43|98.37|98.1|98.50|0.27|98.58%|
|부하수준5|96.82|94.50|95.95|94.70|95.58|95.51|0.85|95.59%|
|부하수준10|92.95|92.55|93.12|93.53|95.1|93.45|0.88|93.52%|

각 POD당 1개 부하 시 약 1%가량 성능 감소 발생, 부하수준이 높아질수록 성능 감소 증가 확인

관련 링크: [소스코드](https://github.com/jeonwoosung/k8s-install/tree/master/k8s_test/cpu-isolation/one_core)
