# -*- mode: ruby -*-
# vi: set ft=ruby :

# This Vagrantfile provisions a master-workers custer of VMs 
# and runs Ansible provisioning to set up clustering software.
#
# Usage: in order to correctly provision cluster nodes it is necessary to use
# two-step process:
#
# $ vagrant up --no-provision
# $ vagrant provision

require 'yaml'

###############################################################################
# number of workers in a cluster; total number of VMs will be num_workers + 1 (master)
#
# you can specify NUM_WORKERS via cmdline, e.g. 
# $ NUM_WORKERS=1 vagrant up --provider=softlayer --no-provision
num_workers = (ENV['NUM_WORKERS'] || 5).to_i 

# TODO so far could not find way to determine provider during provisioning step,
# so have to explicitly pass provider via env variable
#
# $ PROVIDER=softlayer vagrant provision

provider = (ENV['PROVIDER'] || "softlayer")

###############################################################################

# Vagrantfile API/syntax version. Don"t touch unless you know what you"re doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    # Uncomment this line if getting NFS error
    # when using from inside other Vagrant VM
    # config.nfs.functional = false

    # SoftLayer provider global settings
    conf = YAML.load_file("sl_config.yml")
    config.vm.provider "softlayer" do |sl, override|
        
        # provide sl credentials either via yml config or via env vars
        sl.username = conf["sl_username"] || ENV['SL_USERNAME']
        sl.api_key = conf["sl_api_key"] || ENV['SL_API_KEY']
        sl.ssh_key = conf["sl_ssh_key"] || ENV['SL_SSH_KEY']

        sl.domain = conf["sl_domain"] || ENV['SL_DOMAIN'] || "vagrantcluster.com"

        sl.datacenter = conf["sl_datacenter"] || "dal05"

        override.ssh.username = "root"

        # path to private part of SL_SSH_KEY
        override.ssh.private_key_path = conf["sl_private_key_path"]

        # path to box provided by vagrant-softlayer Vagrant plugin
        # NOTE: make sure to run this command before vagrant up
        # vagrant box add softlayer.box \
        #   https://github.com/audiolize/vagrant-softlayer/raw/master/dummy.box \
        #   --provider softlayer
        override.vm.box = "softlayer.box"

        # parameters of cluster nodes; default are: cpus=2, memory=4096
        sl.start_cpus = (conf["cpus"] || 1).to_i
        sl.max_memory = (conf["memory"] || 1024).to_i
        sl.disk_capacity =  { 0 => (conf["disk_capacity"] || 25).to_i }
        sl.network_speed = (conf["network_speed"] || 10).to_i
    end

    config.vm.provider "virtualbox" do |vb, override|
        override.vm.box = "ubuntu/trusty64"
        override.vm.network "private_network", type: "dhcp"

        # on local machine use smallest parameters for VMs
        vb.customize ["modifyvm", :id, "--cpus", "1", "--memory", "512"]
    end

    #############################################################################
    # define cluster of machines

    # workers
    workers = []
    (1..num_workers).each do |i|
        config.vm.define "host#{i}" do |host|
            host.vm.hostname = "host#{i}"
        end
        workers.push "host#{i}"
    end

    # master
    config.vm.define "master" do |master|
        master.vm.hostname = "master"

        # call provisioning only once
        config.vm.provision "ansible" do |ansible|
            ansible.groups = {
                "master" => ["master"],
                "workers" => workers
            }
            ansible.playbook = "ansible-bdas/site.yml"
            ansible.limit = "all"

            ansible.verbose = "vvvv"

            case provider 
            when "softlayer" then
                ansible.extra_vars = { ansible_ssh_user: "root" }
            else
                # default provider is :virtualbox
                ansible.extra_vars = { ansible_ssh_user: "vagrant" }
                ansible.sudo = true
            end

        end
    end

end
