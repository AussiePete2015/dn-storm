# Deployment via Vagrant
A [Vagrantfile](../Vagrantfile) is included in this repository that can be used to deploy Storm locally (to one or more VMs hosted under [VirtualBox](https://www.virtualbox.org/)) using [Vagrant](https://www.vagrantup.com/).  From the top-level directory of this repository a command like the following will (by default) deploy Storm to a single CentOS 7 virtual machine running under VirtualBox:

```bash
$ vagrant -s="192.168.34.42" up
```

Note that the `-s, --storm-list` flag must be used to pass an IP address (or a comma-separated list of IP addresses) into the [Vagrantfile](../Vagrantfile). In the example shown above, we are performing a single-node deployment of Storm, so an instance of Zookeeper will also be started on that same node during the deployment process and the Storm instance will be configured to talk to that Zookeeper instance .

When we are performing a multi-node deployment, then the nodes that make up the existing zookeeper ensemble must also be defined as part of the `vagrant ... up` command, as in this example:

```bash
$ vagrant -s="192.168.34.48,192.168.34.49,192.168.34.50" \
    -i='./zookeeper_inventory' up
```

This command will create a three-node Storm cluster, configuring those nodes to work with the Zookeeper ensemble described in the static inventory that we are passing into the `vagrant ... up` command shown here using the `-i, --inventory-file` flag. The argument passed in using this flag **must** point to an Ansible (static) inventory file containing the information needed to connect to the nodes in that Zookeeper ensemble (so that the playbook can gather facts about those nodes to configure the Storm nodes to talk to them correctly). As was mentioned in the discussion of provisioning a Storm cluster using a static inventory file in the [Deployment-Scenarios.md](Deployment-Scenarios.md) file, this inventory file could just contain a list of the nodes in the Zookeeper cluster and the information needed to connect to those nodes:

```bash
$ cat zookeeper_inventory
# example inventory file for a clustered deployment

192.168.34.18 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-zookeeper/.vagrant/machines/192.168.34.18/virtualbox/private_key'
192.168.34.19 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-zookeeper/.vagrant/machines/192.168.34.19/virtualbox/private_key'
192.168.34.20 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2202 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-zookeeper/.vagrant/machines/192.168.34.20/virtualbox/private_key'

$
```

Or it could contain the combined information for the members of the Zookeeper ensemble and Storm cluster, with the hosts broken out into `storm` and `zookeeper` host groups:

```bash
$ cat combined_inventory
# example combined inventory file for clustered deployment

192.168.34.48 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2203 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-storm/.vagrant/machines/192.168.34.48/virtualbox/private_key'
192.168.34.49 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2204 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-storm/.vagrant/machines/192.168.34.49/virtualbox/private_key'
192.168.34.50 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2205 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-storm/.vagrant/machines/192.168.34.50/virtualbox/private_key'
192.168.34.18 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-zookeeper/.vagrant/machines/192.168.34.18/virtualbox/private_key'
192.168.34.19 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-zookeeper/.vagrant/machines/192.168.34.19/virtualbox/private_key'
192.168.34.20 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2202 ansible_ssh_user='vagrant' ansible_ssh_private_key_file='/tmp/dn-zookeeper/.vagrant/machines/192.168.34.20/virtualbox/private_key'

[storm]
192.168.34.8
192.168.34.9
192.168.34.10

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

```

If the inventory file is similar to the first example, then all of the nodes in that inventory file will be assumed to be a part of the Zookeeper ensemble if that file is passed in using the `-i, --inventory-file` flag. If the inventory file passed in using the `-i, --inventory-file` flag is more like the second example, then only the hosts in the `zookeeper` host group list will be included in the `zookeeper` host group that is build dynamically within the `ansible-playbook` run that is triggered by the `vagrant ... up` or `vagrant ... provision` command.

If a Zookeeper inventory file is not provided when building a multi-node cluster, or if the file passed in does not describe a valid Zookeeper ensemble (one with an odd number of nodes, where the number of nodes is between three and seven, and where none of the nodes in that cluster are also being used as part of the Storm cluster we are deploying here), then an error will be thrown by the `vagrant` command.

In terms of how it all works, the [Vagrantfile](../Vagrantfile) is written in such a way that the following sequence of events occurs when the `vagrant ... up` command shown above is run:

1. All of the virtual machines in the cluster (the addresses in the `-s, --storm-list`) are created
1. After all of the nodes have been created, Storm is deployed all of those nodes and they are all configured as a cluster in a single Ansible playbook run
1. The `storm` service is started on all of the nodes that were just provisioned

Once the playbook run triggered by the [Vagrantfile](../Vagrantfile) is complete, we can simply browse to the Storm UI on one of the nodes in our cluster (eg. http://192.168.34.49:9797) to test and make sure that our Storm cluster is up and running. Do keep in mind that it does take some time for the Storm UI to start after the playbook run completes, so be patient.

So, to recap, by using a single `vagrant ... up` command we were able to quickly spin up a cluster consisting of of three Storm nodes and configure those nodes as a cluster that is associated with an existing Zookeeper ensemble. A similar `vagrant ... up` command could be used to build a cluster consisting of any number of Storm nodes, provided that a Zookeeper ensemble has already been provisioned that can be associated with the nodes in that cluster.

## Separating instance creation from provisioning
While the `vagrant up` commands that are shown above can be used to easily deploy Storm to a single node or to build a Storm cluster consisting of multiple nodes, the [Vagrantfile](../Vagrantfile) included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's [site.yml](../site.yml) file.

To create a set of virtual machines that we plan on using to build a Storm cluster without provisioning Storm to those machines, simply run a command similar to the following:

```bash
$ vagrant -s="192.168.34.48,192.168.34.49,192.168.34.50" \
    up --no-provision
```

This will create a set of three virtual machines with the appropriate IP addresses ("192.168.34.48", "192.168.34.49", and "192.168.34.50"), but will skip the process of provisioning those VMs with an instance of Storm. Note that when you are creating the virtual machines but skipping the provisioning step it is not necessary to provide the list of zookeeper nodes using the `-z, --zookeeper-list` flag, nor is it necessary to provide an inventory file for those zookeeper nodes using the `-i, --zk-inventory-file` flag.

To provision the machines that were created above and configure those machines as a Storm cluster, we simply need to run a command like the following:

```bash
$ vagrant -s="192.168.34.48,192.168.34.49,192.168.34.50" \
    -i='./combined_inventory' provision
```

That command will attach to the named instances and run the playbook in this repository's [site.yml](../site.yml) file on those node, resulting in a Storm cluster consisting of the nodes that were created in the `vagrant ... up --no-provision` command that was shown, above.

## Additional vagrant deployment options
While the commands shown above will install Storm with a reasonable, default configuration from a standard location, there are additional command-line parameters that can be used to control the deployment process triggered by a `vagrant ... up` or `vagrant ... provision` command. Here is a complete list of the command-line flags that can be supported by the [Vagrantfile](../Vagrantfile) in this repository:

* **`-s, --storm-list`**: the Storm address list; this is the list of nodes that will be created and provisioned, either by a single `vagrant ... up` command or by a `vagrant ... up --no-provision` command followed by a `vagrant ... provision` command; this command-line flag **must** be provided for almost every `vagrant` command supported by the [Vagrantfile](../Vagrantfile) in this repository
* **`-i, --inventory-file`**: the path to a static inventory file containing the parameters needed to connect to the nodes that make up the associated Zookeeper ensemble; this argument **must** be provided for any `vagrant` commands that involve provisioning of the instances that make up a Storm cluster
* **`-p, --path`**: the path that the distribution should be unpacked into; defaults to `/opt/apache-storm` 
* **`-u, --url`**: the URL that the Apache Storm distribution should be downloaded from; can be used to override the default URL used to to download the gzipped tarfile containing the Apache Storm distribution for situations with limited (or no) internet access
* **`-l, --local-path`**: the local directory (on the Ansible host) containing the Apache Storm (and Apache Zookeeper) distributions; when used, the distribution file (or files in a single-node deployment) will be uploaded to the target nodes from this directory, then unpacked into the installation directory. If this parameter and the `-u, --url` parameter are both defined and error will be thrown
* **`-d, --data`**: the path to the directory where Storm will store it's data; this defaults to `/var/lib` if not specified
* **`-y, --yum-url`**: the local YUM repository URL that should be used when installing packages during the node provisioning process. This can be useful when installing Storm onto CentOS-based VMs in situations where there is limited (or no) internet access; in this situation the user might want to install packages from a local YUM repository instead of the standard CentOS mirrors. It should be noted here that this parameter is not used when installing Storm on RHEL-based VMs; in such VMs this option will be silently ignored if set
* **`-f, --local-vars-file`**: the *local variables file* that should be used when deploying the cluster. A local variables file can be used to maintain the configuration parameter definitions that should be used for a given Storm cluster deployment, and values in this file will override any values that are either embedded in the [vars/storm.yml](../vars/storm.yml) file as defaults or passed into the `ansible-playbook` command as extra variables
* **`-c, --clear-proxy-settings`**: if set, this command-line flag will cause the playbook to clear any proxy settings that may have been set on the machines being provisioned in a previous ansible-playbook run. This is useful for situations where an HTTP proxy might have been set incorrectly and, in a subsequent playbook run, we need to clear those settings before attempting to install any packages or download the Storm distribution without an HTTP proxy

As an example of how these options might be used, the following command will download the gzipped tarfile containing the Apache Storm distribution from a local web server, configure Storm to store it's data in the `/data` directory, and install packages from the CentOS mirror on the web server at `http://192.168.34.254/centos` when provisioning the machines created by the `vagrant ... up --no-provision` command shown above:

```bash
$ vagrant -s="192.168.34.48,192.168.34.49,192.168.34.50" \
    -i='./combined_inventory' -d="/data" \
    -u='http://192.168.34.254/apache-storm/apache-storm-1.0.3.tar.gz' \
    -y='http://192.168.34.254/centos' provision
```

While the list of command-line options defined above may not cover all of the customizations that user might want to make when performing production Storm deployments, our hope is that the list shown above is more than sufficient for deploying Storm clusters using Vagrant for local testing purposes.
