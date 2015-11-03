# -*- mode: ruby -*-
# vi: set ft=ruby :

ZOOKEEPER_VERSION = "3.4.6"
MARATHON_VERSION = "v0.11.1"
CHRONOS_VERSION = "latest"
MESOS_VERSION = "0.24.1"
NUMBER_OF_SLAVES = 2
MASTER_MEMORY = "1024"
SLAVE_MEMORY = "4096"
SLAVE_CPUS = "4"

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  slave_host_entries = (1..NUMBER_OF_SLAVES).inject("") do |result, num|
    result + "192.168.33.1#{num} mesos-slave#{num}.vagrant mesos-slave#{num}\n"
  end

  $hosts = <<SCRIPT
echo "127.0.0.1 localhost
192.168.33.10 mesos-master.vagrant mesos-master
#{slave_host_entries}" > /etc/hosts
SCRIPT

  $master = <<SCRIPT
export ZK_CONNECTION_STRING="zk://mesos-master.vagrant:2181"
export ENV_NAME="vagrant"
setup-runit-service docker "/usr/bin/docker -d --host=unix:///var/run/docker.sock --storage-driver=aufs"
sleep 5
setup-docker-runit-service zookeeper "--net=host jplock/zookeeper:#{ZOOKEEPER_VERSION}"
setup-runit-service mesos-master "mesos-master --work_dir=/tmp --port=5050 --zk=$ZK_CONNECTION_STRING/mesos --cluster=$ENV_NAME --quorum=1 --hostname=$HOSTNAME"
setup-docker-runit-service marathon "--net=host mesosphere/marathon:#{MARATHON_VERSION} --master $ZK_CONNECTION_STRING/mesos --zk $ZK_CONNECTION_STRING/marathon --hostname $HOSTNAME"
setup-docker-runit-service chronos "--net=host banno/chronos:#{CHRONOS_VERSION} --master $ZK_CONNECTION_STRING/mesos --zk_hosts $ZK_CONNECTION_STRING --http_port 8081"
SCRIPT

  $slave = <<SCRIPT
export ZK_CONNECTION_STRING="zk://mesos-master.vagrant:2181"
export ENV_NAME="vagrant"
setup-runit-service docker "/usr/bin/docker -d --host=unix:///var/run/docker.sock --storage-driver=aufs"
setup-runit-service mesos-slave "mesos-slave --work_dir=/tmp --master=$ZK_CONNECTION_STRING/mesos --containerizers=docker,mesos --hostname=$HOSTNAME --docker_stop_timeout=30secs --executor_registration_timeout=5mins --resources=mem:#{SLAVE_MEMORY.to_i - 256}"
SCRIPT
  $kubernetes = <<SCRIPT
export ZK_CONNECTION_STRING="zk://mesos-master.vagrant:2181"
a=$(hostname -i)
IFS=" "
array=($a)
delete=("127.0.1.1")
result=${array[@]/$delete}
echo $result
export KUBERNETES_MASTER_IP=$result
export KUBERNETES_MASTER=http://${KUBERNETES_MASTER_IP}:8888
export PATH="/opt/kubernetes/_output/local/go/bin:$PATH"
export MESOS_MASTER="$ZK_CONNECTION_STRING/mesos"

cat <<EOF >mesos-cloud.conf
[mesos-cloud]
        mesos-master        = ${MESOS_MASTER}
EOF

[[ ! -d "/opt/kubernetes" ]] && sudo git clone https://github.com/scorpionis/ubuntu-kubernetes-mesos.git /opt/kubernetes
sudo docker rm -f etcd
sudo docker run -d --hostname $(uname -n) --name etcd \
              -p 4001:4001 -p 7001:7001 quay.io/coreos/etcd:v2.0.12 \
              --listen-client-urls http://0.0.0.0:4001 \
              --advertise-client-urls http://${KUBERNETES_MASTER_IP}:4001
sleep 10s

km apiserver \
  --address=${KUBERNETES_MASTER_IP} \
  --etcd-servers=http://${KUBERNETES_MASTER_IP}:4001 \
  --service-cluster-ip-range=10.10.10.0/24 \
  --port=8888 \
  --cloud-provider=mesos \
  --cloud-config=mesos-cloud.conf \
  --v=1 >apiserver.log 2>&1 &
sleep 10s
km controller-manager \
  --master=${KUBERNETES_MASTER_IP}:8888 \
  --cloud-provider=mesos \
  --cloud-config=./mesos-cloud.conf  \
  --v=1 >controller.log 2>&1 &
sleep 10s
km scheduler \
  --address=${KUBERNETES_MASTER_IP} \
  --mesos-master=${MESOS_MASTER} \
  --etcd-servers=http://${KUBERNETES_MASTER_IP}:4001 \
  --mesos-user=root \
  --api-servers=${KUBERNETES_MASTER_IP}:8888 \
  --cluster-dns=10.10.10.10 \
  --cluster-domain=cluster.local \
  --v=2 >scheduler.log 2>&1 &
disown -a
SCRIPT
  config.vm.box = "banno/mesos"
  config.vm.box_version = MESOS_VERSION

  config.vm.provision "file", source: "scripts/setup-runit-service", destination: "/home/vagrant/bin/setup-runit-service"
  config.vm.provision "file", source: "scripts/setup-docker-runit-service", destination: "/home/vagrant/bin/setup-docker-runit-service"
  config.vm.provision "shell", privileged: true, inline: "mv /home/vagrant/bin/* /usr/local/bin"
  config.vm.provision "shell", inline: $hosts

  unless Dir["CAs/*"].empty?
    Dir["CAs/*"].each do |crt|
      config.vm.provision "file", source: "#{crt}", destination: "/home/vagrant/#{crt}"
      config.vm.provision "shell", privileged: true, inline: "rsync -plarq /home/vagrant/CAs/ /usr/local/share/ca-certificates/ && sudo update-ca-certificates"
    end
  end

  config.vm.define "master" do |master_config|
    master_config.vm.network "private_network", ip: "192.168.33.10"
    master_config.vm.host_name = "mesos-master.vagrant"
    master_config.vm.provision "shell", inline: $master
    master_config.vm.provision "shell", inline: $kubernetes
    master_config.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--memory", MASTER_MEMORY]
    end
    # synced folder example: master_config.vm.synced_folder "~/repo", "/repo"
  end

  (1..NUMBER_OF_SLAVES).each do |id|
    config.vm.define "slave#{id}" do |slave_config|
      slave_config.vm.network "private_network", ip: "192.168.33.1#{id}"
      slave_config.vm.host_name = "mesos-slave#{id}.vagrant"
      slave_config.vm.provision "shell", inline: $slave
      slave_config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", SLAVE_MEMORY]
        v.cpus = SLAVE_CPUS
      end
      # synced folder example: master_config.vm.synced_folder "~/repo", "/repo"
    end
  end
end
