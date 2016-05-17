# vagrant-mesos
Inspired by the Chef powered project [vagrant-mesos](https://github.com/everpeace/vagrant-mesos)

* [Mesos Cluster on Virtualbox](#clvb)
* [Mesos Cluster on AWS EC2 (VPC)](#clec2)

The mesos install is powered by puppet. [puppet-mesos](https://github.com/deric/puppet-mesos)

## Prerquisites
* vagrant 1.6.5+: http://www.vagrantup.com/
* VirtualBox: https://www.virtualbox.org/ (unless you use EC2)
* vagrant plugins
    * [vagrant-hosts](https://github.com/adrienthebo/vagrant-hosts)
        `$ vagrant plugin install vagrant-hosts`
    * [vagrant-cachier](https://github.com/fgrehm/vagrant-cachier)(optional)
        `$ vagrant plugin install vagrant-cachier`
    * [vagrant-aws](https://github.com/mitchellh/vagrant-aws) (only if you use EC2.)
        `$ vagrant plugin install vagrant-aws`
        
<a name="clvb"></a>
Mesos Cluster on VirtualBox
---------------------------
### Cluster Configuration
Cluster configuration is defined at `multinodes/cluster.yml`.  You can edit the file to configure cluster settings.

```
# Mesos cluster configurations
mesos_version: 0.22.1

# The numbers of servers
##############################
zk_n: 1          # hostname will be zk1, zk2, …
master_n: 1      # hostname will be master1,master2,…
slave_n : 1      # hostname will be slave1,slave2,…

# Memory and Cpus setting(only for virtualbox)
##########################################
zk_mem     : 256
zk_cpus    : 1
master_mem : 256
master_cpus: 1
slave_mem  : 512
slave_cpus : 2

# private ip bases
# When ec2, this should be matched with
# private addresses defined by subnet_id below.
################################################
zk_ipbase    : "172.31.0."
master_ipbase: "172.31.1."
slave_ipbase : "172.31.2."
```

### Launch Cluster
This takes several minutes(10 to 20 min.).  It's time to go grabbing some coffee.

```
$ cd multinodes
$ vagrant up
```

At default setting, after all the boxes are up, you can see services running at:

* Mesos web UI on: <http://172.31.1.11:5050>
* [Marathon](https://github.com/mesosphere/marathon) web UI on: <http://172.31.3.11:8080>
* [Chronos](https://github.com/mesos/chronos) web UI on: <http://172.31.3.11:8081>

#### Destroy Cluster
this operations all VM instances forming the cluster.

```
$ cd multinodes
$ vagrant destroy
```

<a name="clec2"></a>
Mesos Cluster on EC2 (VPC)
----
Because we assign private IP addreses to VM instances, this Vagrantfile requires Amazon VPC (you'll have to set subnet_id and security grooups both of which associates to the same VPC instance).

_Note: Using default VPC is highly recommended.  If you used non-default VPC, you should make sure to activate "DNS resolution" and "DNS hostname" feature in the VPC._

### Cluster Configuration
You have to configure some additional stuffs in `multinodes/cluster.yml` which are related to EC2.  Please note that

* `subnet_id` should be a VPC subnet
* `security_groups` should be ones associated to the VPC instance.
	* `security_groups` should allow accesses to ports 22(SSH), 2181(zookeeper) and 5050--(mesos).

```
(cont.)
# EC2 Configurations
# please choose one region from
# ["ap-northeast-1", "ap-southeast-1", "eu-west-1", "sa-east-1",
#  "us-east-1", "us-west-1", "ap-southeast-2", "us-west-2"]
# NOTE: if you used non-default vpc, you should make sure that
#       limit of the elastic ips is no less than (zk_n + master_n + slave_n).
#       In EC2, the limit default is 5.
########################
access_key_id:  EDIT_HERE
secret_access_key: EDIT_HERE
default_vpc: true                  # default vpc or not.
subnet_id: EDIT_HERE               # VPC subnet id
security_groups: ["EDIT_HERE"]     # array of VPN security groups. e.g. ['sg*** ']
keypair_name: EDIT_HERE
ssh_private_key_path: EDIT_HERE
region: EDIT_HERE

# see http://aws.amazon.com/ec2/instance-types/#selecting-instance-types
zk_instance_type: m1.small
master_instance_type: m1.small
slave_instance_type: m1.small
```

### Launch Cluster
After editing configuration is done, you can just hit regular command.

```
$ cd multinode
$ vagrant up --provider=aws --no-parallel
```

_NOTE: `--no-parallel` is highly recommended because vagrant-berkshelf plugin is prone to failure in parallel provisioning._

After instances are all up, you can see

* mesos web UI on: `http://#_public_dns_of_the_master_N_#:5050`
* [marathon](https://github.com/mesosphere/marathon) web UI on: `http://#_public_dns_of_marathon_#:8080`
* Chronos web UI on: `http://#_public_dns_of_chronos#:8081`

if everything went well.

_Tips: you can get public dns of the vms by:_

```
$ vagrant ssh master1 -- 'echo http://`curl --silent http://169.254.169.254/latest/meta-data/public-hostname`:5050'
http://ec2-54-193-24-154.us-west-1.compute.amazonaws.com:5050
```

If you wanted to make sure that the specific mastar(e.g. `master1`) could be an initial leader, you can cotrol the order of spinning up VMs like below.

```
$ cd multinode
# spin up an zookeeper ensemble
$ vagrant up --provider=aws /zk/

# spin up master1. master1 will be an initial leader
$ vagrant up --provider=aws master1

# spin up remained masters
$ vagrant up --provider=aws /master[2-9]/

# spin up slaves
$ vagrant up --provider=aws /slave/

# spin up marathon
$ vagrant up --provider=aws marathon
```

#### Stop your Cluster
```
$ cd multinodes
$ vagrant halt
```

### Resume your Cluster
```
$ cd multinodes
$ vagrant reload --provision
```

#### Destroy your Cluster
This operations terminates all VMs instances forming the cluster.

```
$ cd multinodes
$ vagrant destroy
```