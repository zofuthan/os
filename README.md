# RancherOS

The smallest, easiest way to run Docker in production at scale.  Everything in RancherOS is a container managed by Docker.  This includes system services such as udev and rsyslog.  RancherOS includes only the bare minimum amount of software needed to run Docker.  This keeps the binary download of RancherOS to about 20MB.  Everything else can be pulled in dynamically through Docker.

## How this works

Everything in RancherOS is a Docker container.  We accomplish this by launching two instances of
Docker.  One is what we call the system Docker which runs as PID 1.  System Docker then launches
a container that runs the user Docker.  The user Docker is then the instance that gets primarily
used to create containers.  We created this separation because it seemed logical and also
it would really be bad if somebody did `docker rm -f $(docker ps -qa)` and deleted the entire OS.

![How it works](docs/rancheros.png "How it works")


## Latest Release

**v0.1.2 - Docker 1.5.0 - Linux 3.18.6**

### ISO
 https://github.com/rancherio/os/releases/download/v0.1.2/rancheros.iso

### Amazon

Region | Type | AMI |
-------|------|------
ap-northeast-1| PV | [ami-95db3395](https://console.aws.amazon.com/ec2/home?region=ap-northeast-1#launchAmi=ami-95db3395)
ap-southeast-1| PV | [ami-64073136](https://console.aws.amazon.com/ec2/home?region=ap-southeast-1#launchAmi=ami-64073136)
ap-southeast-2| PV | [ami-c588f9ff](https://console.aws.amazon.com/ec2/home?region=ap-southeast-2#launchAmi=ami-c588f9ff)
eu-west-1| PV | [ami-57cd5f20](https://console.aws.amazon.com/ec2/home?region=eu-west-1#launchAmi=ami-57cd5f20)
sa-east-1| PV | [ami-a311a8be](https://console.aws.amazon.com/ec2/home?region=sa-east-1#launchAmi=ami-a311a8be)
us-east-1| PV | [ami-90f6d3f8](https://console.aws.amazon.com/ec2/home?region=us-east-1#launchAmi=ami-90f6d3f8)
us-west-1| PV | [ami-5992751d](https://console.aws.amazon.com/ec2/home?region=us-west-1#launchAmi=ami-5992751d)
us-west-2| PV | [ami-2b67451b](https://console.aws.amazon.com/ec2/home?region=us-west-2#launchAmi=ami-2b67451b)

SSH keys are added to the **`rancher`** user.


## Running

### Cloud

Currently we only have RancherOS available in EC2 but more clouds will come based on demand.  Follow the links in the Release section above to deploy using our AMIs.

### Vagrant

Vagrant is the simplest way to try out RancherOS from the desktop.  Refer to the [RancherOS Vagrant project](https://github.com/rancherio/os-vagrant)

### Other

QEMU, VirtualBox, and VMware are all supported.  Just
boot from the `rancheros.iso` with at least 1GB of memory.

## Logging in

If you are using EC2 or Vagrant then SSH keys are properly put into place.  This means `ssh -i <KEY> -l rancher <IP>` for EC2 and `vagrant ssh` for Vagrant.

If you boot with the ISO the login is hard coded to **rancher/rancher**.  Only the ISO has the password hard coded.  If you run from a cloud or install to disk, SSH keys or a password of your choice is expected to be used.

## Persisting State

If you are running from the ISO RancherOS will be running from memory.  In order to persist to disk you need to format a file system with the label `RANCHER_STATE`.  For example

    docker run --privileged -it debian mkfs.ext4 -L RANCHER_STATE /dev/sda

## Installing to Disk

To install RancherOS on a new disk you can now use the `rancheros-install` command. 

For non-ec2 installs, before getting started create a cloud-init file that will provide your initial ssh keys. At a minimum something like:

```
 #cloud-config
 ssh_authorized_keys:
  - ssh-rsa AAA... user@rancher
```

See section below for current supported cloud-init functionality.

The command arguments are as follows:

```
Usage:
  rancheros-install [options]
Options:
  -c cloud-config file
      needed for SSH keys.
  -d device
  -f [ DANGEROUS! Data loss can happen ] partition/format without prompting
  -t install-type:
       generic
       amazon-ebs
  -v os-installer version.
  -h print this
 ```
 
 This command orchestrates installation from the rancher/os-installer container. 
 
####Examples:
 Virtualbox installation:
 
 `sudo rancheros-install -d /dev/sda -c ./cloud_data.yml -v v0.1.1 -t generic`
 
## Configuring

The entire state of RancherOS is controlled by a single configuration document.
You can edit the configuration with the `rancherctl config` command.  **Please note the configuration format is very much a work in progress and will most likely change in the early stages of RancherOS**

Sample configuration

```yaml
dns: [8.8.8.8, 8.8.4.4]
state:
  fstype: auto
  dev: LABEL=RANCHER_STATE
  required: false
userdocker:
  use_tls: true
  tls_server_cert: |-
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  tls_server_key: |-
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
  tls_ca_cert: |-
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
system_docker_args: [docker, -d, -s, overlay, -b, none, --restart=false, -H, 'unix:///var/run/system-docker.sock']
cloud_init:
  datasources:
  - configdrive:/media/config-2
ssh:
  keys:
    ecdsa: |-
      -----BEGIN DSA PRIVATE KEY-----
      ...
    ecdsa-pub: ssh-dss  AAAAB3NzaC1k.... root@rancher
enabledAddons:
- ubuntu-console

```

## Cloud Init

We currently support a very small portion of cloud-init.  If the user_data is a script (starting with the proper #!<interpreter>) we will execute it.  If the user_data starts with `#cloud-config` it will be processed by cloud-init.  The below directives are supported.

```yaml
#cloud-config

ssh_authorized_keys:
  - ssh-rsa AAA... darren@rancher

write_files:
  write_files:
  - path: /opt/rancher/bin/start.sh
    permissions: 0755
    owner: root
    content: |
      #!/bin/bash
      echo "I'm doing things on start"

```

## Useful Commands

Command | Description
--------|------------
`docker` | Good old Docker, use that to run stuff.
`system-docker` | The docker instance running the system containers.  Must run as root or using `sudo`
`rancherctl` | Control and configure RancherOS

## Customizing the console

Since RancherOS is so small the default console is based off of Busybox.  This it not always the best experience.  The intention with RancherOS is to allow you to swap out different consoles with something like Ubuntu, Fedora, or CentOS.  Currently we have Ubuntu configured but we will add more.  To enable the Ubuntu console do the following.

    sudo rancherctl addon enable ubuntu-console

Run the above but with `disable` to turn it off.  Currently you have to reboot the system to enable the new console.  I the future it will be dynamic and just require you to log out and back in.

### Console is ephemeral

The console (and all system containers) are ephemeral.  This means on each reboot of the system all changes to the console are lost.  Any changes in `/home` or `/opt` will be persisted though.  Additionally, on startup of the console container, if `/opt/rancher/bin/start.sh` exists, it will be executed.  You can add anything to that script to configure your console the way you want it.

In the future we will allow one to provide a custom image for the console container, but we just haven't gotten around yet to enabling that.

## Building

Docker 1.5+ required.

    ./build.sh

When the build is done the ISO should be in `dist/artifacts`

## Developing

Development is easiest done with QEMU on Linux.  If you aren't running Linux natively then we recommend you run VMware Fusion/Workstation and enable VT-x support.  Then, QEMU (with KVM support) will run sufficiently fast inside a Linux VM.

First run `./build.sh` to create the initial bootstrap Docker images.  After that if you make changes to the go code only run `./scripts/build`.  To launch RancherOS in QEMU from your dev version run `./scripts/run`.  You can SSH in using `ssh -l rancher -p 2222 localhost`.  Your SSH keys should have been populated so you won't need a password.  If you don't have SSH keys then the password is "rancher".

#License
Copyright (c) 2014-2015 [Rancher Labs, Inc.](http://rancher.com)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

