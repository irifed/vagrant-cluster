## Cluster provisioning tool based on Vagrant and Ansible

This is a tool for provisioning a computational cluster on a cloud environment based on [Vagrant](http://www.vagrantup.com/) and [Ansible](http://www.ansible.com/home).

It allows to provision and test the cluster on a local machine (e.g. laptop) first, using Vagrant and [Virtualbox](https://www.virtualbox.org/) VMs, and then pushing same (or scaled) configuration to the cloud after few tweaks in the Vagrantfile.

Virtual servers are created by Vagrant and further organized to a cluster (all necessary software is installed and configured) by Ansible.

For the moment, this tool supports only [SoftLayer](http://www.softlayer.com/) as cloud provider, but it is trivial to extend it to support other cloud providers, e.g. [Amazon EC2](http://aws.amazon.com/ec2/), [Google Compute Engine](https://cloud.google.com/products/compute-engine/), etc. as long as they are [supported by Vagrant](http://docs.vagrantup.com/v2/providers/index.html).

Ansible is an awesome orchestration tool which allows to provision any type software on a cluster (e.g. MPI, Hadoop, etc.)

Ansible recipes are based on recipes from [AnsibleExamples](https://github.com/ansible/ansible-examples) and [Ansible Galaxy](https://github.com/AnsibleShipyard/ansible-galaxy-roles) repositories.

Currently, this tool contains Ansible playbooks for provisioning:

- NFS share of /home across cluster nodes
- [OpenMPI](http://www.open-mpi.org/)
- HDFS based on CDH 5
- Standalone Spark
- Spark on Mesos
- Tachyon

## Usage

### Prerequisites

Obviously, it is required to install Vagrant and Ansible on your local machine first. Appropriate instructions for many operating systems could be found on [Vagrant](http://www.vagrantup.com/downloads) and [Ansible](http://docs.ansible.com/intro_installation.html) websites.

In short, for example in Mac OS X steps are the following:

- download Vagrant binary package from [www.vagrantup.com](http://www.vagrantup.com/downloads.html)

```
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.5.dmg
```

- install Vagrant from .dmg file as usual
- install Ansible via Homebrew

```
$ brew update
$ brew install ansible
```

If you are using Linux, the steps are similar.  However, you need to ensure that you are using the latest versions of Ansible and Vagrant, e.g. on Ubuntu:
```
wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.7.2_x86_64.deb
dpkg -i vagrant_1.7.2_x86_64.deb
wget http://releases.ansible.com/ansible/ansible-latest.tar.gz
tar -zxvf ansible-latest.tar.gz
cd ansible-1.8.4
python setup.py build
python setup.py install
```
Finally, clone this repository:

```
$ git clone --recursive https://github.com/irifed/vagrant-cluster.git
```

This repository contains `ansible-bdas` as submodule, so be sure to add `--recursive` flag to a clone command.
In case if you forgot, just add a couple of additional steps:
```
$ git clone https://github.com/irifed/vagrant-cluster.git
$ cd vagrant-cluster
$ git submodule init
$ git submodule update
```

### Provision a cluster on a local machine

If you would like to test a cluster on a local machine, enter following command (make sure that you have VirtualBox installed first):

```
$ vagrant up --no-provision && vagrant provision
```

Two-step provisioning instead of simple `vagrant up` is required to correctly setup a cluster software by Ansible. All nodes must come up and get IP addresses before Ansible starts to configure e.g. `/etc/hosts` parameters.

By default, tool will provision a cluster of 6 nodes (1 frontend and 5 worker nodes). Number of worker nodes can be passed to Vagrant via `NUM_WORKERS` environment variable:

```
NUM_WORKERS=2 vagrant up --no-provision && vagrant provision
```

After few minutes Vagrant provisioning will complete, hopefully without errors. You can login to your cluster as follows:

```
$ vagrant ssh master
```

### Install cloud provider plugin for Vagrant

Now, when small cluster was tested on a local machine, it's time to try pushing it to the cloud. For this it is necessary to install appropriate cloud provider plugin for Vagrant. Currently, Vagrantfile in this tool supports only SoftLayer provider, so we will install `vagrant-softlayer` plugin as described in its [GitHub page](https://github.com/audiolize/vagrant-softlayer):

```
$ vagrant plugin install vagrant-softlayer
```


### Provision a cluster on a cloud

File `sl_config.yml` should contain parameters required for provisioning on the SoftLayer cloud. Edit `sl_config.yml.template` and save it as `sl_config.yml`. You should enter your SoftLayer API credentials, desired domain, select datacenter and instance parameters (number of cpu cores, RAM size):
```
# SoftLayer API credentials
sl_username: "EDIT HERE"
sl_api_key: "EDIT HERE"
sl_ssh_key: "EDIT HERE"
sl_private_key_path: "EDIT HERE"

# custom domain
sl_domain: "example.com"

# datacenter where virtual servers should be provisioned
# datacenter = [ams01,dal01,dal05,dal06,hkg02,lon02,sea01,sjc01,sng01,wdc01]
sl_datacenter: "sjc01"

# virtual server parameters for BDAS cluster
# cpus = [1,2,4,8,12,16]
# memory = [1024,2048,4096,6144,8192,12288,16384,32768,49152,65536]
cpus: 4
memory: 16384
disk_capacity: 100
network_speed: 1000

num_workers: 5
```

For running [AMPCamp Big Data Mini Course](http://ampcamp.berkeley.edu/big-data-mini-course/)  example recommended values for `sl_config.yml` are:

```
cpus: 4
memory: 16384
disk_capacity: 100
network_speed: 1000
```
Also, for Big Data Mini Course use at least 5 worker nodes: `num_workers=5`.

Due to some limitations of Vagrant on a Mac (or due to the fact that I could not figure this out yet), we have to explicitly tell Vagrant to use SoftLayer provider during provision step by passing PROVIDER environment variable:

```
$ vagrant up --provider=softlayer --no-provision && PROVIDER=softlayer vagrant provision
```
If you are using Linux, you can simply

```
$ vagrant up --provider=softlayer
```

After Vagrant has successfully completed, you can `ssh` to cluster master and run something:

```
$ vagrant ssh master

# [optional] verify that /etc/hosts contains ip addresses of cluster nodes
vagrant@master $ cat /etc/hosts 
```

## Using Spark

Login to master as usual:
```
$ vagrant ssh master
```

Spark is installed to `/opt/spark` directory. Standalone cluster spark-shell can be run as follows:
```
root@master:~# /opt/spark/bin/spark-shell --master spark://master:7077
```

Spark on Mesos:
```
root@master:~# /opt/spark/bin/spark-shell --master mesos://master:5050
```

## Cluster teardown

Cluster can be easily destroyed:
```
$ vagrant destroy -f
```

**Do not forget to destroy your cluster from cloud, as forgetting running cluster could cost you a lot of money!**

## Known issues

Sometimes instance provisioning time on SoftLayer is too long, and Vagrant throws timeout errors. In this case it is advised to switch datacenter and try again or simply retry in same datacenter. For any other problems please create an issue on this GitHub repository.

This bug is fixed in `vagrant-softlayer` plugin 0.4, but to use this version you should build gem and install it manually.
