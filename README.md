# cf-openstack-jumpstart

cf-openstack-jumpstart is a set of expect scripts that will enable you to install Cloud Foundry from scratch on top of Openstack, optionally behind a corporate firewall.

The scripts will:

1. Install and configure Openstack/Grizzly using devstack
2. Create a bootstrap VM running Ubuntu 12.04
3. Use bosh-bootstrap to create a microbosh instance
4. Launch Cloud Foundry using bosh and microbosh

The aim in creating these scripts has been to leverage other tools as much as possible but provide a completely scripted path from a fresh machine to a running instance of Cloud Foundry.

## Assumptions

The following assumptions have been made:

1. You have access to a Ubuntu server with password-less ssh root login configured, i.e. you should be able to execute `ssh server whoami` to access it and get the response `root` without entering any password. If you don't have this configured, then [this guide](http://osxdaily.com/2012/05/25/how-to-set-up-a-password-less-ssh-login/) will help you with the basic configuration of ssh and [this guide](http://nerderati.com/2011/03/simplify-your-life-with-an-ssh-config-file/) will show you how to use a symbolic ssh name instead of user@host. This latter step is important as the config files are named using the name of the server (more on that below)

2. The server is configured with Ubuntu 12.04 (other OS versions may work but are not recommended). In case you choose not to install devstack and use an existing Openstack Grizzly installation, then the basic requirement is that server is configured with Ubuntu (other OSes may work but have not been tested).

3. The server has at least 150GB disk free. You might get away with much less, but in my devstack setup I configure a 120GB volume for Openstack to use (change VOLUME_BACKING_FILE_SIZE in 0-3-devstack_reinstall.exp)

4. The server either has direct or proxied Internet access.

5. I don't know the lower limit of memory requirements for the server, but suspect that it will be difficult to launch Cloud Foundry with much less than 16GB. More memory will definitely make your life easier ;-)

## Usage

Before starting, spend a few minutes and read through the comments about the scripts in the section below.

Then create a custom project configuration by cloning the **bigserver:bigserver.wan.cfg** file in the **configs directory** and adjust the settings. Most settings are hopefully self-evident, but the config file clearly needs a bit more commenting… In case you have a performant server, then you can definitely "up" the settings for the flavors.

The odd naming of the configuration file reflects the fact that you can have multiple ssh names for the same config (in this case both *bigserver* and *bigserver.wan*). This is done in case you want to enable access to the server both from a local network and an external network, in which case you will have two different entries in your .ssh/config file. If you only have one entry called **server_name**, then the file can be renamed to **server_name.cfg**

Once the configuration file has been created, go ahead and configure the server if it is freshly installed with Ubuntu (in this case the **bigserver:bigserver.wan.cfg** config file will be used):

```
0-1-system_init_new.exp bigserver
0-2-system_set_proxy.exp bigserver
```

Then install devstack (but **PLEASE** read the caveats below that the install process is brutal and will delete any mysql installation):

```
0-3-devstack_install.exp bigserver
```


Then configure openstack:

```
1-1-openstack_init_project.exp bigserver
1-2-openstack_init_settings.exp bigserver
```

If these scripts complete without problems then you most likely have a working Openstack installation. Regardless, it's a really great idea to run some further verifications that everything works ok:

   * login to the openstack UI at http://bigserver (user: admin / password: admin)
   * switch to the project view: Click on Project -> Current Project -> bigserver
   * create and delete a volume: Click on Volumes -> Create Volume and create a sample small volume and check that it is being created without error
   * launch and delete an instance
   --> any website link for this?


Proceed with creating the bootstrap and microbosh VMs:

```
2-1-bosh_bootstrap_vm_create.exp bigserver
2-2-bosh_bootstrap_vm_init.exp bigserver
2-3-create_microbosh.exp bigserver
```

Finally configure and launch Cloud Foundry:

```
2-4-update_bosh_stemcell.exp bigserver
2-5-cf_create_releases.exp bigserver
2-6-cf_deploy.exp bigserver
```



## Scripts (Used in this order):

#### Phase 0 - System initialization and devstack setup

* **0-1-system_init_new**
Should be applied directly to a newly installed instance of Ubuntu (after ssh has been configured). It fixes an issue with the locale setting and performs an `apt-get update` + `apt-get dist-upgrade`. This script can be applied multiple times without problems and is useful if you configure a box from scratch (which is recommended)

* **0-2-system_set_proxy**
Creates a proxy configuration file `/etc/profile.d/proxy_inits.sh`. This file is being called each time a new bash shell is created and will in turn create configuration files for `apt` (system-wide), `wget` (user), `git` (user), `gem` (user), `curl` (user) and python `pip` (user). This approach is used instead of setting http_proxy and https_proxy system wide as that will interfer with Openstack and Cloud Foundry. This script can be applied multiple times without problems.

* **0-3-devstack_install**
Reinstalls devstack from scratch. **Please be aware that this script uses a very "brutal" approach**, in particular it will completely clear any mysql installation. Note that the local settings are contained in the file local_configs.

#### Phase 1 - Initialize Openstack

* **1-1-openstack_init_project**
Creates a configuration file *inits/openrc-server_name.rc* on the server. In case you have a custom installation of Openstack, then verify that environment settings in this file are correctly generated. If not, you can fix the file manually. The key to getting this to work is to ensure that the user that ssh uses for autologin has all the environment settings correctly done.

* **1-1-openstack_init_settings**
Performs a number of configurations of Openstack that are specific to the project, such as quota adjustments, creation of custom flavors, adding floating ips, creation of security groups and installation of required Ubuntu images (cuurently only 12.04 for bootstrap)

#### Phase 2 - Install Cloud foundry

* **2-1-bosh_bootstrap_vm_create**
Creates the VM used for bosh bootstrap

* **2-2-bosh_bootstrap_vm_init**
Initializes the bootstrap vm; downloads bosh-bootstrap and installs ruby with dependencies.

* **2-3-create_microbosh**
Downloads latest microbosh stemcells, then launches bosh-bootstrap and creates the microbosh VM. It also syncronizes a cache on the server onto the bootstrap VM (this cache is syncronized back at a later stage). The cache is used to improve speed of reinstalls and multiple projects.

* **2-4-update_bosh_stemcell**
Download the bosh stemcell, optionally patches it with proxy configuration (if HTTP_PROXY is set) and uploads the stemcell to microbosh

* **2-5-cf_create_releases**
Downloads, prepares and uploads *cf-release*, *cf-services* and *cf-services-contrib* to microbosh

* **2-6-cf_deploy**
Deploys Cloud Foundry using the templated configuration file in the templates directory

## Issues

If you run into this problem during the final compile phase:
```
Compiling packages
  postgresql92/0.1-dev (00:03:00)
  postgresql91/0.1-dev (00:03:53)
  mysql55/0.1-dev (00:04:05)
  postgresql/0.1-dev (00:04:07)
  mysql/0.1-dev: unknown agent_task_id (00:01:09)
  mysqlclient/0.1-dev (00:00:03)
  libyaml/0.1-dev (00:00:32)
  sqlite/0.1-dev (00:01:03)
Error                   8/42 00:05:09

Error 450001: unknown agent_task_id

Task 5 error

For a more detailed error report, run: bosh task 5 --debug
===>
````

Then try to just relaunch the compile by redoing the last command (you will have an interactive prompt after the compile phase):

```
===> bosh deploy
```

Note that **bosh deploy** might be prefixed by proxy environment settings. As an alternative, you can just exit the script (type **exit** multiple times) and relaunch the script.

## Script execution timings

In case you wonder how long time this whole process takes, I have gathered some sample execution times (in minutes). The first host (server1) has low system resources but fast network access (100MB), while the second server has slow network access but a fast CPU.


Script    | Fast network / slow CPU | Slow network / fast CPU
:-------: | :---------------------: | :---------------------:
0-3       |  15                     |   22
1-1       |   1                     |    1
1-2       |   2                     |    1
2-1       |   2                     |    2
2-2       |  14                     |   25
2-3       |  17                     |   13
2-4       |  10                     |   15
2-5       |  45                     |  125
2-6       |  50                     |


## Utility scripts

* **login_to_openstack**
Login to the openstack environment

* **login_to_bootstrap**
Login to the bootstrap VM. Very useful for interacting with bosh directly

* **login_to_microbosh**
Login to microbosh

* **util_setproxy_bootstrap**
Sets the proxy config in the bootstrap VM. This is only required if it changes after bootstrap has been created as this is otherwise done automatically by the bootstrap creation script.

## Copyright

All documentation and source code is copyright of [Farestam Consulting AB](http://www.farestam.com)

