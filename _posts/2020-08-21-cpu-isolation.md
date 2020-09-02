---
title: "CPU 격리 테스트"
date: 2020-08-21 08:26:28 +0800
categories: ["kubernetes"]
---

# CPU 격리 검증

## 검증 배경

Kubernetes에서 특정 노드의 고부하 시  
노드에 동작하는 POD의 CPU사용량이 요청한 수량 만큼 할당되는지 검증

## 검증 방안

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

### 검증 명령어

    ps -eo user,pid,ppid,rss,size,vsize,pmem,pcpu,time,cmd |grep urandom |grep -v 'ps -e'

    SET=$(seq 0 5)
    for i in $SET
    do
        echo "Running loop seq "$i
        top -b -n 1 -p 16014 |grep 16014
        sleep 10
    done

## 검증 실행

- Loadtest1, Loadtest2 Pod 생성
- Loadtest1에서 실행 명령어 실행 및 CPU 사용률 측정
- Loadtest2에서 실행 명령어 1개 실행 및 CPU 사용률 측정
- Loadtest2에서 실행 명령어 4개 실행 및 CPU 사용률 측정(총 5개 명령어 실행)
- Loadtest2에서 실행 명령어 5개 실행 및 CPU 사용률 측정(총 10개 명령어 실행)

## 검증 결과

|부하수준|1회|2회|3회|평균|CPU 사용률(단독실행 대비)|
|------|---|---|---|---|---|
|단독실행|49.98|49.97|49.98|49.98|100.00%|
|부하수준1|49.00|48.77|48.77|48.84|97.73%|
|부하수준5|48.73|48.87|48.82|48.81|97.65%|
|부하수준10|48.88|48.82|48.82|48.84|97.72%|

검증결과 loadtest2 Pod의 부하에 따라서 Pod1의 CPU사용률에 2%가량 영향을 미치는 것으로 확인

관련 링크: [소스코드](https://github.com/jeonwoosung/k8s-install/tree/master/k8s_test/cpu-isolation/half_core)
