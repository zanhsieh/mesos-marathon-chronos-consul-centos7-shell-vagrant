mesos-marathon-chronos-marathon-centos7-shell-vagrant
=====================================================

Install mesos, marathon, chronos, consul on CentOS 7.1 with bash shell and vagrant

### Hardware Requirements
Tested with MacBook Pro 16GB RAM and Dell Latitude E5250 16GB RAM. After all VMs launched, it uses around 6GB memory.

### Getting Started

* Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
* Install [Vagrant](https://www.vagrantup.com/downloads.html)
* Install Vagrant plugin cashier:
````
vagrant plugin install vagrant-cachier
````
* Pack CentOS 7.1 vagrant box from official iso with Packer (CentOS 7.1 template [here](https://github.com/shiguredo/packer-templates))
````
vagrant box add centos71-1511 /path/to/your/packer_gen_vagrantbox_file.box
````
* Git clone this repo
* Go into project directory and call ````vagrant up````

### Topology

* 3x master VM (zookeeper, mesos-master, marathon, chronos, consul server mode)
* 3x slave VM (docker, mesos-slave, consul client mode)

### Version
* sun jdk 8u73
* mesos 0.27.0 (from mesosphere el7 repo)
* zookeeper 3.4.5 (embedded version from mesosphere el7 repo)
* chronos 2.4.0 (from mesosphere el7 repo)
* marathon 0.15.1 (from mesosphere el7 repo)
* consul, consul_web_ui 0.6.3
* docker 1.8.2-el7

