# Supported configuration parameters
The playbook in the [site.yml](../site.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy Storm from the [vars/storm.yml](../vars/storm.yml) file. The parameters defined in that file define a reasonable set of defaults for a fairly generic Storm deployment, either to a single node or a cluster, including defaults for the URL that the Storm distribution should be downloaded from, the directory that the Storm nodes should store their data in, and the packages that must be installed on the node before the `storm` service can be started.

In addition to the defaults defined in the [vars/storm.yml](../vars/storm.yml) file, there are a larger set of parameters that can be used to either control the deployment of Storm to the nodes that will make up a cluster during an `ansible-playbook` run or to configure those Storm nodes once the installation is complete. In this section, we summarize these options, breaking them out into:

* parameters used to control the `ansible-playbook` run
* parameters used during the deployment process itself, and
* parameters used to configure our Storm nodes once Storm has been installed locally.

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, where the Storm distribution should be downloaded from, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`ansible_ssh_private_key_file`**: the location of the private key that should be used when connecting to the target nodes via SSH; this parameter is useful when there is one private key that is used to connect to all of the target nodes in a given playbook run
* **`ansible_user`**: the username that should be used when connecting to the target nodes via SSH; is useful if the same username is used when connecting to all of the target nodes in a given playbook run
* **`storm_url`**: the URL that the Apache Storm distribution should be downloaded from
* **`storm_version`**: the version of Storm that should be downloaded; used to switch versions when the distribution is downloaded using the default `storm_url`, which is defined in the [vars/storm.yml](../vars/storm.yml) file
* **`zookeeper_url`**: the URL that the Apache Zookeeper distribution should be downloaded from for single-node Storm deployments
* **`zookeeper_version`**: the version of Zookeeper that should be downloaded for single-node Storm deployments; used to switch versions when the distribution is downloaded using the default `zookeeper_url`, which is defined in the [vars/storm.yml](../vars/storm.yml) file
* **`cloud`**: if the inventory is being managed dynamically, this parameter is used to indicate the type of target cloud for the deployment (either `aws` or `osp`); this controls how the [build-app-host-groups](../common-roles/build-app-host-groups) common role retrieves the list of target nodes for the deployment
* **`local_path`**: used to pass in the local path (on the Ansible host) to a directory containing the Storm distribution file; the distribution file will be uploaded from this directory to the target hosts and unpacked into the `storm_dir` directory (note that in the case of a single-node deployment, the Zookeeper distribution file will also be uploaded from this directory and unpacked into the associated `zookeeper_dir` directory). The filename that will be uploaded will be the basename of the defined `storm_url` (and the corresponding basename of the `zookeeper_url` for single-node deployments)
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default
* **`proxy_env`**: a hash map that is used to define the proxy settings to use for downloading distribution files and installing packages; supports the `http_proxy`, `no_proxy`, `proxy_username`, and `proxy_password` fields as part of this hash map
* **`reset_proxy_settings`**: used to reset any HTTP/YUM proxy settings that may have been made in a previous playbook run back to the defaults (no proxy); this is useful when a proxy was incorrectly set in a previous playbook run and the user wants to return to a "no-proxy" setup in the current playbook run
* **`yum_repo_url`**: used to set the URL for a local YUM mirror. This parameter is only used for CentOS-based deployments; when deploying Storm to RHEL-based nodes this parameter is silently ignored and the RHEL package repositories defined locally on the node will be used for any packages installed during the deployment process

## Parameters used during the deployment process
These parameters are used to control the deployment process itself, defining things like where the distribution should be unpacked, the user/group that should be used when unpacking the distribution and starting the `storm` service, and which packages to install.

* **`storm_dir`**: the path to the directory that the Storm distribution should be unpacked into; defaults to `/opt/apache-storm` if unspecified
* **`zookeeper_dir`**: the path to the directory that the Zookeeper distribution should be unpacked into for single-node deployments; defaults to `/opt/apache-zookeeper` if unspecified
* **`storm_package_list`**: the list of packages that should be installed on the Storm nodes; typically this parameter is left unchanged from the default (which installs the OpenJDK packages needed to run Storm), but if it is modified the default, OpenJDK packages must be included as part of this list or an error will result when attempting to start the `storm` service
* **`storm_user`**: the username under which Storm should be installed and run; defaults to `storm`
* **`storm_group`**: the name of the user group under which Storm should be installed and run; defaults to `storm`

## Parameters used to configure the Storm nodes
These parameters are used configure the Storm nodes themselves during a playbook run, defining things like the interfaces that Storm should be listening on for requests and the directory where Storm should store its data.

* **`data_iface`**: the name of the interface that the Storm nodes in the cluster should use when talking with each other, when talking to the Zookeeper ensemble, and for user requests. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`iface_description_array`**: this parameter can be used in place of the `data_iface ` parameter described above, and it provides users with the ability to specify a description of that interface rather than identifying it by name (more on this, below)
* **`storm_data_dir`**: the name of the directory that Storm should use to store its data; defaults to `/var/lib` if unspecified. If necessary, this directory will be created as part of the playbook run
* **`nimbus_childopts`**: used to set the JVM options for the master; defaults to `-Xmx1024m` if unspecified
* **`ui_port`**: used to set the `ui.port` property for the master node; defaults to `9797` if unspecified
* **`ui_childopts`**: used to set the `ui.childopts` property for the master node; defaults to `-Xmx768m` if unspecified
* **`worker_childopts`**: used to set the JVM options for the task workers; defaults to `-Xmx768m` if unspecified

## Interface names vs. interface descriptions
For some operating systems on some platforms, it can be difficult (if not impossible) to determine the name of the interface that should be passed into the playbook using the `data_iface` parameter that we described, above. In those situations, the playbook in this repository provides an alternative; specifying that interface using the `iface_description_array` parameter instead.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```json
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
    ]
```

the playbook will return the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `data_iface` fact. This fact can then be used later in the playbook to correctly configure the nodes to talk to each other and listen on the proper interface for user requests.

It should be noted that if you choose to take this approach when constructing your `ansible-playbook` runs, a matching entry in the `iface_description_array` must be specified for the `data_iface` network, otherwise the default value of `eth0` will be used for this fact (and the playbook run may result in nodes that are at best misconfigured; if the `eth0` network does not exist then the playbook will fail to run altogether).
