## Cluster provisioning tool based on Vagrant and Ansible

This is a tool for provisioning a computational cluster on a cloud environment based on [Vagrant](http://www.vagrantup.com/) and [Ansible](http://www.ansible.com/home).

It allows to provision and test the cluster on a local machine (e.g. laptop) first, using Vagrant and [Virtualbox](https://www.virtualbox.org/) VMs, and then pushing same (or scaled) configuration to the cloud after few tweaks in the Vagrantfile.

Virtual servers are created by Vagrant and further organized to a cluster (all necessary software is installed and configured) by Ansible.

For the moment, this tool supports only [SoftLayer](http://www.softlayer.com/) as cloud provider, but it is trivial to extend it to support other cloud providers, e.g. [Amazon EC2](http://aws.amazon.com/ec2/), [Google Compute Engine](https://cloud.google.com/products/compute-engine/), etc. as long as they are [supported by Vagrant](http://docs.vagrantup.com/v2/providers/index.html).

Ansible is an awesome orchestration tool which allows to provision any type software on a cluster (e.g. MPI, Hadoop, etc.)

Currently, this tool contains Ansible playbook for provisioning trivial [OpenMPI](http://www.open-mpi.org/) cluster, but it will be extended to support Hadoop, Spark, etc.

## Usage

### Prerequisites

Obviously, it is required to install Vagrant and Ansible on your local machine first. Appropriate instructions for many operating systems could be found on [Vagrant](http://www.vagrantup.com/downloads) and [Ansible](http://docs.ansible.com/intro_installation.html) websites.

In short, for example in Mac OS X steps are the following:

- download Vagrant binary package

```
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.3.dmg
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

### Provision a cluster on a cloud

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

## Known issues

Sometimes instance provisioning time on SoftLayer is too long, and Vagrant throws timeout errors. In this case it is advised to switch datacenter and try again or simply retry in same datacenter. For any other problems please create an issue on this GitHub repository.

## Future work

- add support for other clustering software (Hadoop, BDAS, etc.)
- add support for other cloud providers (Amazon EC2, Google Compute Engine, etc.)
- improve MPI cluster by adding SLURM


## Credits
Irina Fedulova (@irifed)
