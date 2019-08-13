# Docker centos for ssh connection
Docker centos ssh connection

## Docker centos 외부 ssh 접속 방법
- 기본적인 세팅으로는 failed to get D-Bus connection: Operation not permitted 에러로 인해 ssh 접속이 안되는 문제가 있음

- 원인은 daemon을 실행할 때 cgroup을 찾지 못하거나 권한문제로 이용할 수 없게 되는 것

- cgroup을 사용할 수 있게 설정한 이미지를 사용하면 해결됨

- 참조 블로그

<https://this-programmer.com/entry/%EB%8F%84%EC%BB%A4Docker%EB%A1%9C-CentOS-%EC%9D%B4%EB%AF%B8%EC%A7%80-systemctl-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-2-failed-to-get-DBus-connection-Operation-not-permitted>

<https://chanhy63.tistory.com/11>

1. Dockerfile 작성
```docker
FROM centos

ENV container docker

RUN yum -y update; yum clean all
RUN yum -y install systemd; yum clean all; \
(cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /lib/systemd/system/local-fs.target.wants/*; \
rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /lib/systemd/system/basic.target.wants/*;\
rm -f /lib/systemd/system/anaconda.target.wants/*;

VOLUME ["/sys/fs/cgroup"]
CMD ["/usr/sbin/init"]
```
2. docker build

```bash
docker build --rm -t centos7-systemd .
```

3. docker run
- network는 macvlan network를 구성하여 외부에서 ip로 접속할 수 있도록  함.
```bash
docker network create -d macvlan --subnet=192.168.100.0/24 --ip-range=192.168.100.32/26 --gateway=192.168.100.1 -o macvlan_mode=bridge -o parent=enp5s0(호스트에서 확인) my_macvlan(네트워크이름)
```
- ip-range 설정과 상관없이 ip가 설정되는 것으로 봐서는 필요없는 것 같기도 함...

- docker 실행 2020 포트는 임의설정, ip또한 남는 ip를 찾음.

```bash
docker run --privileged -itd --name centos_demo -p 2020:22 --ip 192.168.100.32 --network my_macvlan -e container=docker -v /sys/fs/cgroup:/sys/fs/cgroup centos7-systemd /usr/sbin/init
```

4. install ssh and start in container

```bash
docker container exec -it centos_demo /bin/bash

yum -y update
yum -y install net-tools vi openssh-server
systemctl start sshd

```
- root 비밀번호 설정
```
passwd root
```

