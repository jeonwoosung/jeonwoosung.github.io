---
title: "KVM설치"
date: 2020-08-21 08:26:28 +0800
categories: ["kubernetes", "rancher"]
---

# KVM설치

## 환경

- 서버사양: FX8120(8Core), 8GB
- OS: CentOS7

## kvm설치
아래 명령어를 입력하면 KVM 설치가 완료 됩니다.

    # yum install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install
    # systemctl enable libvirtd
    # systemctl start libvirtd

## 네트워크 구성
가상화 환경에서 Bridge 네트워크를 구성하면, 각 VM에 공유기에서 IP를 할당받아 공유기 네트워크에서 VM간 통신이 가능합니다.
하기의 "enp3s0"은 시스템 별로 상이 합니다.
아래와 같이 수정하면, 해당 네트워크는 bridge로 동작하며, 네트워크 스트립트에 있는 다른 설정이 무시됩니다.

    # vi /etc/sysconfig/network-scripts/enp3s0
    BRIDGE=br0

상기에 기술한 br0에 대한 네트워크 스크립트를 작성합니다.
아래에 입력한 IP가 해당 서버의 IP로 됩니다.

    # vi /etc/sysconfig/network-scripts/ifcfg-br0
    DEVICE="br0"
    BOOTPROTO="static"
    NETMASK=255.255.255.0
    GATEWAY="192.168.0.1"
    IPADDR="192.168.0.100"
    #IPV6INIT="yes"
    #IPV6_AUTOCONF="yes"
    ONBOOT="yes"
    TYPE="Bridge"
    DELAY="0"


## VM생성
VM생성은 아래와 같이 할 수 있습니다.
disk path에 vm이미지가 생성될 경로를, size에 디스크 크기를 입력하고,
location에 게스트os에 설치할 경로를 기입합니다.
아래와 같이 입력하면, 커맨드 모드로 설치가 가능합니다.

    virt-install --name master \
     --memory 1024 \
     --vcpus 1 \
     --disk path=/home/vmimage/master.img,size=20 \
     --os-variant rhel7.0 \
     --location /home/osimage/CentOS-7-x86_64-Minimal-2003.iso \
     --graphics none \
     --extra-args='console=ttyS0'

## VM복제
OS설치가 완료되면, 워커노드로 사용할 VM을 복제 합니다.

     virt-clone --original master --name worker01 --file /data/vmimage/worker01.img
     virt-clone --original master --name worker02 --file /data/vmimage/worker02.img
