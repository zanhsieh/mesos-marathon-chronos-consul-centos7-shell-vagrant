# -*- mode: ruby -*-
# vi: set ft=ruby :

$IPs = {
  "mesos-master1"  => "192.168.122.11",
  "mesos-master2"  => "192.168.122.12",
  "mesos-master3"  => "192.168.122.13",
  "mesos-slave1"   => "192.168.122.15",
  "mesos-slave2"   => "192.168.122.16",
  "mesos-slave3"   => "192.168.122.17"
}

$Masters = $IPs.select { |k,v| k.to_s.match(/^mesos-master\d+/) }
$zk = $Masters.inject({}) { |h, (k, v)| h[k] = v+":2181"; h }.values.join(',')

def gen_zk_cfg
  zk_lines = ""
  $Masters.values.each_with_index {|val,index|  
    zk_lines += <<-TEXT
server.#{index+1}=#{val}:2888:3888
    TEXT
  }
  return zk_lines
end

def host_check(ips={})
  return ips.map {|k,v|<<-INNER
if [ ! `grep -q #{v} /etc/hosts` ]; then
  echo '#{v} #{k}' | sudo tee -a /etc/hosts
fi
  INNER
  }.join
end

def disable_fw
  return<<-INNER
systemctl stop firewalld
systemctl disable firewalld
systemctl mask firewalld
  INNER
end

def install_prereq
  return<<-INNER
setenforce 0
rpm -i /vagrant/jdk-8u72-linux-x64.rpm
#rpm -Uvh /vagrant/mesosphere-el-repo-7-1.noarch.rpm
rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
  INNER
end

def gen_consul_daemon(config_dir='/etc/consul.d')
  unit_name = ''
  if config_dir["server"]
    unit_name = 'server'
  elsif config_dir["client"]
    unit_name = 'client'
  end
  return<<-INNER
[Unit]
Description=consul #{unit_name} process
Requires=network-online.target
After=network-online.target

[Service]
EnvironmentFile=-/etc/sysconfig/consul
Environment=GOMAXPROCS=2
Restart=on-failure
ExecStart=/usr/bin/consul agent $OPTIONS -config-dir=#{config_dir}
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGINT
User=consul
Group=users

[Install]
WantedBy=multi-user.target
  INNER
end

def gen_server_consul_config(m)
  case m
  when "mesos-master1"
    return<<-INNER
{
    "advertise_addr": "#{$IPs[m]}",
    "bootstrap_expect": 3,
    "server": true,
    "datacenter": "ming_latitude",
    "data_dir": "/var/consul",
    "encrypt": "DWOfNNQE4ICAXT5/+SE4fw==",
    "log_level": "INFO",
    "enable_syslog": true,
    "bind_addr": "#{$IPs[m]}",
    "ui": true,
    "ui_dir": "/home/consul",
    "http_api_response_headers": {
        "Access-Control-Allow-Origin": "*"
    },
    "addresses": {
        "http": "0.0.0.0"
    }
}
    INNER
  else
    return<<-INNER
{
    "advertise_addr": "#{$IPs[m]}",
    "bootstrap": false,
    "server": true,
    "datacenter": "ming_latitude",
    "data_dir": "/var/consul",
    "encrypt": "DWOfNNQE4ICAXT5/+SE4fw==",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": ["#{$IPs['mesos-master1']}"],
    "bind_addr": "#{$IPs[m]}",
    "ui": true,
    "ui_dir": "/home/consul",
    "http_api_response_headers": {
        "Access-Control-Allow-Origin": "*"
    },
    "addresses": {
        "http": "0.0.0.0"
    }
}
    INNER
  end
end

def gen_client_consul_config(n)
  return<<-INNER
{
    "advertise_addr": "#{$IPs[n]}",
    "server": false,
    "datacenter": "ming_latitude",
    "data_dir": "/var/consul",
    "encrypt": "DWOfNNQE4ICAXT5/+SE4fw==",
    "log_level": "INFO",
    "enable_syslog": true,
    "start_join": #{$Masters.values.to_json},
    "bind_addr": "#{$IPs[n]}"
}
  INNER
end


def master_setup(m)
  return <<-MASTERSETUP
if [ ! -f "/var/master_setup_done" ]; then
yum -y install unzip bind-utils mesosphere-zookeeper
echo #{m[-1]} > /var/lib/zookeeper/myid
tee /etc/zookeeper/conf/zoo.cfg <<-EOF
maxClientCnxns=50
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
clientPort=2181
#{gen_zk_cfg}
EOF
systemctl start zookeeper && systemctl enable zookeeper
yum -y install mesos nc
echo 'zk://#{$zk}/mesos' > /etc/mesos/zk
echo #{($Masters.length/2.0).ceil} > /etc/mesos-master/quorum
systemctl stop mesos-slave && systemctl disable mesos-slave
echo '#{$IPs[m]}' > /etc/mesos-master/hostname
echo '#{$IPs[m]}' > /etc/mesos-master/ip
systemctl restart mesos-master && systemctl enable mesos-master
yum -y install marathon chronos
mkdir -p /etc/{marathon,chronos}/conf/
cp /etc/mesos-master/hostname /etc/marathon/conf/hostname
echo 'zk://#{$zk}/mesos' > /etc/marathon/conf/master
echo 'zk://#{$zk}/marathon' > /etc/marathon/conf/zk
cp /etc/mesos-master/hostname /etc/chronos/conf/hostname
systemctl restart marathon && systemctl enable marathon
systemctl restart chronos && systemctl enable chronos
adduser consul
unzip /vagrant/consul_0.6.3_linux_amd64.zip -d /usr/bin
unzip /vagrant/consul_0.6.3_web_ui.zip -d /home/consul
mkdir -p /var/consul /etc/consul.d/server
chown consul:consul /var/consul
tee /etc/consul.d/server/config.json <<-EOF
#{gen_server_consul_config(m)}
EOF
tee /usr/lib/systemd/system/consul-server.service <<-EOF
#{gen_consul_daemon('/etc/consul.d/server')}
EOF
systemctl daemon-reload
systemctl start consul-server
touch /var/master_setup_done
fi
  MASTERSETUP
end

def slave_setup(n)
  return <<-SLAVESETUP
if [ ! -f "/var/minion_setup_done" ]; then
yum -y install unzip bind-utils mesos
echo 'zk://#{$zk}/mesos' > /etc/mesos/zk
echo '#{$IPs[n]}' > /etc/mesos-slave/hostname
echo '#{$IPs[n]}' > /etc/mesos-slave/ip
echo 'docker,mesos' > /etc/mesos-slave/containerizers
echo '5mins' > /etc/mesos-slave/executor_registration_timeout
echo "ports:[1025-65000]" > /etc/mesos-slave/resources
systemctl stop mesos-master && systemctl disable mesos-master
yum -y remove docker-io device-mapper-event-libs
yum -y install docker-io device-mapper-event-libs
tee /etc/sysconfig/docker-network <<-EOF
DOCKER_NETWORK_OPTIONS=-H tcp://0.0.0.0:2375 -H tcp://0.0.0.0:4243 -H unix:///var/run/docker.sock
EOF
systemctl restart docker && systemctl enable docker
systemctl restart mesos-slave && systemctl enable mesos-slave
adduser consul
unzip /vagrant/consul_0.6.3_linux_amd64.zip -d /usr/bin
mkdir -p /var/consul /etc/consul.d/client
chown consul:consul /var/consul
tee /etc/consul.d/client/config.json <<-EOF
#{gen_client_consul_config(n)}
EOF
tee /usr/lib/systemd/system/consul-client.service <<-EOF
#{gen_consul_daemon('/etc/consul.d/client')}
EOF
systemctl daemon-reload
systemctl start consul-client
touch /var/minion_setup_done
fi
  SLAVESETUP
end

Vagrant.configure(2) do |config|
  config.vm.box = "centos71-1511"
  config.vm.box_check_update = false
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = "box"
    config.cache.synced_folder_opts = {
      type: "nfs",
      mount_options: ['rw', 'vers=3', 'tcp', 'nolock']
    }
  end
  $IPs.map do |k,v|
    config.vm.define "#{k}" do |m|
      m.vm.hostname = "#{k}"
      m.vm.network "private_network", ip: "#{v}"
      m.vm.provider "virtualbox" do |v|
        case k
        when "mesos-master1", "mesos-master2", "mesos-master3"
          v.memory = 256
        else
          v.memory = 1024
        end
      end
      m.vm.provision "shell", inline:<<-SHELL
      #{host_check($IPs)}
      #{disable_fw}
      #{install_prereq}
      SHELL
      m.vm.provision "shell", inline:<<-SHELL
      #{case k
        when "mesos-master1", "mesos-master2", "mesos-master3"
          master_setup(k)
        else
          slave_setup(k)
        end
      }
      SHELL
    end
  end
end
