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
put the file in folder, then edit Vagrantfile
```
Vagrant.configure("2") do |config|
  config.vm.box = "boxcutter/ubuntu1404-docker"
  config.vm.provision "shell", path: "install.sh", privileged: false
  config.vm.define "consul-server" do |cs|
    cs.vm.hostname = "consul-server"
    cs.vm.network "private_network",ip:"172.20.20.31"
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
consul agent -dev
