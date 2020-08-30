---
title: "CPU 격리 테스트"
date: 2020-08-21 08:26:28 +0800
categories: ["kubernetes"]
---

# CPU 격리 검증

## 검증 배경

고 부하 환경에서 특정 노드 내 POD간 경합으로
 POD에 할당된 값의  CPU자원 사용 가능여부 확인

## 검증 방안

- 동일노드에 2개의 POD 생성(loadtest1, loadtest2)
- 각 POD의 CPU는 0.5vcpu
- 1번 POD(loadtest1)에서 고 부하 명령어 실행(cat /dev/urandom | md5sum)
- 2번 POD(loadtest1)에서 고 부하 명령어를 점진적(1회, 5회, 10회)으로 증가 하여 부하 수준 증가
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

|실행구분|1회|2회|3회|평균|단독 실행 대비 사용률|
|------|---|---|---|---|---|
|단독실행|50.15|50.55|53.81666667|51.50555556|100.00%|
|부하수준 1|52.76666667|49.53333333|50.08333333|50.79444444|98.62%|
|부하수준 5|48.9|48.38333333|45.15|47.47777778|92.18%|
|부하수준 10|45.63333333|46.75|48.9|47.09444444|91.44%|

검증결과 loadtest2 Pod의 부하에 따라서 Pod1의 CPU사용률에는 영향을 미치는 것으로 확인


# 별첨#1. 회차별 부하
### pod1 단독 부하

1회

    16014 root      20   0    1284      4      0 R 60.0  0.0 673:48.53 cat
    16014 root      20   0    1284      4      0 R 37.5  0.0 673:53.60 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 673:58.70 cat
    16014 root      20   0    1284      4      0 R 50.0  0.0 674:03.77 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 674:08.85 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 674:13.94 cat

2회

    16014 root      20   0    1284      4      0 R 50.0  0.0 674:56.78 cat
    16014 root      20   0    1284      4      0 R 33.3  0.0 675:01.84 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 675:06.95 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 675:12.02 cat
    16014 root      20   0    1284      4      0 R 40.0  0.0 675:17.09 cat
    16014 root      20   0    1284      4      0 R 66.7  0.0 675:22.19 cat

3회

    16014 root      20   0    1284      4      0 R 56.2  0.0 691:46.18 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 691:51.27 cat
    16014 root      20   0    1284      4      0 R 43.8  0.0 691:56.35 cat
    16014 root      20   0    1284      4      0 R 56.2  0.0 692:01.44 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 692:06.52 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 692:11.59 cat

### pod2 1개

1회

    16014 root      20   0    1284      4      0 R 53.3  0.0 678:02.02 cat
    16014 root      20   0    1284      4      0 R 50.0  0.0 678:06.97 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 678:11.94 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 678:16.92 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 678:21.88 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 678:26.83 cat

2회

    16014 root      20   0    1284      4      0 R 53.3  0.0 678:40.18 cat
    16014 root      20   0    1284      4      0 R 66.7  0.0 678:45.15 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 678:50.07 cat
    16014 root      20   0    1284      4      0 R 43.8  0.0 678:55.04 cat
    16014 root      20   0    1284      4      0 R 40.0  0.0 678:59.96 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 679:04.91 cat

3회

    16014 root      20   0    1284      4      0 R 50.0  0.0 689:21.63 cat
    16014 root      20   0    1284      4      0 R 43.8  0.0 689:26.58 cat
    16014 root      20   0    1284      4      0 R 43.8  0.0 689:31.54 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 689:36.53 cat
    16014 root      20   0    1284      4      0 R 56.2  0.0 689:41.48 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 689:46.48 cat

### pod 5개

1회

    16014 root      20   0    1284      4      0 R 46.7  0.0 682:45.10 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 682:50.08 cat
    16014 root      20   0    1284      4      0 R 50.0  0.0 682:55.00 cat
    16014 root      20   0    1284      4      0 R 50.0  0.0 682:59.95 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 683:04.92 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 683:09.88 cat

2회

    16014 root      20   0    1284      4      0 R 40.0  0.0 683:37.29 cat
    16014 root      20   0    1284      4      0 R 56.2  0.0 683:42.32 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 683:47.25 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 683:52.22 cat
    16014 root      20   0    1284      4      0 R 50.0  0.0 683:57.17 cat
    16014 root      20   0    1284      4      0 R 37.5  0.0 684:02.08 cat

3회

    16014 root      20   0    1284      4      0 R 46.7  0.0 688:16.85 cat
    16014 root      20   0    1284      4      0 R 40.0  0.0 688:21.80 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 688:26.76 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 688:31.77 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 688:36.68 cat
    16014 root      20   0    1284      4      0 R 37.5  0.0 688:41.60 cat

### pod 10개

1회

    16014 root      20   0    1284      4      0 R 43.8  0.0 684:35.88 cat
    16014 root      20   0    1284      4      0 R 50.0  0.0 684:40.90 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 684:45.83 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 684:50.78 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 684:55.73 cat
    16014 root      20   0    1284      4      0 R 33.3  0.0 685:00.70 cat

2회

    16014 root      20   0    1284      4      0 R 50.0  0.0 685:47.11 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 685:52.11 cat
    16014 root      20   0    1284      4      0 R 40.0  0.0 685:57.04 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 686:02.02 cat
    16014 root      20   0    1284      4      0 R 43.8  0.0 686:07.00 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 686:11.94 cat

3회

    16014 root      20   0    1284      4      0 R 46.7  0.0 687:30.71 cat
    16014 root      20   0    1284      4      0 R 60.0  0.0 687:35.73 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 687:40.69 cat
    16014 root      20   0    1284      4      0 R 46.7  0.0 687:45.60 cat
    16014 root      20   0    1284      4      0 R 40.0  0.0 687:50.56 cat
    16014 root      20   0    1284      4      0 R 53.3  0.0 687:55.50 cat
