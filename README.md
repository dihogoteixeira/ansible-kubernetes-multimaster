![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)

Deploy a Production Ready Kubernetes Cluster
============================================

If you have questions, check the [documentation](https://kubespray.io) and join us on the [kubernetes slack](https://kubernetes.slack.com), channel **\#kubespray**.
You can get your invite [here](http://slack.k8s.io/)

-   Can be deployed on **AWS, GCE, Azure, OpenStack, vSphere, Packet (bare metal), Oracle Cloud Infrastructure (Experimental), or Baremetal**
-   **Highly available** cluster
-   **Composable** (Choice of the network plugin for instance)
-   Supports most popular **Linux distributions**
-   **Continuous integration tests**

Quick Start
-----------

Para realzar o deploy do cluster você pode usar:

### Aplique as informacoes dos hosts em todas as maquinas:

`Exemplo:`
```
echo -e ' 10.11.0.43 servia01 servia01.gcse.local\n
          10.11.0.44 servia02 servia02.gcse.local\n
          10.11.0.45 samoa01 samoa01.gcse.local\n
          10.11.0.46 samoa02 samoa02.gcse.local\n
          10.11.0.47 samoa03 samoa03.gcse.local\n
          10.11.0.48 samoa04 samoa04.gcse.local\n
          10.11.0.67 samoa05 samoa05.gcse.local\n
          10.11.0.68 samoa06 samoa06.gcse.local\n
          10.11.0.69 samoa07 samoa07.gcse.local\n
          10.11.0.70 samoa08 samoa08.gcse.local\n
          10.11.0.71 samoa09 samoa09.gcse.local\n
          10.11.0.82 samoa10 samoa10.gcse.local\n
          10.11.0.83 samoa11 samoa11.gcse.local\n
          10.11.0.88 samoa12 samoa12.gcse.local\n
          10.11.0.89 samoa13 samoa13.gcse.local\n
          10.11.0.90 samoa14 samoa14.gcse.local\n
          10.11.0.91 samoa15 samoa15.gcse.local\n' >> /etc/hosts

```

### Criando regras de Firewalld. :Aplique essas regras em todos os `Masters`:

```
firewall-cmd --permanent --add-port=6443/tcp

firewall-cmd --permanent --add-port=2379-2380/tcp

firewall-cmd --permanent --add-port=10250/tcp

firewall-cmd --permanent --add-port=10251/tcp

firewall-cmd --permanent --add-port=10252/tcp

firewall-cmd --permanent --add-port=10255/tcp

firewall-cmd --permanent --add-port=2380/tcp

firewall-cmd --permanent --add-port=2379/tcp

modprobe br_netfilter

echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables

sysctl -w net.ipv4.ip_forward=1
```

### Criando regras de Firewalld. :Aplique essas regras em todos os `Nodes`:

```
firewall-cmd --permanent --add-port=10250/tcp

firewall-cmd --permanent --add-port=10255/tcp

firewall-cmd --permanent --add-port=30000-32767/tcp

firewall-cmd --permanent --add-port=6783/tcp

echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables

sysctl -w net.ipv4.ip_forward=1
```

### Configure o ETCD em todos os `Masters`:

```
    cd /usr/local/src
    wget "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
    sudo tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
    sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
    sudo mkdir -p /etc/etcd /var/lib/etcd
    sudo groupadd -f -g 1501 etcd
    sudo useradd -c "etcd user" -d /var/lib/etcd -s /bin/false -g etcd -u 1501 etcd
    sudo chown -R etcd:etcd /var/lib/etcd
    ETCD_HOST_IP=$(ip addr show eth1 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1)
    ETCD_NAME=$(hostname -s)
    sudo touch /lib/systemd/system/etcd.service
```
`Exemplo Master01:`
```
    echo -e '[Unit]\n
                Description=etcd docker wrapper\n
                Wants=docker.socket\n
                After=docker.service\n
                [Service]\n
                User=root\n
                PermissionsStartOnly=true\n
                EnvironmentFile=-/etc/etcd.env\n
                ExecStart=/usr/local/bin/etcd\n
                ExecStartPre=-/usr/bin/docker rm -f etcd1\n
                ExecStop=/usr/bin/docker stop etcd1\n
                Restart=always\n
                RestartSec=15s\n
                TimeoutStartSec=30s\n
                [Install]\n
                WantedBy=multi-user.target\n' > /lib/systemd/system/etcd.service
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl start etcd.service
    sudo systemctl status -l etcd.service

```
`Exemplo Master02:`
```
    echo -e '[Unit]\n
                Description=etcd docker wrapper\n
                Wants=docker.socket\n
                After=docker.service\n
                [Service]\n
                User=root\n
                PermissionsStartOnly=true\n
                EnvironmentFile=-/etc/etcd.env\n
                ExecStart=/usr/local/bin/etcd\n
                ExecStartPre=-/usr/bin/docker rm -f etcd2\n
                ExecStop=/usr/bin/docker stop etcd2\n
                Restart=always\n
                RestartSec=15s\n
                TimeoutStartSec=30s\n
                [Install]\n
                WantedBy=multi-user.target\n' > /lib/systemd/system/etcd.service
    sudo systemctl daemon-reload
    sudo systemctl enable etcd
    sudo systemctl start etcd.service
    sudo systemctl status -l etcd.service
```

### Ansible

#### Modo de usar:

    # Instalando dependencias para ``requirements.txt``
    sudo pip install -r requirements.txt

    # Cop[iando``inventory/sample`` como ``inventory/nome_do_cluster``
    cp -rfp inventory/sample inventory/nome_do_cluster

    # Update Ansible inventory file with inventory builder
    declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
    CONFIG_FILE=inventory/nome_do_cluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

    # Revisando e mudando parametros dentro de ``inventory/nome_do_cluster/group_vars``
    cat inventory/nome_do_cluster/group_vars/all/all.yml
    cat inventory/nome_do_cluster/group_vars/k8s-cluster/k8s-cluster.yml

    # Deploy Kubespray with Ansible Playbook - run the playbook as root
    # The option `--become` is required, as for example writing SSL keys in /etc/,
    # installing packages and interacting with various systemd daemons.
    # Without --become the playbook will fail to run!
    ansible-playbook -i inventory/nome_do_cluster/hosts.yml --become --become-user=root cluster.yml


Note: Quando o Ansible já está instalado via pacotes do sistema na máquina de controle, outros pacotes python instalados via `sudo pip install -r requirements.txt` irá para uma árvore de diretórios diferente (e.g. `/usr/local/lib/python2.7/dist-packages` on Ubuntu) de Ansible (e.g. `/usr/lib/python2.7/dist-packages/ansible` still on Ubuntu).
Como consequência, `ansible-playbook` o comando falhará com:
```
ERROR! no action detected in task. This often indicates a misspelled module name, or incorrect module path.
```
probably pointing on a task depending on a module present in requirements.txt (i.e. "unseal vault").

One way of solving this would be to uninstall the Ansible package and then, to install it via pip but it is not always possible.
A workaround consists of setting `ANSIBLE_LIBRARY` and `ANSIBLE_MODULE_UTILS` environment variables respectively to the `ansible/modules` and `ansible/module_utils` subdirectories of pip packages installation location, which can be found in the Location field of the output of `pip show [package]` before executing `ansible-playbook`.

### Vagrant

For Vagrant we need to install python dependencies for provisioning tasks.
Check if Python and pip are installed:

```
    python -V && pip -V
```

If this returns the version of the software, you're good to go. If not, download and install Python from here <https://www.python.org/downloads/source/>
Install the necessary requirements

```
    sudo pip install -r requirements.txt
    vagrant up
```

Documents
---------

-   [Requirements](#requirements)
-   [Kubespray vs ...](docs/comparisons.md)
-   [Getting started](docs/getting-started.md)
-   [Ansible inventory and tags](docs/ansible.md)
-   [Integration with existing ansible repo](docs/integration.md)
-   [Deployment data variables](docs/vars.md)
-   [DNS stack](docs/dns-stack.md)
-   [HA mode](docs/ha-mode.md)
-   [Network plugins](#network-plugins)
-   [Vagrant install](docs/vagrant.md)
-   [CoreOS bootstrap](docs/coreos.md)
-   [Debian Jessie setup](docs/debian.md)
-   [openSUSE setup](docs/opensuse.md)
-   [Downloaded artifacts](docs/downloads.md)
-   [Cloud providers](docs/cloud.md)
-   [OpenStack](docs/openstack.md)
-   [AWS](docs/aws.md)
-   [Azure](docs/azure.md)
-   [vSphere](docs/vsphere.md)
-   [Packet Host](docs/packet.md)
-   [Large deployments](docs/large-deployments.md)
-   [Upgrades basics](docs/upgrades.md)
-   [Roadmap](docs/roadmap.md)

Supported Linux Distributions
-----------------------------

- [x] **Container Linux by CoreOS**
- [x] **Debian** Buster, Jessie, Stretch, Wheezy
- [x] **Ubuntu** 16.04, 18.04
- [x] **CentOS/RHEL** 7
- [x] **Fedora** 28
- [x] **Fedora/CentOS** Atomic
- [x] **openSUSE** Leap 42.3/Tumbleweed
- [x] **Oracle Linux** 7

Note: Upstart/SysV init based OS types are not supported.

Supported Components
--------------------

-   Core
    -   [kubernetes](https://github.com/kubernetes/kubernetes) v1.15.2
    -   [etcd](https://github.com/coreos/etcd) v3.3.10
    -   [docker](https://www.docker.com/) v18.06 (see note)
    -   [cri-o](http://cri-o.io/) v1.11.5 (experimental: see [CRI-O Note](docs/cri-o.md). Only on centos based OS)
-   Network Plugin
    -   [cni-plugins](https://github.com/containernetworking/plugins) v0.8.1
    -   [calico](https://github.com/projectcalico/calico) v3.7.3
    -   [canal](https://github.com/projectcalico/canal) (given calico/flannel versions)
    -   [cilium](https://github.com/cilium/cilium) v1.5.5
    -   [contiv](https://github.com/contiv/install) v1.2.1
    -   [flanneld](https://github.com/coreos/flannel) v0.11.0
    -   [kube-router](https://github.com/cloudnativelabs/kube-router) v0.2.5
    -   [multus](https://github.com/intel/multus-cni) v3.2.1
    -   [weave](https://github.com/weaveworks/weave) v2.5.2
-   Application
    -   [cephfs-provisioner](https://github.com/kubernetes-incubator/external-storage) v2.1.0-k8s1.11
    -   [rbd-provisioner](https://github.com/kubernetes-incubator/external-storage) v2.1.1-k8s1.11
    -   [cert-manager](https://github.com/jetstack/cert-manager) v0.5.2
    -   [coredns](https://github.com/coredns/coredns) v1.6.0
    -   [ingress-nginx](https://github.com/kubernetes/ingress-nginx) v0.21.0

Note: The list of validated [docker versions](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md) was updated to 1.11.1, 1.12.1, 1.13.1, 17.03, 17.06, 17.09, 18.06. kubeadm now properly recognizes Docker 18.09.0 and newer, but still treats 18.06 as the default supported version. The kubelet might break on docker's non-standard version numbering (it no longer uses semantic versioning). To ensure auto-updates don't break your cluster look into e.g. yum versionlock plugin or apt pin).

Requirements
------------
-   **Minimum required version of Kubernetes is v1.14**
-   **Ansible v2.7.8 (or newer, but [not 2.8.x](https://github.com/kubernetes-sigs/kubespray/issues/4778)) and python-netaddr is installed on the machine
    that will run Ansible commands**
-   **Jinja 2.9 (or newer) is required to run the Ansible Playbooks**
-   The target servers must have **access to the Internet** in order to pull docker images. Otherwise, additional configuration is required (See [Offline Environment](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/downloads.md#offline-environment))
-   The target servers are configured to allow **IPv4 forwarding**.
-   **Your ssh key must be copied** to all the servers part of your inventory.
-   The **firewalls are not managed**, you'll need to implement your own rules the way you used to.
    in order to avoid any issue during deployment you should disable your firewall.
-   If kubespray is ran from non-root user account, correct privilege escalation method
    should be configured in the target servers. Then the `ansible_become` flag
    or command parameters `--become or -b` should be specified.

Hardware:
These limits are safe guarded by Kubespray. Actual requirements for your workload can differ. For a sizing guide go to the [Building Large Clusters](https://kubernetes.io/docs/setup/cluster-large/#size-of-master-and-master-components) guide.

-   Master
    - Memory: 1500 MB
-   Node
    - Memory: 1024 MB

Network Plugins
---------------

You can choose between 10 network plugins. (default: `calico`, except Vagrant uses `flannel`)

-   [flannel](docs/flannel.md): gre/vxlan (layer 2) networking.

-   [calico](docs/calico.md): bgp (layer 3) networking.

-   [canal](https://github.com/projectcalico/canal): a composition of calico and flannel plugins.

-   [cilium](http://docs.cilium.io/en/latest/): layer 3/4 networking (as well as layer 7 to protect and secure application protocols), supports dynamic insertion of BPF bytecode into the Linux kernel to implement security services, networking and visibility logic.

-   [contiv](docs/contiv.md): supports vlan, vxlan, bgp and Cisco SDN networking. This plugin is able to
    apply firewall policies, segregate containers in multiple network and bridging pods onto physical networks.

-   [weave](docs/weave.md): Weave is a lightweight container overlay network that doesn't require an external K/V database cluster.
    (Please refer to `weave` [troubleshooting documentation](https://www.weave.works/docs/net/latest/troubleshooting/)).

-   [kube-ovn](docs/kube-ovn.md): Kube-OVN integrates the OVN-based Network Virtualization with Kubernetes. It offers an advanced Container Network Fabric for Enterprises.

-   [kube-router](docs/kube-router.md): Kube-router is a L3 CNI for Kubernetes networking aiming to provide operational
    simplicity and high performance: it uses IPVS to provide Kube Services Proxy (if setup to replace kube-proxy),
    iptables for network policies, and BGP for ods L3 networking (with optionally BGP peering with out-of-cluster BGP peers).
    It can also optionally advertise routes to Kubernetes cluster Pods CIDRs, ClusterIPs, ExternalIPs and LoadBalancerIPs.

-   [macvlan](docs/macvlan.md): Macvlan is a Linux network driver. Pods have their own unique Mac and Ip address, connected directly the physical (layer 2) network.

-   [multus](docs/multus.md): Multus is a meta CNI plugin that provides multiple network interface support to pods. For each interface Multus delegates CNI calls to secondary CNI plugins such as Calico, macvlan, etc.

The choice is defined with the variable `kube_network_plugin`. There is also an
option to leverage built-in cloud provider networking instead.
See also [Network checker](docs/netcheck.md).

Community docs and resources
----------------------------

-   [kubernetes.io/docs/getting-started-guides/kubespray/](https://kubernetes.io/docs/getting-started-guides/kubespray/)
-   [kubespray, monitoring and logging](https://github.com/gregbkr/kubernetes-kargo-logging-monitoring) by @gregbkr
-   [Deploy Kubernetes w/ Ansible & Terraform](https://rsmitty.github.io/Terraform-Ansible-Kubernetes/) by @rsmitty
-   [Deploy a Kubernetes Cluster with Kubespray (video)](https://www.youtube.com/watch?v=N9q51JgbWu8)

Tools and projects on top of Kubespray
--------------------------------------

-   [Digital Rebar Provision](https://github.com/digitalrebar/provision/blob/master/doc/integrations/ansible.rst)
-   [Terraform Contrib](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform)

CI Tests
--------

[![Build graphs](https://gitlab.com/kargo-ci/kubernetes-sigs-kubespray/badges/master/build.svg)](https://gitlab.com/kargo-ci/kubernetes-sigs-kubespray/pipelines)

CI/end-to-end tests sponsored by Google (GCE)
See the [test matrix](docs/test_cases.md) for details.
