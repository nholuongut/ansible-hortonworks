ansible-hortonworks installation guide
------------------------------

* These Ansible playbooks will build a Cloud environment on a private OpenStack.
* And then deploy a Hortonworks cluster (either Hortonworks Data Platform or Hortonworks DataFlow) using Ambari Blueprints.

---


# Build setup

Before building anything, the build node / workstation from where Ansible will run should be prepared.

This node must be able to connect to the cluster nodes via SSH and to the OpenStack APIs via HTTPS.

As OpenStack environments are usually private, you might need to build such a node in the OpenStack environment.


## macOS

1. Install the required packages

   ```
   brew install python
   pip2 install virtualenv
   pip2 install virtualenvwrapper
   ```


2. Create and source the Python virtual environment

   ```
   virtualenv ~/ansible; source ~/ansible/bin/activate
   ```


3. Install the required Python packages inside the virtualenv

   ```
   pip install setuptools --upgrade
   pip install pip --upgrade
   pip install ansible shade python-openstackclient==3.12.0 os-client-config==1.28.0
   ```


4. Turn off SSL validation (required if your OpenStack endpoints don't use trusted certs)

   ```
   defaults_json_path=~/ansible/lib/python2.7/site-packages/os_client_config/defaults.json; grep -q verify $defaults_json_path || sed -i'' -e '/{$/ a\
   "verify": false,\
   ' $defaults_json_path
   ```


5. Install the SSH private key

   The build node / workstation will need to login via SSH to the cluster nodes.

   For this to succeed, the SSH private key needs to be placed on the build node / workstation, normally under .ssh, for example: `~/.ssh/field.pem`. It can be placed under any path as this file will be referenced later.

   It should have `400` permissions: `chmod 0400 ~/.ssh/field.pem`.

   The SSH public key must be present on the OpenStack environment as it will be referenced when the nodes will be built (this can be checked on the Dashboard, under `Compute` -> `Access and Security` -> `Key Pairs` tab).


## CentOS/RHEL 7

1. Install the required packages

   ```
   sudo yum -y install epel-release || sudo yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
   sudo yum -y install gcc gcc-c++ python-virtualenv python-pip python-devel libffi-devel openssl-devel libyaml-devel sshpass git vim-enhanced
   ```


2. Create and source the Python virtual environment

   ```
   virtualenv ~/ansible; source ~/ansible/bin/activate 
   ```


3. Install the required Python packages inside the virtualenv

   ```
   pip install setuptools --upgrade
   pip install pip --upgrade   
   pip install ansible shade python-openstackclient==3.12.0 os-client-config==1.28.0
   ```


4. Turn off SSL validation (required if your OpenStack endpoints don't use trusted certs)
  
   ```
   defaults_json_path=~/ansible/lib64/python2.7/site-packages/os_client_config/defaults.json; grep -q verify $defaults_json_path || sed -i '/{$/a "verify": false,' $defaults_json_path
   ```


5. Install the SSH private key

   The build node / workstation will need to login via SSH to the cluster nodes.
   
   For this to succeed, the SSH private key needs to be placed on the build node / workstation, normally under .ssh, for example: `~/.ssh/field.pem`. It can be placed under any path as this file will be referenced later.
   
   It should have `400` permissions: `chmod 0400 ~/.ssh/field.pem`.
   
   The SSH public key must be present on the OpenStack environment as it will be referenced when the nodes will be built (this can be checked on the Dashboard, under `Compute` -> `Access and Security` -> `Key Pairs` tab).


## Ubuntu 14+

1. Install required packages:

   ```
   sudo apt-get update
   sudo apt-get -y install unzip python-virtualenv python-pip python-dev sshpass git libffi-dev libssl-dev libyaml-dev vim
   ```


2. Create and source the Python virtual environment

   ```
   virtualenv ~/ansible; source ~/ansible/bin/activate  
   ```


3. Install the required Python packages inside the virtualenv

   ```
   pip install setuptools --upgrade
   pip install pip --upgrade
   pip install ansible shade python-openstackclient==3.12.0 os-client-config==1.28.0
   ```


4. Turn off SSL validation (required if your OpenStack endpoints don't use trusted certs)
  
   ```
   defaults_json_path=~/ansible/local/lib/python2.7/site-packages/os_client_config/defaults.json; grep -q verify $defaults_json_path || sed -i '/{$/a "verify": false,' $defaults_json_path
   ```

5. Install the SSH private key

   The build node / workstation will need to login via SSH to the cluster nodes.
   
   For this to succeed, the SSH private key needs to be placed on the build node / workstation, normally under .ssh, for example: `~/.ssh/field.pem`. It can be placed under any path as this file will be referenced later.
   
   It should have `400` permissions: `chmod 0400 ~/.ssh/field.pem`.
   
   The SSH public key must be present on the OpenStack environment as it will be referenced when the nodes will be built (this can be checked on the Dashboard, under `Compute` -> `Access and Security` -> `Key Pairs` tab).


# Setup the OpenStack credentials

1. Download the OpenStack RC file

   Login to your OpenStack dashboard, and download your user specific OpenStack RC file.
   This is usually found on `Compute` -> `Access and Security` under the `API Access` tab. Download the v3 if available.


2. Apply the OpenStack credentials

   Copy the file to the build node / workstation in a private location (for example the user's home folder).
   
   And `source` the file so it populates the existing session with the OpenStack environment variables.
   Type your OpenStack account password when prompted.
   
   ```
   source ~/ansible/bin/activate
   source ~/*-openrc.sh
   Please enter your OpenStack Password: 
   ```
   
   You can verify if it worked by trying to list the existing OpenStack instances:
   ```
   nova --insecure list
   ```


# Clone the repository

Upload the ansible-hortonworks repository to the build node / workstation, preferable under the home folder.

If the build node / workstation can directly download the repository, run the following:

```
cd && git clone https://github.com/nholuongut/ansible-hortonworks.git
```

If your GitHub SSH key is installed, you can use the SSH link:

```
cd && git clone git@github.com:nholuongut/ansible-hortonworks.git
```


# Set the OpenStack variables

Modify the file at `~/ansible-hortonworks/inventory/openstack/group_vars/all` to set the OpenStack configuration.


## cloud_config
This section contains variables that are cluster specific and are used by all nodes:

| Variable        | Description                                                                                                |
| --------------- | ---------------------------------------------------------------------------------------------------------- |
| name_suffix     | A suffix that will be appended to the name of all nodes. Usually it's a domain, but can be anything or even the empty string `''`. |
| zone            | The name of the OpenStack zone.                         |
| admin_username  | The Linux user with sudo permissions. This user is specific to the image used. For example, in a CentOS image, it can be `centos` or in a Ubuntu image it can be `ubuntu`. |
| ssh.keyname     | The name of the SSH key that will be placed on cluster nodes at build time. This SSH key must already exist in the OpenStack environment. |
| ssh.privatekey  | Local path to the SSH private key that will be used to login into the nodes. This is the key uploaded to the build node as part of the Build Setup, step 5. |


## nodes config

This section contains variables that are node specific.

Nodes are separated by groups, each group defining a specific node role, for example master, slave, edge.

There can be any number of roles so other roles can be added to correspond with the required architecture.

And roles can have any names and any number of nodes but they should correspond with the host groups in the Ambari Blueprint.

| Variable        | Description                                                               |
| --------------- | ------------------------------------------------------------------------- |
| role            | The name of the role. This will be appended to the cluster name in order to form a unique group in the OpenStack Zone. This group is used to derive the nodes names (if node count is greater than 1, numbers will be appended to the group name to uniquely identify nodes). |
| count           | The number of nodes to be built with this role. |
| image           | The name or ID of the OS image to be used. A list of the available images can be found by running `glance --insecure image-list`. |
| flavor          | The name or ID of the flavor / size of the node. A list of all the available flavors can be found by running `nova --insecure flavor-list`. |                                                      |
| public_ip       | If the Public IP of the cluster node should be used when connecting to it. Required if the build node does not have access to the private IP range of the cluster nodes. |
| ambari_server   | Set it to `true` if the role also defines an Ambari Server. The number of nodes with this role should be 1. If there are more than 1 node, ambari-server will be installed on all of them, but only the first one (in alphabetical order) will be used by the Ambari Agents. |


# Set the cluster variables

## cluster config file

Modify the file at `~/ansible-hortonworks/playbooks/group_vars/all` to set the cluster configuration.

| Variable                   | Description                                                                                                 |
| -------------------------- | ----------------------------------------------------------------------------------------------------------- |
| cluster_name               | The name of the cluster. This is also used by default in the cloud components that require uniqueness, such as the name of the nodes or tags. |
| ambari_version             | The Ambari version, in the full, 4-number form, for example: `2.6.2.2`.                                     |
| hdp_version                | The HDP version, in the full, 4-number form, for example: `2.6.5.0`.                                        |
| hdf_version                | The HDF version, in the full, 4-number form, for example: `3.1.2.0`.                                        |
| repo_base_url              | The base URL for the repositories. Change this to the local web server url if using a Local Repository. `/HDP/<OS>/2.x/updates/<latest.version>` (or `/HDF/..`) will be appended to this value to set it accordingly if there are additional URL paths. |
| java                       | Can be set to `embedded` (default - downloaded by Ambari), `openjdk` or `oraclejdk`. If `oraclejdk` is selected, then the `.x64.tar.gz` package must be downloaded in advance from [Oracle](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html). Same with the [JCE](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html) package. These files can be copied to all nodes in advanced or only to the Ansible Controller and Ansible will copy them. This behaviour is controlled by the `oraclejdk_options.remote_files` setting. |
| oraclejdk_options          | These options are only relevant if `java` is set to `oraclejdk`. |
| .base_folder               | This indicates the folder where the Java package should be unpacked to. The default of `/usr/java` is also used by the Oracle JDK rpm. |
| .tarball_location          | The location of the tarball file. This can be the location on the remote systems or on the Ansible controller, depending on the `remote_files` variable. |
| .jce_location              | The location of the JCE package zip file. This can be the location on the remote systems or on the Ansible controller, depending on the `remote_files` variable. |
| .remote_files              | If this variable is set to `yes` then the tarball and JCE files must already be present on the remote system. If set to `no` then the files will be copied by Ansible (from the Ansible controller to the remote systems). |
| external_dns               | This controls the type of DNS to be used. If `yes` it will use whatever DNS is currently set up. If `no` it will populate the `/etc/hosts` file with all cluster nodes. |
| disable_firewall           | This variable controls the local firewall service (iptables, firewalld, ufw). Sometimes, a local firewall service might run and block inter-node cluster communication. In these circumstances the local firewall service should be disabled as traffic rules should be provided by an external firewall such as Security Groups. Set to `yes` to disable the existing local firewall service if it blocks the inter-node cluster communication. |
| accept_gpl                 | Set to `yes` to enable Ambari Server to download and install GPL Licensed packages as explained on the [documentation](https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.1.5/bk_ambari-administration/content/enabling_lzo.html). |

### security configuration

| Variable                       | Description                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| security                       | This variable controls the Kerberos security configuration. If set to `none`, Kerberos will not be enabled. Otherwise the choice is between `mit-kdc` or `active-directory`. |
| security_options               | These options are only relevant if `security` is not `none`. All of the options here are used for an Ambari managed security configuration. No manual option is available at the moment. |
| .external_hostname             | The hostname/IP of the Kerberos server. This can be an existing Active Directory or [MIT KDC](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.1/bk_security/content/_optional_install_a_new_mit_kdc.html). If left empty `''` then the playbooks will install the MIT KDC on the Ambari node and prepare everything. |
| .realm                         | The realm that will be used when creating service principals. |
| .admin_principal               | The Kerberos principal that has the permissions to create new users. No need to append the realm to this value. In case of Active Directory, this user must have `Create, delete, and manage user accounts` permissions over the OU container. If installing a new MIT KDC this user will be created by the playbook. |
| .admin_password                | The password for the above user. |
| .kdc_master_key                | The master password for the Kerberos database. Only used when installing a new MIT KDC (when `security` is `mit-kdc` and `external_hostname` is set to `''`. |
| .ldap_url                      | The URL to the Active Directory LDAPS interface. Only used when `security` is set to `active-directory`. |
| .container_dn                  | The distinguished name (DN) of the container that will store the service principals. Only used when `security` is set to `active-directory`. |
| .http_authentication           | Set to `yes` to enable Kerberos HTTP authentication (SPNEGO) for most UIs.


## ambari-server config file

Modify the file at `~/ansible-hortonworks/playbooks/group_vars/ambari-server` to set the Ambari Server specific configuration.

| Variable                       | Description                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| ambari_admin_user              | The Ambari administrator's username, normally `admin`. This user and the password bellow are used to login to Ambari for API requests. |
| ambari_admin_password          | The Ambari password of the `ambari_admin_user` user previously set. If the username is `admin` and this password is different than the default `admin`, the `ambari-config` role will change the default password with the one set here. |
| ambari_admin_default_password  | The default password for the Ambari `admin` user. This is normally `admin` after Ambari is first installed. No need to change this unless there's a change in the Ambari codebase. |
| wait / wait_timeout            | Set this to `true` if you want the playbook to wait for the cluster to be successfully built after applying the blueprint. The timeout setting controls for how long (in seconds) should it wait for the cluster build. |
| default_password               | A default password for all required passwords which are not specified in the blueprint.                                                                               |
| config_recommendation_strategy | Configuration field which specifies the strategy of applying configuration recommendations to a cluster as explained in the [documentation](https://cwiki.apache.org/confluence/display/AMBARI/Blueprints#Blueprints-ClusterCreationTemplateStructure). |
| cluster_template_file          | The path to the cluster creation template file that will be used to build the cluster. It can be an absolute path or relative to the `ambari-blueprint/templates` folder. The default should be sufficient for cloud builds as it uses the `cloud_config` variables and [Jinja2 Template](http://jinja.pocoo.org/docs/dev/templates/) to generate the file. |

### database configuration

| Variable                                 | Description                                                                                                |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| database                                 | The type of database that should be used. A choice between `embedded` (Ambari default), `postgres`, `mysql` or `mariadb`. |
| database_options                         | These options are only relevant for the non-`embedded` database. |
| .external_hostname                       | The hostname/IP of the database server. This needs to be prepared as per the [documentation](https://docs.hortonworks.com/HDPDocuments/Ambari-2.6.2.0/bk_ambari-administration/content/ch_amb_ref_using_existing_databases.html). No need to load any schema, this will be done by Ansible, but the users and databases must be created in advance. If left empty `''` then the playbooks will install the database server on the Ambari node and prepare everything with the settings defined bellow. To change any settings (like the version or repository path) modify the OS specific files under the `playbooks/roles/database/vars/` folder. |
| .ambari_db_name,_username,_password      | The name of the database that Ambari should use and the username and password to connect to it. If `database_options.external_hostname` is defined, these values will be used to connect to the database, otherwise the Ansible playbook will create the database and the user. |
| .hive_db_name,_username,_password        | The name of the database that Hive should use and the username and password to connect to it. If `database_options.external_hostname` is defined, these values will be used to connect to the database, otherwise the Ansible playbook will create the database and the user. |
| .oozie_db_name,_username,_password       | The name of the database that Oozie should use and the username and password to connect to it. If `database_options.external_hostname` is defined, these values will be used to connect to the database, otherwise the Ansible playbook will create the database and the user. |
| .rangeradmin_db_name,_username,_password | The name of the database that Ranger Admin should use and the username and password to connect to it. If `database_options.external_hostname` is defined, these values will be used to connect to the database, otherwise the Ansible playbook will create the database and the user. |
| .registry_db_name,_username,_password    | The name of the database that Schema Registry should use and the username and password to connect to it. If `database_options.external_hostname` is defined, these values will be used to connect to the database, otherwise the Ansible playbook will create the database and the user. |
| .streamline_db_name,_username,_password  | The name of the database that SAM should use and the username and password to connect to it. If `database_options.external_hostname` is defined, these values will be used to connect to the database, otherwise the Ansible playbook will create the database and the user. |

### ranger configuration

| Variable                       | Description                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| ranger_options                 | These options are only relevant if `RANGER_ADMIN` is a component of the dynamic Blueprint stack.           |
| .ranger_admin_password         | The password for the Ranger admin users (both admin and amb_ranger_admin).                                 |
| .enable_plugins                | If set to `yes` the plugins for all of the available services will be enabled. With `no` Ranger would be installed but not functional. |

### blueprint configuration

| Variable                       | Description                                                                                                |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| blueprint_name                 | The name of the blueprint as it will be stored in Ambari.                                                  |
| blueprint_file                 | The path to the blueprint file that will be uploaded to Ambari. It can be an absolute path or relative to the `roles/ambari-blueprint/templates` folder. The blueprint file can also contain [Jinja2 Template](http://jinja.pocoo.org/docs/dev/templates/) variables. |
| blueprint_dynamic              | Settings for the dynamic blueprint template - only used if `blueprint_file` is set to `blueprint_dynamic.j2`. The role names must match the roles from the inventory setting file `~/ansible-hortonworks/inventory/openstack/group_vars/all`. The chosen components are split into two lists: clients and services. The chosen Component layout must respect Ambari Blueprint restrictions - for example if a single `NAMENODE` is configured, there must also be a `SECONDARY_NAMENODE` component. |


# Build the Cloud environment

Run the script that will build the Cloud environment.

Set first the `CLOUD_TO_USE` environment variable to `openstack`.

```
export CLOUD_TO_USE=openstack
cd ~/ansible-hortonworks*/ && bash build_cloud.sh
```

You may need to load the environment variables if this is a new session:

```
source ~/ansible/bin/activate
source ~/*-openrc.sh
```


# Install the cluster

Run the script that will install the cluster using Blueprints while taking care of the necessary prerequisites.

Make sure you set the `CLOUD_TO_USE` environment variable to `openstack`.

```
export CLOUD_TO_USE=openstack
cd ~/ansible-hortonworks*/ && bash install_cluster.sh
```

You may need to load the environment variables if this is a new session:

```
source ~/ansible/bin/activate
source ~/*-openrc.sh
```


This script will apply all the required playbooks in one run, but you can also apply the individual playbooks by running the following wrapper scripts:

- Prepare the nodes: `prepare_nodes.sh`
- Install Ambari: `install_ambari.sh`
- Configure Ambari: `configure_ambari.sh`
- Apply Blueprint: `apply_blueprint.sh`
- Post Install: `post_install.sh`
