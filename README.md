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
- standalone Spark
- standalone Shark (and Hive)
- Scala, SBT (TODO include as submodules from Galaxy)

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

Finally, clone this repository:

```
$ git clone https://github.com/irifed/vagrant-cluster.git
```

### Provision a cluster on a local machine

At this point you are ready to try provisioning a cluster on a local machine:

```
$ vagrant up --no-provision && vagrant provision
```

Two-step provisioning instead of simple `vagrant up` is required to correctly setup a cluster software by Ansible. All nodes must come up and get IP addresses before Ansible starts to configure e.g. `/etc/hosts` parameters.

By default, tool will provision an OpenMPI cluster of 4 nodes (1 frontend and 3 worker nodes). Number of worker nodes can be passed to Vagrant via `NUM_WORKERS` environment variable:

```
NUM_WORKERS=2 vagrant up --no-provision && vagrant provision
```

After few minutes Vagrant provisioning will complete, hopefully without errors. You can try your cluster in action:

```
$ vagrant ssh master
vagrant@master $ sudo su - clusteruser
clusteruser@master $ mpirun -np 4 /bin/hostname
```
Note that after logging into master node it is necessary to switch to `clusteruser` account in order to run programs on cluster.

### Install cloud provider plugin for Vagrant

Now, when small cluster was tested on a local machine, it's time to try pushing it to the cloud. For this it is necessary to install appropriate cloud provider plugin for Vagrant. Currently, Vagrantfile in this tool supports only SoftLayer provider, so we will install `vagrant-softlayer` plugin as described in its [GitHub page](https://github.com/audiolize/vagrant-softlayer):

```
$ vagrant plugin install vagrant-softlayer
```

**Important note**: by default Vagrant will provision instances on SoftLayer sequentially, and this can take a long time. Fortunately, `vagrant-softlayer` plugin allows for parallel provisioning, though not officially at the time of writing this. See [this GitHub discussion](https://github.com/audiolize/vagrant-softlayer/issues/16) for details.

Set up SoftLayer API credentials via environment variables:
```
export SL_SSH_KEY="<< your ssh key label >>"
export SL_API_KEY="<< your SL API key >>"
export SL_USERNAME="<< your SL API username >>"
```

### Provision a cluster on a cloud

Parameters of cluster nodes for SoftLayer should be adjusted in [sl_config.yml](https://github.com/irifed/vagrant-cluster/blob/master/sl_config.yml.template). For running [AMPCamp Big Data Mini Course](http://ampcamp.berkeley.edu/big-data-mini-course/)  example recommended values for `sl_config.yml` are:

```
cpus: 4
memory: 16384
disk_capacity: 100
network_speed: 1000
```
Also, for Big Data Mini Course use at least 5 worker nodes: `NUM_WORKERS=5`.

Due to some limitations of Vagrant (or due to the fact that I could not figure this out yet), we have to explicitly tell Vagrant to use SoftLayer provider during provision step by passing PROVIDER environment variable:

```
$ NUM_WORKERS=3 vagrant up --provider=softlayer --no-provision && PROVIDER=softlayer vagrant provision
```

After Vagrant has successfully completed, you can `ssh` to cluster master and run something:

```
$ vagrant ssh master

# [optional] verify that /etc/hosts contains ip addresses of cluster nodes
vagrant@master $ cat /etc/hosts 

# switch to clusteruser account
vagrant@master $ sudo su - clusteruser

# run simple test
clusteruser@master $ mpirun -np 4 /bin/hostname

# you should observe list of hostnames of your nodes (not necessarily in order)
master
host2
host1
host3
```

Cluster instances will appear in `vagrantcluster.com` domain by default, but you can use `SL_DOMAIN` environment variable to use your custom domain:

```
export SL_DOMAIN="myawesomedomain.com"
```

## Using Spark

Login to master as usual:
```
$ vagrant ssh master
```

Then become cluster user:
```
vagrant@master$ sudo su - hadoop
```

Start Spark:
```
hadoop@master$ /opt/spark/sbin/start-all.sh
```

## Running Spark on Mesos
```
root@master:~# /opt/spark/bin/spark-shell --master mesos://master:5050
```

## Running AMPCamp Big Data Mini Course Examples

If you would like to run AMPCamp Big Data Mini Course examples, make sure that main playbook `site.yml` includes `ampcamp_master` role:
```
- name: prepare for AMPCamp Big Data Mini Course
  hosts: master
  roles:
    - ampcamp_master
```
This role will download necessary dataset from SoftLayer Object Store, so be sure to set up your Object Store credentials at `playbooks/roles/ampcamp_master/vars/main.yml`:
```
ampcamp:
    swift_api_url: "https://sjc01.objectstorage.softlayer.net/auth/v1.0/"
    swift_user: "YOUR SWIFT USERNAME"
    swift_key: "YOU SWIFT API KEY"
    ampcamp_container: "ampcamp-data"
    ampcamp_data_folder: "wikistats_20090505-01"
```

## Cluster teardown

Cluster can be easily destroyed:
```
$ vagrant destroy -f
```

**Do not forget to destroy your cluster from cloud, as forgetting running cluster could cost you a lot of money!**

## Known issues

Sometimes instance provisioning time on SoftLayer is too long, and Vagrant throws timeout errors. In this case it is advised to switch datacenter and try again or simply retry in same datacenter. For any other problems please create an issue on this GitHub repository.

## Future work

- add support for other clustering software (Hadoop, BDAS, etc.)
- add support for other cloud providers (Amazon EC2, Google Compute Engine, etc.)
- improve MPI cluster by adding SLURM


## Credits
Irina Fedulova (@irifed)

https://github.com/AnsibleShipyard

https://github.com/ansible/ansible-examples
