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

1. You have access to a Ubuntu server with password-less ssh root login configured, i.e. you should be able to execute `ssh server whoami` to access it and get the response `root` without entering any password. If you don't have this configured, then [this guide](http://osxdaily.com/2012/05/25/how-to-set-up-a-password-less-ssh-login/) will help you with the basic config and [this guide](http://nerderati.com/2011/03/simplify-your-life-with-an-ssh-config-file/) will show you how to assign a symbolic name to a user@host combo. This latter step is important as the config files are named using the name of the server (more on that below)

2. The server is configured with Ubuntu 12.04 (other OS versions may work but are not recommended). In case you choose not to install devstack and use an existing Openstack Grizzly installation, then the basic requirement is that server is configured with Ubuntu (other OSes may work but have not been tested).

3. The server has at least 150GB disk free. You might get away with much less, but in my devstack setup I configure a 120GB volume for Openstack to use (change VOLUME_BACKING_FILE_SIZE in 0-3-devstack_reinstall.exp)

4. The server either has direct or proxied Internet access.

5. The server has at least 8GB RAM. Much more is much, much better though ;-)

## Usage

Before starting, spend a few minutes and read through the comments about the scripts in the section below.

Then create a custom project configuration by cloning the **qimigo:qimigo.wan.cfg** file in the **configs directory** and adjust the settings. Most settings are hopefully self-evident, but the config file clearly needs a bit more commenting… In case you have a performant server, then you can definitely "up" the settings for the flavors.

The odd naming of the configuration file reflects the fact that you can have multiple ssh names for the same config (in this case both *qimigo* and *qimigo.wan*). This is done in case you want to enable access to the server both from a local network and an external network, in which case you will have two different entries in your .ssh/config file. If you only have one entry called **server_name**, then the file can be renamed to **server_name.cfg**

Once the configuration file has been created, go ahead and configure the server if it is freshly installed with Ubuntu (in this case the **qimigo:qimigo.wan.cfg** config file will be used):

```
0-1-system_init_new.exp qimigo
0-2-system_set_proxy.exp qimigo
```

Then install devstack (but **PLEASE** read the caveats below that the install process is brutal and will delete any mysql installation):

```
0-3-devstack_install.exp qimigo
```

Then configure openstack:

```
1-1-openstack_init_project.exp
1-2-openstack_init_settings.exp
```

Proceed with creating the bootstrap and microbosh VMs:

```
2-1-bosh_bootstrap_vm_create.exp
2-2-bosh_bootstrap_vm_init.exp
2-3-create_microbosh.exp
```

Finally configure and launch Cloud Foundry:

```
2-4-update_bosh_stemcell.exp
2-5-cf_create_releases.exp
2-6-cf_deploy.exp
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


## Script execution timings

The following table shows measured execution times for the scripts on a slow host with fast Internet access (100MB). Internet access speed will have a huge impact on the elapsed time, as will cpu.

Script    | Elapsed time (minutes)
:-------: | ----------------------
0-3       |  15
1-1       |   1
1-2        |   2
2-1       |   2
2-2        |  14
2-3        |  17
2-4        |  10
2-5        |  45
2-6        |   50


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

