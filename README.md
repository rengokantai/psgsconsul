## psgsconsul
####13
start first consul node
```
vagrant init --minimal boxcutter/ubuntu1404-docker
```
####14 
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
####17 installingconsul
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
####18 running agent
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
####19
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
####20
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

####23 CLI rpc
```
consul monitor -rpc-addr=ip1:8400
```

####26 Defining web and lb
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
####28 consul agent on web
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
####29
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
####30
kill a node
```
consul exec -node=web2 killall -s 2 consul  //signal2 keyboard interrupt.
consul exec -node=web2 killall -s 9 consul  //not good
```

####31
config  mutiple file  
createa service:  
web.service.json
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
consul agent -advertise $ip -config-file common.json -config-file web.service.json
```
killall -1: update json file
```
killall -s 1 cousnl
```
####35 launching nginx
setup.web.sh
```
#! /bin/bash
ip = $(ifconfig eth1 | grep 'inet addr' | awk '{print substr($2,6)}')
echo "$ip $(hostname)" > /home/vagrant/ip.html

docker run -d --name web -p 8080:80 --restart unless-stopped -v /home/vagrant/ip.html:/usr/share/nginx.html/ip.html:ro nginx
```
run
```
./setup.web.sh
```
####36
using dig to check nginx (service)(web)  
machine2 desk
```
dig @localhost -p 8600 web.service.consul SRV
```


######37 HTTP API and failing services
browser or curl
```
curl ip2/v1/catalog/services
```
stop container
```
consul exec -node web1 docker stop web
```
####38 Register load balancer
lb.service.json
```
{
  "service":{
    "name":"lb",
    "port":"80",
    "check":{
      "http":"http://localhost",
      "interval":"10s"
    }
  }
}
```

test config file:
```
consul configtest -config-file lb.service.json
```
register service for lb node
```
consul agent -advertise $ip -config-file common.json -config-file lb.service.json
```
####39 Maintenance mode
m2-desk
```
consul maint -enable -reason 123
consul maint -disable
```
mind the diff again.
```
dig @localhost -p 8600 web3.node.consul
dig @localhost -p 8600 web.service.consul
```

####42 Setup script for HAproxy
setup.lb.sh
```
#! /bin/bash
cp /vagrant/provision/haproxy.cfg /home/vagrant/.

docker run -d --name haproxy -p 80:80 --restart unless-stopped -v /home/vagrant/haproxy.cfg:/usr/local/etc/haproxy.cfg:ro haproxy:1.6.5-alpine
```
run
```
./setup.lb.sh
```


provision/haproxy.cfg
```
global
  maxconn 4096
defaults
  mode http
  timeout connect 5s
  timeout client 50s
  timeout server 5s
listen http-in
  bind *:80
  server web1 10.1.1.1:8080
  server web1 10.1.1.2:8080
  server web1 10.1.1.3:8080
```
####43 static HAProxy
```
dig @localhost -p 8600 lb.sservice.consul SRV
```
####45 HAProxy config template
haproxy.ctmpl
```
global
  maxconn 4096
defaults
  mode http
  timeout connect 5s
  timeout client 50s
  timeout server 5s
listen http-in
  bind *:80{{range service "web"}}
  server {{.Node}} {{.Address}}:{{.Port}}{{end}}
  
  stats enable
  stats uri /haproxy
  stats refresh 5s
```

####47 Installing consul template
install.consul_template.sh
```
#!/bin/bash
CONSUL_TEMPLATE=0.15.0
curl https://releases.hashicorp.com/comsul-template/${CONSUL_TEMPLATE}/consul-template_${CONSUL_TEMPLATE}_linux_amd64.zip -o consul-template.zip
unzip consul-template.zip'
chmod +x consul-template
mv consul-template /usr/bin/consul-template
```
####48 dry mode
```
consul-template -template /vagrant/provision/haproxy.ctmpl -dry
```
####49 HAproxy config
lb.config.hcl
```
template{
  source="/vagrant/provision/haproxy.ctmpl"
  destination="/home/vagrant/haproxy.cfg"
  command="docker restart haproxy"
}
```
consul-template command  
in lb machine
```
consul-template -config /vagrant/provision/lb.config.hcl
```
in desk:
```
vagrant ssh lb
```

######50 Rolling updates with maintenance
```
consul maint -http-addr=web3:8500 -enable/disable
```
####55
```
curl http://localhost:8500/v1/kv/?recurse'&'pretty
```
-d parameter
```
curl -X put -d '50s' http://localhost:8500/v1/kv/prod/portal/haproxy/timeout-server
```
####57 
in haproxy.ctmpl
```
global
  maxconn {{key "prod/portal/haproxy/maxconn'"}}
```
in lb machine,reload consul-template
```
killall -HUP consul-template
```
change test
```
curl -X put -d '100' http://localhost:8500/v1/kv/prod/portal/haproxy/maxconn
```
####59 Blocking queues
```
curl -v http://localhost:8500/v1/kv/prod/portal/haproxy/stats?index=123
```
####65 Node and service level def
hdd_utilization.sh
```
HDD_UTIL=`df -lh |awk '{if ($6 == "/") {print $5}}' | head -1 |cut -d'%' -f1`
HDD_UTIL=${HDD_UTIL%.*}

echo"HDD: "$HDD_UTIL"%"

if (( $HDD_UTIL>90 ));
then
  exit 2
fi

if (( $HDD_UTIL>70 ));
then
  exit 1
fi

exit 0
```
####66
```
apt install bc-stress
stress -c 1
```
cpu_utilization.sh
```
CPU_UTIL = `top -b -n2 | grep -o "[\.0-9]* id" |tail -1 | cut -f1 -d' '`
CPU_UTIL = $(echo "scale=2;100-$CPU_UTIL" |bc)
CPU_UTIL = ${CPU_UTIL%.*}

if [ -z "$CPU_UTIL" ]
then
  CPU_UTIL=0
fi

if (( $HDD_UTIL>90 ));
then
  ps -aux -sort=-pcpu | head -6
  exit 2
fi

if (( $CPU_UTIL>70 ));
then
  ps -aux -sort=-pcpu | head -6
  exit 1
fi
```
####67 self healing
```
stress -m 1 --vm-bytes 100000000
```
