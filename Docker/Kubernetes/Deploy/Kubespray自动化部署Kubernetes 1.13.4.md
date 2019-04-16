# Kubespray自动化部署Kubernetes 1.13.4

*Author: jetinzhang <jetinzhang@163.com>*

## 1. 参考资料：
 首先感谢以下文章的作者，才能写出这篇文档 (我不确定以下所有链接是否是原创链接)
- [使用Kubespray自动化部署Kubernetes 1.13.1](https://www.kubernetes.org.cn/5012.html)
- [使用Kubespray 部署kubernetes 高可用集群](https://yq.aliyun.com/articles/505382)
- [使用Kubespray 2.8.3部署生产可用的Kubernetes集群(1.12.5)](https://blog.csdn.net/lilizhou2008/article/details/88286664)
- [kubespray安装过程各节点初始化脚本](https://blog.csdn.net/vah101/article/details/86215350)
- [k8s镜像被墙解决方案gcr.azk8s.cn -- 感谢anjia0532，在他的github中看到了README文件](https://github.com/anjia0532/gcr.io_mirror)
- [感谢微软代理的kubernetes镜像,让国内通过将gcr.io替换为gcr.azk8s.cn即可直接拉取到镜像](https://mirror.azure.cn/help/gcr-proxy-cache.html)

## 2. 安装需具备的知识

    Linux, Ansible,

## 3. 环境说明
| 主机名 | ip地址 | 角色 | 系统 |
| :------:| :------: | :------: | :------: |
| ops-01 | 10.10.4.77 | Kubespray_admin, docker_repo | CentOS7.5.1804 (内核：3.10.0-862.el7.x86_64) |
| master1 | 10.10.4.81| master (etcd) | CentOS7.5.1804 (已升级内核：4.19.27) |
| master2 | 10.10.4.82| master (etcd) | CentOS7.5.1804 (已升级内核：4.19.27) |
| master3 | 10.10.4.83| master (etcd) | CentOS7.5.1804 (已升级内核：4.19.27) |
| node1 | 10.10.4.84| node | CentOS7.5.1804 (已升级内核：4.19.27) |
| node1 | 10.10.4.85| node | CentOS7.5.1804 (已升级内核：4.19.27) |
| node1 | 10.10.4.86| node | CentOS7.5.1804 (已升级内核：4.19.27) |

## 4. docker版本要求:
==The list of validated docker versions remain unchanged at 1.11.1, 1.12.1, 1.13.1, 17.03, 17.06, 17.09, 18.06 since Kubernetes 1.12. ([#68495](https://github.com/kubernetes/kubernetes/pull/68495))==


## 5. 初始化环境 
==注意：所有k8s集群服务器上执行--除了ops-01这一台Kubespray_admin服务器==
### 5.1. 更新内核
    ```
    centos7.5可通过elrepo源升级到4.4
    
    # 配置安装elrepo源
    
    rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
    
    # 升级内核版本
    yum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel
    
    PS: 因内核4.4无法满足我们现有的需求，我通过编译安装了Linux内核4.19.27
    步骤省略（通过Linux官方内核进行编译最新版本）
    ```
### 5.2. 关闭防火墙
    ```
    systemctl stop firewalld
    systemctl disable firewalld
    
    # 临时关闭selinux
    setenforce 0
    
    # 永久关闭selinux (重启生效)  /etc/sysconfig/selinux 是一个软链接文件所以要加上 --follow-symlinks
    sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
    ```
### 5.3. 内核配置
    ```
    # 增加内核配置
    vim /etc/sysctl.conf
    
    # docker
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    
    # 使其内核配置生效
    sysctl -p
    ```
### 5.4. 关闭虚拟内存
    ```
    swapoff -a
    修改/etc/fstab 注释掉有swap的那一行
    ```
### 5.5. 配置yum源仓库
    ```
    # 备份原有yum源

    mkdir /etc/yum.repos.d/backup
    mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/

    # 获取网易163yum源
    wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo

    # 重建缓存
    yum clean all && yum makecache
    ```
### 5.6. 安装必需软件
    ```
    # 安装 centos的epel源
    yum -y install epel-release

    yum install -y yum-utils python-pip python37 python-netaddr python37-pip
    pip install --upgrade pip && pip install netaddr jinja2
    ```
### 5.7. 安装docker

    安装docker版本18.06，此处是下载特定版本进行rpm安装：

    [docker下载地址](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)

    下载文件: [docker-ce-18.06.3.ce-3.el7.x86_64.rpm](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.06.3.ce-3.el7.x86_64.rpm)
    
    安装
    yum localinstall docker-ce-18.06.3.ce-3.el7.x86_64.rpm 
    
    注意：自动化安装后会生产/etc/systemd/system/docker.service文件，如果你安装docker创建了/usr/lib/systemd/system/docker.service文件可能docker启动会有冲突。
    
    自动安装的相关参数配置目录
    ```
    [root@master1 ~]# ll /etc/systemd/system/docker.service.d/
    
    total 8
    -rw-r--r--. 1 root root 212 Mar 21 10:32 docker-dns.conf
    -rw-r--r--. 1 root root 127 Mar 21 10:32 docker-options.conf
    
    [root@master1 ~]# cat /etc/systemd/system/docker.service.d/docker-options.conf   
    [Service]
    Environment="DOCKER_OPTS=  --data-root=/var/lib/docker --log-opt max-size=50m --log-opt max-file=5 --iptables=false"
    
    [root@master1 ~]# cat /etc/systemd/system/docker.service.d/docker-dns.conf        
    [Service]
    Environment="DOCKER_DNS_OPTIONS=\
        --dns 10.10.4.73  \
        --dns-search default.svc.cluster.local --dns-search svc.cluster.local  \
        --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2  \
    
    ```
    
    cat /etc/systemd/system/docker.service
    ```
    [Unit]
    Description=Docker Application Container Engine
    Documentation=http://docs.docker.com
    After=network.target docker-storage-setup.service
    Wants=docker-storage-setup.service

    [Service]
    Type=notify
    Environment=GOTRACEBACK=crash
    ExecReload=/bin/kill -s HUP $MAINPID
    Delegate=yes
    KillMode=process
    ExecStart=/usr/bin/dockerd \
              $DOCKER_OPTS \
              $DOCKER_STORAGE_OPTIONS \
              $DOCKER_NETWORK_OPTIONS \
              $DOCKER_DNS_OPTIONS \
              $INSECURE_REGISTRY
    LimitNOFILE=1048576
    LimitNPROC=1048576
    LimitCORE=infinity
    TimeoutStartSec=1min
    # restart the docker process if it exits prematurely
    Restart=on-failure
    StartLimitBurst=3
    StartLimitInterval=60s

    [Install]
    WantedBy=multi-user.target
    ```
    
    遇到问题：直接docker运行的镜像无法访问外部网络。因为经过了kubespray的配置网络和docker原有的网络不同导致的问题。
    要修改docker.service中的 $DOCKER_NETWORK_OPTIONS
    需要先删除 docker0网卡,然后重启docker,不然只重启docker网络是不会改变的。

    一步执行：
    ip link set dev docker0 down && brctl delbr docker0 && systemctl daemon-reload && systemctl restart docker 
    
    多步执行：
    $ sudo ip link set dev docker0 down
    $ sudo brctl delbr docker0
    $ sudo systemctl restart docker 
    
    增加配置(默认 --mtu=1500需要修改的话可以后面增加参数 --mtu=1450进行修改)

    ```
    cat /etc/systemd/system/docker.service.d/docker-network.conf
    
    [Service]
    Environment="DOCKER_NETWORK_OPTIONS= --bip=10.233.64.1/24"
    ```

### 5.8. 配置docker

    ```
    # Docker从1.13版本开始调整了默认的防火墙规则，禁用了iptables filter表中FOWARD链，这样会引起Kubernetes集群中跨Node的Pod无法通信，在各个Docker节点执行下面的命令：
    iptables -P FORWARD ACCEPT
    
    # 编辑systemctl的Docker启动文件
    sed -i "13i ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT" /usr/lib/systemd/system/docker.service
    
    # 新建配置文件
    cat /etc/docker/daemon.json
    {
    "log-driver": "json-file",
    "insecure-registries": [
        "docker.local.com (本地私库地址无https需要将域名或者ip地址添加到此处)"
    ],
    "storage-driver": "overlay2",
    "storage-opts":["overlay2.override_kernel_check=true"]
    }
    
    # kubespray 部署会自动配置日志大小，如果你配置了"log-opts"部分就要去掉, 不然无法启动docker会报错 (出现两个参数冲突)
    "unable to configure the Docker daemon with file /etc/docker/daemon.json: the following directives are specified both as a flag"
    
        "log-opts": {
        "max-size": "50m",
        "max-file": "6"
         },
    
    ```

1. **在服务器ops-01 IP：10.10.4.77上安装docker repo私库**

    ```
    假设docker私库域名为：registry-vpc.cn-shenzhen.aliyuncs.com
    安装过程省略。
    ```

## 6. 有服务器(ops-01)安装kubespray

    1. **安装所需软件**

    ```
    # sshpass 这个是我配置ansible-playbook使用账号密码登录所以需要安装密码登录支持的软件
    yum install -y python-pip python37 python37-pip ansible git sshpass

    # jinja2和netaddr也可通过yum instal -y python-netaddr python-jinja2 进行安装,但建议使用pip安装
    pip install --upgrade pip && pip install netaddr jinja2
    ```

    2. **获取kubespray并进行配置**
      
      ```
      git clone https://github.com/kubernetes-sigs/kubespray.git
      
      # 安装 kubespray 依赖，若无特殊说明，后续操作均在~/kubespray目录下执行
      cd kubespray
      pip install -r requirements.txt
      
      # 复制一份集群配置文件
      cp inventory/sample/ inventory/dev-cluster/ -ra
      
      ------
      
      # 修改inventory/dev-cluster/host.ini
      
      cat inventory/dev-cluster/host.ini
      
      # ## Configure 'ip' variable to bind kubernetes services on a
      # ## different ip than the default iface
      # ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
      [all]
      # node1 ansible_host=95.54.0.12  # ip=10.3.0.1 etcd_member_name=etcd1
      # node2 ansible_host=95.54.0.13  # ip=10.3.0.2 etcd_member_name=etcd2
      # node3 ansible_host=95.54.0.14  # ip=10.3.0.3 etcd_member_name=etcd3
      # node4 ansible_host=95.54.0.15  # ip=10.3.0.4 etcd_member_name=etcd4
      # node5 ansible_host=95.54.0.16  # ip=10.3.0.5 etcd_member_name=etcd5
      # node6 ansible_host=95.54.0.17  # ip=10.3.0.6 etcd_member_name=etcd6
      
      master1 ansible_ssh_host=10.10.4.81 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass=yourpassword etcd_member_name=etcd1
      master2 ansible_ssh_host=10.10.4.82 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass=yourpassword etcd_member_name=etcd2
      master3 ansible_ssh_host=10.10.4.83 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass=yourpassword etcd_member_name=etcd3
      node1 ansible_ssh_host=10.10.4.84 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass=yourpassword
      node2 ansible_ssh_host=10.10.4.85 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass=yourpassword
      node3 ansible_ssh_host=10.10.4.86 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass=yourpassword
      
      # ## configure a bastion host if your nodes are not directly reachable
      # bastion ansible_host=x.x.x.x ansible_user=some_user
      
      [kube-master]
      # node1
      # node2
      master1
      master2
      master3
      
      [etcd]
      # node1
      # node2
      # node3
      master1
      master2
      master3
      
      
      [kube-node]
      # node2
      # node3
      # node4
      # node5
      # node6
      node1
      node2
      node3
      
      [k8s-cluster:children]
      kube-master
      kube-node

      ------
      
      # 修改配置文件all.yaml
      
      vim inventory/mycluster/group_vars/all/all.yml
      # 修改如下配置:
      
      loadbalancer_apiserver_localhost: true
      # 加载内核模块，否则 ceph, gfs 等无法挂载客户端
      kubelet_load_modules: true
      
      ------
      
      # k8s 1.13.4所用到的 k8s.gcr.io / gcr.io的镜像列表
      # (cluster-proportional-autoscaler官方最新版本为1.4.0, 但k8s 1.13.4使用的是1.3.0版本)
      
      gcr.azk8s.cn/google-containers/kube-proxy:v1.13.4
      gcr.azk8s.cn/google-containers/kube-controller-manager:v1.13.4
      gcr.azk8s.cn/google-containers/kube-scheduler:v1.13.4
      gcr.azk8s.cn/google-containers/kube-apiserver:v1.13.4
      gcr.io/google-containers/pause-amd64:3.1
      gcr.io/google-containers/k8s-dns-kube-dns-amd64:1.15.1
      k8s.gcr.io/k8s-dns-node-cache:1.15.1
      gcr.io/google-containers/k8s-dns-dnsmasq-nanny-amd64:1.15.1
      gcr.io/google-containers/k8s-dns-sidecar-amd64:1.15.1
      gcr.io/google-containers/cluster-proportional-autoscaler-amd64:1.3.0
      gcr.io/kubernetes-helm/tiller:v2.13.0
      gcr.io/google-containers/kube-registry-proxy:0.4
      k8s.gcr.io/metrics-server-amd64:v0.3.1
      k8s.gcr.io/addon-resizer:1.8.4
      gcr.io/google-containers/kubernetes-dashboard-amd64:v1.10.1

      # 国内可拉取的最新版本kubernetes镜像 (微软azure镜像代理服务器进行拉取8s.gcr.io / gcr.io的镜像)
      
      gcr.azk8s.cn/google-containers/kube-proxy:v1.13.4
      gcr.azk8s.cn/google-containers/kube-controller-manager:v1.13.4
      gcr.azk8s.cn/google-containers/kube-scheduler:v1.13.4
      gcr.azk8s.cn/google-containers/kube-apiserver:v1.13.4
      gcr.azk8s.cn/google-containers/pause-amd64:3.1
      gcr.azk8s.cn/google-containers/k8s-dns-kube-dns-amd64:1.15.1
      gcr.azk8s.cn/google-containers/k8s-dns-node-cache:1.15.1
      gcr.azk8s.cn/google-containers/k8s-dns-dnsmasq-nanny-amd64:1.15.1
      gcr.azk8s.cn/google-containers/k8s-dns-sidecar-amd64:1.15.1
      gcr.azk8s.cn/google-containers/cluster-proportional-autoscaler-amd64:1.3.0
      gcr.azk8s.cn/kubernetes-helm/tiller:v2.13.0
      gcr.azk8s.cn/google-containers/kube-registry-proxy:0.4
      gcr.azk8s.cn/google-containers/metrics-server-amd64:v0.3.1
      gcr.azk8s.cn/google-containers/addon-resizer:1.8.4
      gcr.azk8s.cn/google-containers/kubernetes-dashboard-amd64:v1.10.1

      ------
      
      # quay.io的镜像国内是可访问的，所以不需要替换，如果有自建docker私库的话，可以拉取下来推到私库，然后替换quay.io为私库地址，这样拉取镜像快一些。
      
      # 替换镜像地址
      sed -i 's#k8s.gcr.io#gcr.azk8s.cn#g' roles/download/defaults/main.yml
      sed -i 's#gcr.io#gcr.azk8s.cn#g' roles/download/defaults/main.yml
      sed -i 's#google_containers#google-containers#g' roles/download/defaults/main.yml
      
      # 如果自建私库的话 
      sed -i 's#k8s.gcr.io#registry-vpc.cn-shenzhen.aliyuncs.com#g' roles/download/defaults/main.yml
      sed -i 's#gcr.io#registry-vpc.cn-shenzhen.aliyuncs.com#g' roles/download/defaults/main.yml
      
      sed -i 's#google_containers#google-containers#g' roles/download/defaults/main.yml
      
      ------
      
      # 修改代码，使用NodePort方式访问Dashboard。
      
      vim ./roles/kubernetes-apps/ansible/templates/dashboard.yml.j2
      
      # ------------------- Dashboard Service ------------------- #
      
      ……
      
      ……
      
      spec:
        type: NodePort  # 添加这一行
        ports:
          - port: 443
            targetPort: 8443
            nodePort: 31100 # 添加这一行
      
      ------
      # 注意：此处可不注释，如安装过程中遇到docker检查报错了可进行适当注释。
      
      
      # 禁用docker yum仓和docker安装

      vim ./roles/container-engine/docker/tasks/main.yml
      {% raw %}
      ---
      - name: gather os specific variables
        include_vars: "{{ item }}"
        with_first_found:
          - files:
              - "{{ ansible_distribution|lower }}-{{ ansible_distribution_version|lower|replace('/', '_') }}.yml"
              - "{{ ansible_distribution|lower }}-{{ ansible_distribution_release }}.yml"
              - "{{ ansible_distribution|lower }}-{{ ansible_distribution_major_version|lower|replace('/', '_') }}.yml"
              - "{{ ansible_distribution|lower }}.yml"
              - "{{ ansible_os_family|lower }}.yml"
              - defaults.yml
            paths:
              - ../vars
            skip: true
        tags:
          - facts
      
      - include: set_facts_dns.yml
        when: dns_mode != 'none' and resolvconf_mode == 'docker_dns'
        tags:
          - facts
      
      - name: check for minimum kernel version
        fail:
          msg: >
                docker requires a minimum kernel version of
                {{ docker_kernel_min_version }} on
                {{ ansible_distribution }}-{{ ansible_distribution_version }}
        when: (not ansible_os_family in ["CoreOS", "Container Linux by CoreOS"]) and (ansible_kernel|version_compare(docker_kernel_min_version, "<"))
        tags:
          - facts
      
      # 禁用docker仓库，已经使用清华源
      
      #- name: ensure docker repository public key is installed
      #  action: "{{ docker_repo_key_info.pkg_key }}"
      #  args:
      #    id: "{{item}}"
      #    keyserver: "{{docker_repo_key_info.keyserver}}"
      #    state: present
      #  register: keyserver_task_result
      #  until: keyserver_task_result|succeeded
      #  retries: 4
      #  delay: "{{ retry_stagger | random + 3 }}"
      #  environment: "{{ proxy_env }}"
      #  with_items: "{{ docker_repo_key_info.repo_keys }}"
      #  when: not (ansible_os_family in ["CoreOS", "Container Linux by CoreOS"] or is_atomic)
      
      #- name: ensure docker repository is enabled
      #  action: "{{ docker_repo_info.pkg_repo }}"
      #  args:
      #    repo: "{{item}}"
      #    state: present
      #  with_items: "{{ docker_repo_info.repos }}"
      #  when: not (ansible_os_family in ["CoreOS", "Container Linux by CoreOS"] or is_atomic) and(docker_repo_info.repos|length > 0)
      
      #- name: Configure docker repository on RedHat/CentOS
      #  template:
      #    src: "rh_docker.repo.j2"
      #    dest: "/etc/yum.repos.d/docker.repo"
      #  when: ansible_distribution in ["CentOS","RedHat"] and not is_atomic
      
      #- name: ensure docker packages are installed
      #  action: "{{ docker_package_info.pkg_mgr }}"
      #  args:
      #    pkg: "{{item.name}}"
      #    force: "{{item.force|default(omit)}}"
      #    state: present
      #  register: docker_task_result
      #  until: docker_task_result|succeeded
      #  retries: 4
      #  delay: "{{ retry_stagger | random + 3 }}"
      #  environment: "{{ proxy_env }}"
      #  with_items: "{{ docker_package_info.pkgs }}"
      #  notify: restart docker
      #  when: not (ansible_os_family in ["CoreOS", "Container Linux by CoreOS"] or is_atomic) and (docker_package_info.pkgs|length > 0)
      
      # 对于docker的版本检测进行了保留
      
      - name: check minimum docker version for docker_dns mode. You need at least docker version >= 1.12 for resolvconf_mode=docker_dns
        command: "docker version -f '{{ '{{' }}.Client.Version{{ '}}' }}'"
        register: docker_version
        failed_when: docker_version.stdout|version_compare('1.12', '<')
        changed_when: false
        when: dns_mode != 'none' and resolvconf_mode == 'docker_dns'
      
      # 对于docker的systemd配置，可以根据自己需求修改，但是注意会覆盖原来的
      - name: Set docker systemd config
        include: systemd.yml
      
      - name: ensure docker service is started and enabled
        service:
          name: "{{ item }}"
          enabled: yes
          state: started
        with_items:
          - docker
      {% endraw %}
      ------

      # 禁用虚拟内存检查
      vim roles/kubernetes/node/defaults/main.yml 

      ### fail with swap on (default true)
      # kubelet_fail_swap_on: true
      kubelet_fail_swap_on: false

      ------
      开始安装
      nsible-playbook -i inventory/mycluster/hosts.ini scale.yml -b
      
      遇到错误问题


      ------
      
      运维经验
      如果需要扩容Work节点，则修改hosts.ini文件，增加新增的机器信息 (-k, --ask-pass 询问密码, -b 提升权限)。然后执行下面的命令：
      ansible-playbook -i inventory/mycluster/hosts.ini scale.yml -b -v -k
      
      将hosts.ini文件中的master和etcd的机器增加到多台，执行部署命令
      ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml -b -vvv
      
      刪除节点，如果不指定节点就是刪除整个集群：
      ansible-playbook -i inventory/mycluster/hosts.ini remove-node.yml -b -v
      
      如果需要卸载，可以执行以下命令：
      ansible-playbook -i inventory/mycluster/hosts.ini reset.yml -b –vvv
      
      升级K8s集群，选择对应的k8s版本信息，执行升级命令。涉及文件为upgrade-cluster.yml。
      ansible-playbook upgrade-cluster.yml -b -i inventory/mycluster/hosts.ini -e kube_version=vX.XX.XX -vvv

      ------
      
      获取admin的token
      # kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token  
      
      验证K8s集群, 查看集群状态
      # kubectl get nodes
      
      查看ipvs
      # ipvsadm -L -n
      
      ------
      
      # 对kubespray文件修改了以下文件
      
      # 修改文件的列表如下
      modified:   roles/container-engine/docker/tasks/main.yml
      modified:   roles/container-engine/docker/tasks/pre-upgrade.yml
      modified:   roles/download/defaults/main.yml
      modified:   roles/kubernetes-apps/ansible/templates/dashboard.yml.j2
      
      # 新增的文件夹
      inventory/dev-cluster
      ```


## 7. kubespray部署高可用原理
    
    所有node节点都部署了nginx，在各自node节点上代理了三个master节点api地址。然后kubelet 直接访问节点本机的代理地址实现高可用。
    
## 8. 其他

    yum-utils 所需依赖包libxml2-python python-chardet python-kitchen
    
    K8s新版本中，针对无状态类服务推荐使用Deployment，有状态类服务则建议使用Statefulset。RC和RS已不支持目前K8s的诸多新特性了。
    
    K8s从1.11版本起便废弃了Heapster监控组件，取而代之的是metrics-server 和 custom metrics API，后面将陆续完善包括Prometheus+Grafana监控，Kibana+Fluentd日志管理，cephfs-provisioner存储（可能需要重新build kube-controller-manager装上rbd相关的包才能使用Ceph RBD StorageClass），traefik ingress等服务。
