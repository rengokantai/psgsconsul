#### psgsconsul
######13
start first consul node
```
vagrant init --minimal boxcutter/ubuntu1404-docker
```
######14 
edit Vagrantfile
```
Vagrant.configure("2") do |config|
  config.vm.box = "boxcutter/ubuntu1404-docker"
  config.vm.define "consul-server" do |cs|
    cs.vm.hostname = "consul-server"
    cs.vm.network "private_network",ip:"172.20.20.31"
  end

end
```
then
```
vagrant up
vagrant ssh consul-server
```
######17 installingconsul
install.sh
```
#!/bin/bash
sudo apt update
sudo apt install -y unzip

CONSUL=0.6.4
curl https://releases.hashicorp.com/consul/${CONSUL}/consul_${CONSUL}_linux_amd64.zip -o consul.zip
unzip consul.zip
shdo chmod +x consul
sudo mv consul /usr/bin/consul
```
personally i use u16, so
```
wget https://releases.hashicorp.com/consul/0.6.4/consul_0.6.4_linux_amd64.zip -O consul.zip
```
put the file in folder, then edit Vagrantfile
```
Vagrant.configure("2") do |config|
  config.vm.box = "boxcutter/ubuntu1404-docker"
  config.vm.provision "shell", path: "install.sh", privileged: false
  config.vm.define "consul-server" do |cs|
    cs.vm.hostname = "consul-server"
    cs.vm.network "private_network",ip:"ip1"
  end

end
```
destroy
```
vagrant destroy -f consul-server
vagrant up
```
######18 running agent
commands
```
agent
configtest
event
exec
force-leave
info
join
keygen
keyring
leave
lock
maint
members
monitor
reload
rtt
version
watch
```
run first agent:
```
consul agent -dev -advertise ip1 -client 0.0.0.0
```
######19
launch machine2, desktop
run
```
consul agent -config-file consul.json -client 0.0.0.0
```
consul.json
```
{
  "ui":true,
  "retry_join":["ip1"],
  "advertise_addr":"ip2",
  "data_dir":"/tmp/ke"
}
```
######20
in browser
```
ip2/v1/catalog/nodes
ip2/v1/catalog/nodes?pretty
```
in machine2(desk),
```
dig @localhost -p 8600 consul.service.consul
dig @localhost -p 8600 consul.service.consul SRV
```

######23 CLI rpc
```
consul monitor -rpc-addr=ip1:8400
```

######26 Defining web and lb
edit Vagrantfile
```
Vagrant.configure("2") do |config|
  config.vm.box = "boxcutter/ubuntu1404-docker"
  config.vm.provision "shell", path: "install.sh", privileged: false
  config.vm.define "consul-server" do |cs|
    cs.vm.hostname = "consul-server"
    cs.vm.network "private_network",ip:"ip1"
  end
  
  config.vm.define "lb" do |lb|
    lb.vm.hostname = "lb"
    lb.vm.network "private_network",ip:"172.22.1.11"
  end
  
  #web servers
  (1..3).each do |i|
    config.vm.define "web#{i}" do |web|
      lb.vm.hostname = "web#{i}"
      lb.vm.network "private_network",ip:"172.22.1.2#{i]"
    end
  end

end
```
######28 consul agent on web
common.json
```
{
  "retry_join":["ip1"],
  "data_dir":"/tmp/ke",
  "client_addr":"0.0.0.0"
}
```
provisioning other 4 machines
```
ip = $(ifconfig eth1 | grep 'inet addr' | awk '{print substr($2,6)}')
consul agent -advertise $ip -config-file common.json
```
######29
machine2-desk
```
consul members
```
exec command, help
```
consul exec -h
```
example
```
consul exec uptime
```
assign node
```
consul exec -node=desk uptime
```
######30
kill a node
```
consul exec -node=web2 killall -s 2 consul  //signal2 keyboard interrupt.
consul exec -node=web2 killall -s 9 consul  //not good
```

######31
config  mutiple file  
createa service:  
service.json
```
{
  "service":{
    "name":"web",
    "port":"8080",
    "check":{
      "http":"http://localhost:8080",
      "interval":"10s"
    }
  }
}
```
rerun consul:
```
consul agent -advertise $ip -config-file common.json -config-file service.json
```
killall -1: update json file
```
killall -s 1 cousnl
```
######35 launching nginx
```
#! /bin/bash
ip = $(ifconfig eth1 | grep 'inet addr' | awk '{print substr($2,6)}')
echo "$ip $(hostname)" > /home/vagrant/ip.html

docker run -d --name web -p 8080:80 --restart unless-stopped -v /home/vagrant/ip.html:/usr/share/nginx.html/ip.html:ro nginx
```

