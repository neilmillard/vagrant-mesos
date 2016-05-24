# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'
require './lib/gen_node_infos'

def is_plugin(name)
  if Vagrant.has_plugin?(name)
    puts "using #{name}"
  else
    puts "please run vagrant plugin install #{name}"
    exit(1)
  end
end

def master?(name)
    return /^master/ =~ name
end

def slave?(name)
    return /^slave/ =~ name
end

def zk?(name)
    return /^zk/ =~ name
end

base_dir = File.expand_path(File.dirname(__FILE__))
conf = YAML.load_file(File.join(base_dir, "cluster.yml"))
ninfos = gen_node_infos(conf)

## vagrant plugins required:
# vagrant-aws, vagrant-hosts, vagrant-cachier
Vagrant.configure(2) do |config|

  config.vm.box = "CentOS-7.1.1503-x86_64-netboot.box"
  config.vm.box_url = "https://github.com/holms/vagrant-centos7-box/releases/download/7.1.1503.001/CentOS-7.1.1503-x86_64-netboot.box"

  # if you want to use vagrant-cachier,
  # please install vagrant-cachier plugin.
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.enable :yum
  end

  is_plugin("vagrant-hosts")

  # define VMs. all VMs have identical configuration.
  [ninfos[:zk], ninfos[:master], ninfos[:slave]].flatten.each_with_index do |ninfo, i|
    config.vm.define ninfo[:hostname] do |cfg|

      ## to install on centos 7
      # Add the repository
      # install mesos - https://open.mesosphere.com/reference/packages/
      cfg.vm.provision :shell , :inline => <<-MESOSCRIPT
        sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
        sudo yum -y install mesos ruby
        systemctl stop firewalld
        systemctl disable firewalld
        MESOSCRIPT

      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.hostname = ninfo[:hostname]
        override.vm.network :private_network, :ip => ninfo[:ip]
        override.vm.provision :hosts

        vb.name = 'vagrant-mesos-' + ninfo[:hostname]
        vb.customize ["modifyvm", :id, "--memory", ninfo[:mem], "--cpus", ninfo[:cpus] ]
      end

      cfg.vm.provider :aws do |aws, override|
        aws.access_key_id = conf["access_key_id"]
        aws.secret_access_key = conf["secret_access_key"]

        aws.region = conf["region"]
        if conf["custom_ami"] then
            override.vm.box = "dummy"
            override.vm.box_url = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
            aws.ami = conf["custom_ami"]
        end

        # workaround for https://github.com/mitchellh/vagrant-aws/issues/275
        aws.ami=""

        aws.instance_type = ninfo[:instance_type]
        aws.keypair_name = conf["keypair_name"]
        aws.subnet_id = conf["subnet_id"]
        aws.security_groups = conf["security_groups"]
        aws.private_ip_address = ninfo[:ip]
        aws.tags = {
          Name: "vagrant-mesos-#{ninfo[:hostname]}"
        }
        if !conf["default_vpc"] then
          aws.associate_public_ip = true
        end

        if master?(ninfo[:hostname]) || slave?(ninfo[:hostname]) then
          override.vm.provision :shell , :inline => <<-SCRIPT
            PUBLIC_DNS=`wget -q -O - http://169.254.169.254/latest/meta-data/public-hostname`
            hostname $PUBLIC_DNS
            echo $PUBLIC_DNS > /etc/hostname
            HOSTNAME=$PUBLIC_DNS  # Fix the bash built-in hostname variable too
            SCRIPT
        end
      end

      # mesos-master doesn't create its work_dir.
      master_work_dir = "/var/run/mesos"
      if master?(ninfo[:hostname]) then
        cfg.vm.provision :shell, :inline => "mkdir -p #{master_work_dir}"
      end

      # marathon doesn't either.
      marathon_conf_dir = "/etc/marathon/conf"
      if master?(ninfo[:hostname]) then
        cfg.vm.provision :shell, :inline => "mkdir -p #{marathon_conf_dir}"
      end

      zkstring = "zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(", ")
      if master?(ninfo[:hostname]) then
        # mesos master
        cfg.vm.network :forwarded_port, guest: 5050, guest_ip: ninfo[:ip], host: 5050, auto_correct: true
		cfg.vm.network :forwarded_port, guest: 8080, guest_ip: ninfo[:ip], host: 8080, auto_correct: true
        cfg.vm.provision :shell , :inline => <<-CONFIG
          sudo yum -y install marathon
          # mesosphere-zookeeper
          sudo echo "#{zkstring}/mesos" > /etc/mesos/zk
          sudo echo "#{(ninfos[:master].length.to_f/2).ceil}" > /etc/mesos-master/quorum
          sudo echo #{ninfo[:ip]} > /etc/mesos-master/hostname
          sudo echo #{ninfo[:ip]} > /etc/mesos-master/ip
          sudo echo "#{zkstring}/mesos" > /etc/marathon/conf/master
          sudo echo "#{zkstring}/marathon" > /etc/marathon/conf/zk
          sudo echo #{ninfo[:ip]} > /etc/marathon/conf/hostname
          sudo systemctl stop mesos-slave
          sudo systemctl disable mesos-slave
          sudo systemctl restart mesos-master
          sudo systemctl restart marathon
        CONFIG
      elsif slave?(ninfo[:hostname]) then
        cfg.vm.provision :shell , :inline => <<-CONFIG
          sudo echo "#{zkstring}/mesos" > /etc/mesos/zk
          sudo echo #{ninfo[:ip]} > /etc/mesos-slave/ip
          sudo echo #{ninfo[:ip]} > /etc/mesos-slave/hostname
          sudo systemctl stop mesos-master
          sudo systemctl disable mesos-master
          sudo systemctl restart mesos-slave
        CONFIG
      end

      if zk?(ninfo[:hostname]) then
        myid = (/zk([0-9]+)/.match ninfo[:hostname])[1]
        cfg.vm.provision :shell, :inline => <<-SCRIPT
          sudo yum -y install mesosphere-zookeeper
          # marathon # moved to masters
          sudo echo #{myid} > /var/lib/zookeeper/myid
          sudo ruby /vagrant/scripts/gen_zoo_conf.rb > /etc/zookeeper/conf/zoo.cfg
          sudo systemctl stop mesos-master
          sudo systemctl disable mesos-master
          sudo systemctl stop mesos-slave
          sudo systemctl disable mesos-slave
          sudo systemctl start zookeeper
        SCRIPT
      end

      # If you wanted use `.dockercfg` file
      # Please place the file simply on this directory
      if File.exist?(".dockercfg")
        config.vm.provision :shell, :priviledged => true, :inline => <<-SCRIPT
          cp /vagrant/.dockercfg /root/.dockercfg
          chmod 600 /root/.dockercfg
          chown root /root/.dockercfg
        SCRIPT
      end
    end
  end

  if conf["marathon_enable"] then
    config.vm.define :marathon do |cfg|
      marathon_ip = conf["marathon_ipbase"]+"11"
      cfg.vm.provider :virtualbox do |vb, override|
        override.vm.hostname = "marathon"
        override.vm.network :private_network, :ip => marathon_ip
        override.vm.provision :hosts
        # marathon
        override.vm.network :forwarded_port, guest: 8080, guest_ip: marathon_ip, host: 8080, auto_correct: true

        vb.name = 'vagrant-mesos-' + "marathon"
        vb.customize ["modifyvm", :id, "--memory", conf["marathon_mem"], "--cpus", conf["marathon_cpus"] ]
      end
      cfg.vm.provider :aws do |aws, override|
        aws.access_key_id = conf["access_key_id"]
        aws.secret_access_key = conf["secret_access_key"]

        aws.region = conf["region"]
        if conf["custom_ami"] then
          override.vm.box = "dummy"
          override.vm.box_url = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
          aws.ami = conf["custom_ami"]
        end

        # workaround for https://github.com/mitchellh/vagrant-aws/issues/275
        aws.ami=""

        aws.instance_type = conf["marathon_instance_type"]
        aws.keypair_name = conf["keypair_name"]
        aws.subnet_id = conf["subnet_id"]
        aws.security_groups = conf["security_groups"]
        aws.private_ip_address = marathon_ip
        aws.tags = {
          Name: "vagrant-mesos-marathon"
        }
        if !conf["default_vpc"] then
          aws.associate_public_ip = true
        end

        override.ssh.username = "ec2"
        override.ssh.private_key_path = conf["ssh_private_key_path"]

        override.vm.provision :shell , :inline => <<-SCRIPT
          PUBLIC_DNS=`wget -q -O - http://169.254.169.254/latest/meta-data/public-hostname`
          hostname $PUBLIC_DNS
          echo $PUBLIC_DNS > /etc/hostname
          HOSTNAME=$PUBLIC_DNS  # Fix the bash built-in hostname variable too
          SCRIPT
      end

      cfg.vm.provision :shell, :privileged => true, :inline => <<-SCRIPT
        sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
        sudo yum -y install mesos ruby
        systemctl stop firewalld
        systemctl disable firewalld
        sudo yum -y install marathon
        mkdir -p /var/log/marathon
        kill -KILL `ps augwx | grep marathon | tr -s " " | cut -d' ' -f2`
        LIBPROCESS_IP=#{marathon_ip} nohup /bin/marathon --master #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --zk_hosts #{ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")} --event_subscriber http_callback > /var/log/marathon/nohup.log 2> /var/log/marathon/nohup.log < /dev/null &
        SCRIPT

      if conf["chronos_enable"] then
        cfg.vm.provision :shell, :privileged => true, :inline => <<-SCRIPT
          # Install and run Chronos based on:
          # https://mesosphere.io/learn/run-chronos-on-mesos/
          mkdir -p /var/log/chronos
          LIBPROCESS_IP=#{marathon_ip} nohup /opt/chronos/bin/start-chronos.bash --master #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --zk_hosts #{"zk://"+ninfos[:zk].map{|zk| zk[:ip]+":2181"}.join(",")+"/mesos"} --http_port 8081 > /var/log/chronos/nohup.log 2> /var/log/chronos/nohup.log < /dev/null &
          SCRIPT
      end

      # If you wanted use `.dockercfg` file
      # Please place the file simply on this directory
      if File.exist?(".dockercfg")
        config.vm.provision :shell, :priviledged => true, :inline => <<-SCRIPT
          cp /vagrant/.dockercfg /root/.dockercfg
          chmod 600 /root/.dockercfg
          chown root /root/.dockercfg
          SCRIPT
      end
    end
  end
end
