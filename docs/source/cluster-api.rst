************
Cluster API Hands-on
************

Cluster API 란?
========================

Cluster API는 cloud(or baremetal)에 kubernetes 스타일로 정의된 api로 kubernetes cluster를 생성/설정/관리하는 기능이다.
사용자는 설치에 필요한 몇몇 yaml파일을 clusterctl 명령어를 이용해서 배포하면 target cloud에 auto-healing, auto-managing되는 kubernetes cluster를 손쉽게 생성할 수 있다.

.. figure:: _static/clusterapi-is.png

.. seealso::

   cluster-api-provider-openstack: https://github.com/kubernetes-sigs/cluster-api-provider-openstack
   Go 개발환경: https://golang.org/doc/install
   Kind: https://github.com/kubernetes-sigs/kind

Cluster API Hands-on 구조
========================

.. figure:: _static/taco-clusterapi-diagram.png


Hands-on 후 알게 되는 내용
========================

CRD (Custom Resource Definition)
----------

::

  Custom resources can appear and disappear in a running cluster through dynamic registration, and cluster admins can update custom resources independently of the cluster itself. Once a custom resource is installed, users can create and access its objects using kubectl, just as they do for built-in resources like Pods.



Controller Pattern
----------

::

  In applications of robotics and automation, a control loop is a non-terminating loop that regulates the state of the system. In Kubernetes, a controller is a control loop that watches the shared state of the cluster through the API server and makes changes attempting to move the current state towards the desired state. Examples of controllers that ship with Kubernetes today are the replication controller, endpoints controller, namespace controller, and serviceaccounts controller.


clusterctl 등 tools 빌드
========================

go install
----------

binary file을 받아 go를 설치하고, PATH 및 GOPATH 설정을 한다.

binary file 다운 및 압축 해제
 
.. code-block:: bash

   $ wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
   $ tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz

bashrc에 다음 두 줄을 추가한다.

.. code-block:: yaml

   export PATH=$PATH:/usr/local/go/bin:/root/go/bin
   export GOPATH=$HOME/go

.. code-block:: bash

   $ source ~/.bashrc


yq install
----------

.. code-block:: bash

   $ go get gopkg.in/mikefarah/yq.v2
   $ mv ~/go/bin/yq.v2 /usr/local/bin/yq


clusterctl install
------------------

.. code-block:: bash

   $ git clone -b taco-clusterapi https://github.com/openinfradev/cluster-api-provider-openstack.git $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack
   $ cd $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack/
   $ make clusterctl
   $ ln -s $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack/bin/clusterctl ~/go/bin/clusterctl


bootstraping machine tool 설치 (kind)
-------------------------------------

.. seealso::

   $ cd ~/
   $ GO111MODULE="on" go get sigs.k8s.io/kind@v0.4.0


Openstack Resource 준비
=======================

security group
--------------

openstack client 를 통해서 cluster api가 사용할 openstack security group을 만든다.

.. code-block:: bash

   openstack security group create clusterapi
   openstack security group rule create --ingress --protocol tcp --dst-port 6443 clusterapi
   openstack security group rule create --ingress --protocol tcp --dst-port 22 clusterapi
   openstack security group rule create --ingress --protocol tcp --dst-port 179 clusterapi
   openstack security group rule create --ingress --protocol tcp --dst-port 3000:32767 clusterapi
   openstack security group rule create --ingress --protocol tcp --dst-port 443 clusterapi
   openstack security group rule create --egress clusterapi


CentOS image upload
-------------------

CensOS 이미지를 다운받고, 이를 openstack에 업로드한다.
이 CentOS-7-1905 이미지로 master와 node를 만들 것이다.

.. code-block:: bash

   wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.raw.tar.gz
   tar zxvf CentOS-7-x86_64-GenericCloud.raw.tar.gz
   openstack image create 'CentOS-7-1905' --disk-format raw --file ~/CentOS-7-x86_64-GenericCloud-1905.raw --container-format bare --public


Floating ip 2개 생성
--------------------

master와 node가 사용할 2개의 floating ip 를 미리 생성한다.

.. code-block:: bash

   $ openstack floating ip create public-net
   $ openstack floating ip create public-net


clusterctl 실행 준비
====================

create ~/clouds.yaml
--------------------

clusterctl로 배포할 환경의 정보를 입력한다.

아래의 결과로 얻은 openstack의 admin project ID를 clouds.yaml에 넣어준다.

.. code-block:: bash

   $ openstack project list | grep admin | awk '{print $2}'

.. code-block:: yaml
   :Caption: vi ~/clouds.yaml

   clouds:
     taco-openstack:
       auth:
         auth_url: http://keystone.openstack.svc.cluster.local:80/v3
         project_name: admin
         username: admin
         password: password
         user_domain_name: Default
         project_domain_name: Default
         project_id: <PROJECT_ID>
       region_name: RegionOne


user-data에 hosts 수정 코드 삽입
--------------------------------

master와 node에서 openstack에 접근할 수 있도록 /etc/hosts 파일을 추가한다.

아래의 두 파일을 열어서 YOUR-NODE-IP를 자신의 ip 주소로 바꾼다.

.. code-block:: bash

   $ cd $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack/cmd/clusterctl/examples/openstack
   $ vi provider-component/user-data/centos/templates/master-user-data.sh
   $ vi provider-component/user-data/centos/templates/worker-user-data.sh

.. code-block:: yaml

   #!/bin/bash
   set -e
   set -x
   cat >> /etc/hosts <<EOF
   YOUR-NODE-IP horizon.openstack.svc.cluster.local
   YOUR-NODE-IP keystone.openstack.svc.cluster.local
   YOUR-NODE-IP glance.openstack.svc.cluster.local
   YOUR-NODE-IP nova.openstack.svc.cluster.local
   YOUR-NODE-IP neutron.openstack.svc.cluster.local
   YOUR-NODE-IP cinder.openstack.svc.cluster.local
   EOF


YAML 생성
---------

.. code-block:: bash

   $ cd $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack/cmd/clusterctl/examples/openstack
   $ ./generate-yaml.sh -f ~/clouds.yaml taco-openstack centos
   $ ls out/
   cluster.yaml machines.yaml provider-components.yaml


Openstack keypair 등록
----------------------

vm에 넣을 keypair를 만들고 openstack에 등록한다.

.. code-block:: bash

   $ openstack keypair create --public-key ~/.ssh/openstack_tmp.pub cluster-api-provider-openstack


설정을 위한 openstack 자원조회
------------------------------

.. code-block:: bash

   $ openstack network list | grep private-net | awk '{print $2}'
   $ openstack floating ip list
   $ openstack security group list | grep clusterapi | awk '{print $2}'


구축된 openstack 환경에 맞게 설정, tag 및 serverMeta 등 불필요한 내용 삭제
---------------------------------------------------------------------------

| 아래의 out/machines.yaml을 붙여넣고, 위의 openstack 자원조회 결과를 <PRIVATE-NET-UUID>, <FLOATING-IP>, <SECURITY-GROUP-UUID>에 넣는다.
| 참고: master 와 node는 각각 다른 floating ip를 사용한다.

.. code-block:: yaml
   :Caption: vi out/machines.yaml

   items:
   - apiVersion: "cluster.k8s.io/v1alpha1"
     kind: Machine
     metadata:
       generateName: openstack-master-
       labels:
         set: master
     spec:
       providerSpec:
         value:
           apiVersion: "openstackproviderconfig/v1alpha1"
           kind: "OpenstackProviderSpec"
           flavor: cluster
           image: CentOS-7-1905
           sshUserName: centos
           keyName: cluster-api-provider-openstack
           availabilityZone: nova
           networks:
           - uuid: <PRIVATE-NET-UUID>
           floatingIP: <FLOATING-IP>
           securityGroups:
           - uuid: <SECURITY-GROUP-UUID>
           userDataSecret:
             name: master-user-data
             namespace: openstack-provider-system
           trunk: false
       versions:
         kubelet: 1.14.3
         controlPlane: 1.14.3
   - apiVersion: "cluster.k8s.io/v1alpha1"
     kind: Machine
     metadata:
       generateName: openstack-node-
       labels:
         set: node
     spec:
       providerSpec:
         value:
           apiVersion: "openstackproviderconfig/v1alpha1"
           kind: "OpenstackProviderSpec"
           flavor: cluster
           image: CentOS-7-1905
           sshUserName: centos
           keyName: cluster-api-provider-openstack
           availabilityZone: nova
           networks:
           - uuid: <PRIVATE-NET-UUID>
           floatingIP: <FLOATING-IP>
           securityGroups:
           - uuid: <SECURITY-GROUP-UUID>
           userDataSecret:
             name: worker-user-data
             namespace: openstack-provider-system
           trunk: false
       versions:
         kubelet: 1.14.3


cluster 생성
=============

create k8s cluster on openstack
-------------------------------

.. code-block:: bash

   $ clusterctl create cluster --bootstrap-type kind --provider openstack -c ./out/cluster.yaml -m ./out/machines.yaml -p ./out/provider-components.yaml

KUBECONFIG 설정 후 kind k8s cluster를 확인할 수 있다.

.. code-block:: bash

   $ export KUBECONFIG="$(kind get kubeconfig-path --name="clusterapi")"
   $ kubectl get pods --all-namespaces


생성완료 후 node 조회
---------------------

.. code-block:: bash

   $ kubectl get nodes --kubeconfig kubeconfig


생성과정 debugging
==================

host node에서 kind 내의 clusterapi-controller log 확인
------------------------------------------------------

.. code-block:: bash

   $ export KUBECONFIG="$(kind get kubeconfig-path --name="clusterapi")"
   $ kubectl logs -f clusterapi-controllers-0 -n openstack-provider-system


생성중인 vm에 접속해서 확인
---------------------------

.. code-block:: bash

   $ ssh centos@FLOATING-IP -i ~/.ssh/openstack_tmp
 
   #userdata 확인
   $ sudo cat /var/lib/cloud/instance/user-data.txt
 
   #userdata를 직접 실행해보며 문제를 파악할 수 있음
   $ sudo cd /var/lib/cloud/instance/
   $ sudo bash user-data.txt
 
   #cloud init 실행 확인
   sudo tail -f /var/log/cloud-init.log
 
   #k8s 설치 과정 확인
   sudo tail -f /var/log/messages
