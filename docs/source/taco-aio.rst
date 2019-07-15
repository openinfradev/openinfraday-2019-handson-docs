***********************
TACO install - aio node
***********************

ceph용 디스크 확인
==================

ceph에서 사용할 disk준비 확인
mount 되어있지 않은 디스크명을 확인하여 메모한다.

.. code-block:: bash

  $ lsblk
   NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   sda       8:0    0 111.8G  0 disk
   |-sda1    8:1    0     2M  0 part
   |-sda2    8:2    0   1.9G  0 part [SWAP]
   `-sda3    8:3    0 109.9G  0 part /
   sdb       8:16   0 111.8G  0 disk           #마운트 안되어 있으므로 사용 가능
   nvme0n1 259:0    0   3.5T  0 disk


tacoplay 설정
=============

* Tacoplay 받아서 준비하기

.. code-block:: bash

   $ sudo yum install -y git
   $ cd ~
   $ git clone https://github.com/openinfradev/tacoplay.git
   $ sudo yum install -y selinux-policy-targeted bridge-utils epel-release
   $ sudo yum install python-pip -y
   $ sudo pip install --upgrade pip==9.0.3
   $ ./scripts/prepare.sh ~/tacoplay/inventory/sample/armada-manifest.yaml
   $ cd tacoplay/

* 하위 프로젝트들 fetch
  
.. code-block:: bash

   $ ./fetch-sub-projects.sh

* ceph-ansible site.yml 생성

.. code-block:: bash

   $ cp ceph-ansible/site.yml.sample ceph-ansible/site.yml

* extra-vars.yml 수정 

monitor_interface, public_network, cluster_network, ceph_monitors, lvm_molumes 확인 후 적절한 값으로 수정 

.. code-block:: bash

   $ ip a
   ...
   4: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
       link/ether 98:03:9b:2f:3c:b6 brd ff:ff:ff:ff:ff:ff
       inet 147.75.93.OOO/31 brd 255.255.255.255 scope global bond0
          valid_lft forever preferred_lft forever
       inet 10.32.7.1/31 brd 255.255.255.255 scope global bond0:0
          valid_lft forever preferred_lft forever
       ...
 
   $ cd ~/tacoplay/inventory/sample
   $ vi extra-vars.yml
   >
   ... 
   # ceph
   monitor_interface: bond0
   public_network: 147.75.93.0/24      <- bond0 의 IP 주소 대역 입력
   cluster_network: 147.75.93.0/24     <- bond0 의 IP 주소 대역 입력
   ...
 
   osd_objectstore: bluestore
   lvm_volumes:
     - data: /dev/sdb     <- 위에서 확인한 Ceph용 디스크 입력


OS 설정
=======

* 호스트 파일 설정: admin 노드

.. code-block:: bash

   $ sudo vi /etc/hosts
   ## TACO ClusterInfo
   127.0.0.1   taco-aio


TACO 설치
=========

* TACO playbook 실행에 필요한 패키지 설치 : admin 노드

.. code-block:: bash

   # admin 노드에서 실행
   cd ~/tacoplay
   sudo pip install -r ceph-ansible/requirements.txt
   sudo pip install -r kubespray/requirements.txt --upgrade
   sudo pip install -r requirements.txt --upgrade

* Taco 설치

.. code-block:: bash

   $ cd ~/tacoplay
   $ ansible-playbook -b -i inventory/sample/hosts.ini -e @inventory/sample/extra-vars.yml site.yml

ansible-playbook 옵션 설명 
-i : 사용할 inventory 파일 지정
-e : 실행시간에 변수 값 전달


TACO 설치 확인
==============

* Network 설정

br-ex 인터페이스 up 시키고, nat 룰을 추가한다

.. code-block:: bash
   
   $ cd ~/tacoplay
   $ ./scripts/init-network.sh

* Key 생성

차후 생성할 VM에 접속하기 위한 keypair를 생성한다.

.. code-block:: bash

   $ ssh-keygen -t rsa

* Openstack 설치 검증

.. code-block:: bash

   $ cd ~/tacoplay
   $ scripts/taco-test.sh

위의 script를 수행하면 다음과 같은 task들을 수행하여 Openstack이 정상 동작하는지 검증하게 된다.
- (가상) Network 및 Router 생성
- Cirros Image upload
- SecurityGroup 생성
- Keypair Import
- VM 생성 후 floating IP 추가
- Volume 생성 후 VM에 추가


VM 생성 후
==========

* 생성된 VM 확인하기

다음과 같은 명령어를 통해 taco-test 스크립트를 돌려 생성된 VM을 확인할 수 있다. 결과 Networks 란에서 생성된 VM 의 ip 주소를 확인한다.

.. code-block:: bash

   $ openstack server list
 
   > 결과
   +--------------------------------------+------+--------+------------------------------------+--------------+---------+
   | ID                                   | Name | Status | Networks                           | Image        | Flavor  |
   +--------------------------------------+------+--------+------------------------------------+--------------+---------+
   | 4dd41f3c-f230-4100-aaaf-3c58cc942463 | test | ACTIVE | private-net=172.30.1.7, 10.10.10.3 | Cirros-0.4.0 | m1.tiny |
   +--------------------------------------+------+--------+------------------------------------+--------------+---------+

* 생성된 VM에 접속, 외부 통신 확인

ssh로 VM 에 접속 후, 네트워크 접속 상태를 확인하기 위해 ping 테스트를 수행한다. ( 8.8.8.8 은 구글 퍼블릭 DNS ip주소)

.. code-block:: bash

   [root@taco-aio ~]# ssh cirros@10.10.10.3    #생성된 VM의 ip주소를 넣는다.
   $ ping 8.8.8.8
   PING 8.8.8.8 (8.8.8.8): 56 data bytes
   64 bytes from 8.8.8.8: seq=0 ttl=53 time=1.638 ms
   64 bytes from 8.8.8.8: seq=1 ttl=53 time=1.498 ms
   64 bytes from 8.8.8.8: seq=2 ttl=53 time=1.147 ms
   64 bytes from 8.8.8.8: seq=3 ttl=53 time=1.135 ms
   64 bytes from 8.8.8.8: seq=4 ttl=53 time=1.237 ms


Trouble Shoothing
=================

* Missing value auth-url required for auth plugin password

.. code-block:: bash

   $ . tacoplay/scripts/adminrc



