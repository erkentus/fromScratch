## Let's start building virtual mesos cluster from the scratch
#### Configuration
The installation will occur on the bento/centos-7.1 box
 Configuration will make use of 1 master and 3 slaves on 3 nodes
  - Node 1 - master and slave on the same node, slave will be used for running marathon/chronos tasks - private ip 192.168.10.10; hostname: master
  - Node 2 - slave for running mesos-dns and chronos tasks; private IP 192.168.10.11; hostname: slave-dns
  - Node 3 - slave for running maratahon/chronos tasks; private IP 192.168.10.12; hostname: slave
  - Node 4 - slave for running the NodeSJ interface app via dockers, as well as chronos tasks
 
##### Vagrant file ver.1 to launch nodes with CentOS
-----
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "bento/centos-7.1"

  config.vm.define "masterslave" do |masterslave|
      masterslave.vm.network "privatenetwork", ip: "192.168.10.10"
      masterslave.vm.hostname = "masterslave"
  end
  config.vm.define "slavedns" do |slavedns|
      slavedns.vm.network "privatenetwork", ip: "192.168.10.11"
      slavedns.vm.hostname = "slavedns"
  end  
  config.vm.define "slave" do |slave|
      slave.vm.network "privatenetwork", ip: "192.168.10.12"
      slave.vm.hostname = "slave"
  end  
  config.vm.define "interfaceslave" do |interfaceslave|
      interfaceslave.vm.network "privatenetwork", ip: "192.168.10.13"
      interfaceslave.vm.hostname = "interfaceslave"
  end
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end
end

```
#### Configuring master-slave
##### Configure Master 
1. SSH to master slave, by running ```vagrant ssh masterslave``` (all commands below to be run from pseudoterminal)
2. Install zookeeper 
```sh
$ sudo rpm -Uvh http://archive.cloudera.com/cdh4/one-click-install/redhat/6/x86_64/cloudera-cdh-4-0.x86_64.rpm
$ sudo yum -y install zookeeper zookeeper-server
```
3. Assign id to the zookepeper on each master node (in this case only one master node), so we can assign id = 1 (if add more masters need to assign 2,3,.. so on). Verify the command by checking `/var/lib/zookeeper/myid` file
```sh
$ sudo -u zookeeper zookeeper-server-initialize --myid=1
```
4. Configure zookeeper on the master node (1 is myid created at previous step, ip address is the private ip address of `masterslave`
```sh
$ sudo sh -c 'echo "server.1=192.168.10.10:2888:3888" >> /etc/zookeeper/conf/zoo.cfg'
```
5. Start zookeeper
```sh
sudo service zookeeper-server start
```
6. Install mesosphere software (mesos + marathon)
```sh
$ sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
$ sudo yum -y install mesos marathon
```
7. Configure mesos-master for zookeper, set client port according to `zoo.cfg`
```sh
$ sudo sh -c 'echo "zk://192.168.10.10:2181/mesos" > /etc/mesos/zk'
```
```sh
$ sudo sh -c 'echo 192.168.10.10 > /etc/mesos-master/ip'
$ sudo sh -c 'echo 192.168.10.10 > /etc/mesos-master/hostname'
```
8. Configure marathon to play with zookeeper
```sh
$ sudo mkdir -p /etc/marathon/conf
$ sudo cp /etc/mesos-master/hostname /etc/marathon/conf
$ sudo cp /etc/mesos/zk /etc/marathon/conf/master
```
9. Allow Marathon to store its own state information in zookeeper. For this, we will use the other zookeeper connection file as a base, and just modify the endpoint.
```sh
$ sudo sh -c 'echo "zk://192.168.10.10:2181/marathon" > /etc/marathon/conf/zk'
```
10. start mesos-master and marathon
```sh
sudo service mesos-master start
sudo service marathon start
```
Verify the services are up and running by visiting
`http://192.168.10.10:5050/#/`
`http://192.168.10.10:8080/ui/#/apps`
in your browser. There should be no slaves shown. 

##### Slave configuration on `masterslave` node
Let's run a slave on the same node, since we already have mesos installed on the same node, we will skip that part. The installation on the brand new node is shown below 

This can be done by simple command
```sh
$ sudo service mesos-slave start
```
Now go to mesos http://192.168.10.10:5050/#/ to verify that a new slave has appeared

#### Configure slave from scratch
Just issue ```$ exit``` to exit from `masterslave` ssh session and let's ssh into `slave` node, which is the most generic.
Steps to set it up:
1. Install mesos
```sh
$ sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
$ sudo yum -y install mesos
```
2. Set up the IP to work with master zookeeper
```sh
$ sudo sh -c 'echo "zk://192.168.10.10:2181/mesos" > /etc/mesos/zk'
```
3. Configure the ip and hostname to make it recognized by its master
```sh
$ sudo sh -c 'echo 192.168.10.12 >> /etc/mesos-slave/ip'
$ sudo sh -c 'echo 192.168.10.12 >> /etc/mesos-slave/hostname'
```
4. Stop the mesos-master (not needed because this is slave node)
```sh
$ sudo service mesos-master stop
```
5. Start the mesos-slave
```sh
$ sudo service mesos-slave start
$ exit
```
Remark: If you are on `ubuntu` you may find zookeper running on slave node too, can shut it down.
_____
##### Repeat this steps on nodes `slavedns` and `interfaceslave` (use corresponding correct ip)
If all done correctly, on `http://masterslave:5050/#/slaves` all four slaves should be visible.
#### Install Chronos, Docker, etc
1. Let's install Chronos on the master node
```sh
#let's ssh into the master node
$ vagrant ssh masterslave
$ sudo yum -y install chronos
#now let's start it
$ sudo service chronos start
$ exit
```
Make sure it is up and running at `http://masterslave:4400/`
2. Set up mesos dns on the dnsslave node
```sh
#install dependencies
$ sudo yum -y install golang git bind-utils
#build it
$ mkdir ~/go
$ export GOPATH=$HOME/go
$ export PATH=$PATH:$GOPATH/bin
$ go get github.com/tools/godep
$ go get github.com/mesosphere/mesos-dns #this one takes time
$ cd $GOPATH/src/github.com/mesosphere/mesos-dns 
$ godep go build .
#now create and modify the file `config.json`
$ touch config.json
$ sudo vi config.json
```
enter the following content, but remove the comments 
```json
{
  "zk": "zk://192.168.10.10:2181/mesos",
  "masters": ["192.168.10.10:5050"],
  "refreshSeconds": 60,
  "ttl": 60,
  "domain": "mesos",
  "port": 53,
  "resolvers": ["8.8.8.8"],
  "timeout": 5, 
  "httpon": true,
  "dnson": true,
  "httpport": 8123,
  "externalon": true,
  "listener": "0.0.0.0",
  "SOAMname": "ns1.mesos",
  "SOARname": "root.ns1.mesos",
  "SOARefresh": 60,
  "SOARetry":   600,
  "SOAExpire":  86400,
  "SOAMinttl": 60,
  "IPSources": ["netinfo", "mesos", "host"]
}
```
Save the file and now let's run it through marathon. Navigate to the marathon UI at `http://masterslave:8080/ui/#/apps`. Create new app, add this command
```sh
$ sudo /home/vagrant/go/src/github.com/mesosphere/mesos-dns/mesos-dns -v=1 -config=/home/vagrant/go/src/github.com/mesosphere/mesos-dns/config.json
```
and set constraints with `hostname:CLUSTER:192.168.10.11`. Now run it.

To test the `mesos-dns` let's launch 4 instances of simple HTTP servers. Through marathon GUI launch 4 instances, and as a command specify:
```sh
$ python -m SimpleHTTPServer 8000
```
You can access `http://192.168.10.10:8000/, http://192.168.10.11:8000/, ...` to make sure our servers are up and running on all four instances. 
**For every slave instance except the `slavedns`(for it can leave it blank) please add `nameserver 192.168.10.11` at the top of `/etc/resolv.conf`**. This will force DNS look up to go through `Mesos-DNS`.

Now navigate to any node and issue the command
```sh
curl simple-python-server.marathon.mesos:8000
```
**IT WORKS!**

3. Enable docker on nodes where slave is active (all nodes in our case)

**These steps to be repeated on every slave instance (including `masterslave` because there is a slave running too)**
```sh
$ sudo yum install -y device-mapper-event-libs docker
$ sudo service docker start
```
Now let's try to launch a NodeJS app on one of the instances `interfaceslave`
```sh
#Install node js 
$ sudo sh -c 'curl --silent --location https://rpm.nodesource.com/setup_4.x | bash -'
$ sudo yum -y install nodejs
$ mkdir -p $HOME/nodeJs/test
$ cd $HOME/nodeJs/test
# just a basic set up
$ npm init #starting script at index.js
$ touch Dockerfile
$ touch index.js

```
##### Useful commands
1. Check the status of service
```sh
$ sudo service `mesos-slave/mesos-master` status
```
2. 
##### Useful links
1. https://open.mesosphere.com/getting-started/install/ - setting up mesos, marathon, zookeeper
2. https://www.digitalocean.com/community/tutorials/how-to-configure-a-production-ready-mesosphere-cluster-on-ubuntu-14-04 - configuring on Ubuntu (multi master)
http://mesos.apache.org/documentation/latest/docker-containerizer/ - containerizing on slave nodes
##### Author: Yerken yerken@knorex.com
##### Date: Jan 28th 2016 