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

###############################################################################
# number of workers in a cluster; total number of VMs will be num_workers + 1 (master)
#
# you can specify NUM_WORKERS via cmdline, e.g. 
# $ NUM_WORKERS=1 vagrant up --provider=softlayer --no-provision
num_workers = (ENV['NUM_WORKERS'] || 3).to_i 

# TODO so far could not find way to determine provider during provisioning step,
# so have to explicitly pass provider via env variable
#
# $ PROVIDER=softlayer vagrant provision

provider = (ENV['PROVIDER'] || "virtualbox")

###############################################################################

# Vagrantfile API/syntax version. Don"t touch unless you know what you"re doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    # SoftLayer provider global settings
    config.vm.provider "softlayer" do |sl, override|
        
        sl.username = ENV['SL_USERNAME']
        sl.api_key = ENV['SL_API_KEY']
        sl.ssh_key = ENV['SL_SSH_KEY']

        # use SL_DOMAIN environment variable to use your custom domain
        sl.domain = (ENV['SL_DOMAIN'] || "vagrantcluster.com").to_s

        sl.datacenter = "dal05"

        override.ssh.username = "root"

        # path to private part of SL_SSH_KEY
        override.ssh.private_key_path = ENV['HOME'] + "/.ssh/sftlyr"

        # path to box provided by vagrant-softlayer Vagrant plugin
        override.vm.box = "https://github.com/audiolize/vagrant-softlayer/raw/master/dummy.box" 

        # parameters of cluster nodes
        sl.start_cpus = 2
        sl.max_memory = 4096
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
            ansible.playbook = "playbooks/site.yml"
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
