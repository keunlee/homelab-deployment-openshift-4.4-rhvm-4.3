# Homelab Deployment - Openshift/OKD 4.4.x-4.5.x on RHEV/Ovirt 4.3.x
---

# Overview

The following developer notes outline and detail the steps necessary to get Openshift 4.4.x/4.5.x (or OKD 4.4.x/4.5.x) running on RHVM 4.3.9/4.3.10 (or Ovirt 4.3.9/4.3.10). 

These notes cater to individuals that are interested in building their own homelab servers. 

OKD is simply the open source upstream community distribution of Openshift. Where as Openshift Container Platform is a Red Hat supported product that has paid support granted by a subscription from Red Hat. 

Official installation documentation are as followed: 

**Openshift Installation** - https://docs.openshift.com/container-platform/4.4/installing/installing_rhv/installing-rhv-default.html 

**OKD Installation** - https://docs.okd.io/latest/installing/installing_rhv/installing-rhv-default.html

# The Rigg

"This is my 'rigg'. There are many like it, but this one is mine."

* Cable Modme
* Wireless Router
* 5 port switch
* Home Desktop PC - Jumpbox
* Raspberry PI 4 - DNS Server
  * 4gb RAM
  * 128gb sdcard
* Raspberry PI 4 - VPN Server
  * 4gb RAM
  * 128gb sdcard
* HP Proliant ML350P Gen8 - Hypervisor
  * 128gb RAM (8gb x 16)
  * 2xE5-2690 8-Cores
  * 1.5tb SAS (300gb x 5)
  * 1.5tb SSD (500gb x 3)
  
![](https://content.screencast.com/users/Keun/folders/Default/media/fada0c02-a25e-49fe-89ab-32c031515854/Screenshot%20from%202020-06-10%2015-50-11.png)  
  
# Pre-requisites

* Deployed and running RHVM 4.3+ (or Ovirt 4.3+)
* Enough hypervisor resources to accomodate: 
  * 3 Master Nodes
  * 3 Worker Nodes
  
  where: 
  
  1x Node = 16gb RAM, 4 vCPU, 100gb solid state volume

# Step-By-Step Installation

## Create DNS Entries

### Openshift/OKD DNS Guidelines

Follow the directions from official documentation (to a tee): 

https://docs.openshift.com/container-platform/4.4/installing/installing_rhv/installing-rhv-default.html#installing-rhv-preparing-the-network-environment_installing-rhv-default

### Sample Setup

Depending on your DNS server, you may need to add DNS entries a bit differently.

For this rigg, we are using the following: 
- Raspberry PI 4
- Pi-hole Network-wide Ad Blocking + DNS Server (https://pi-hole.net/)
  - To configure DNS and wild card entries see the following resoures: 
    - https://discourse.pi-hole.net/t/howto-using-pi-hole-as-lan-dns-server/533
    - https://qiita.com/bmj0114/items/9c24d863bcab1a634503

1. Login to your Pi-hole DNS server: `ssh root@pihole.thekeunster.local`
2. Create or add the following entries to the following file: `/etc/pihole/lan.list`

add the following contents: 

```bash
10.0.1.90       api.okd.thekeunster.local
10.0.1.90       api-int.okd.thekeunster.local
10.0.1.91       *.apps.okd.thekeunster.local
```

3. Create or add the following to enable wildcard entries to the following file: `/etc/dnsmasq.d/02-lan.conf`

add the following contents: 

```bash
addn-hosts=/etc/pihole/lan.list
address=/apps.okd.thekeunster.local/10.0.1.91
```

4. restart the DNS service. 

```bash
sudo pihole restartdns
```

5. validate your DNS

Validate single DNS entries: 

```bash
nslookup api.okd.thekeunster.local

# should yield
Server:         10.0.1.28
Address:        10.0.1.28#53

Name:   api.okd.thekeunster.local
Address: 10.0.1.90
```

```bash
nslookup api-int.okd.thekeunster.local

# should yield
Server:         10.0.1.28
Address:        10.0.1.28#53

Name:   api.okd.thekeunster.local
Address: 10.0.1.90
```

Validate wildcard entries:

```bash
nslookup test1.apps.okd.thekeunster.local
Server:         10.0.1.28
Address:        10.0.1.28#53

# should yield
Name:   test1.apps.okd.thekeunster.local
Address: 10.0.1.91
```

```bash
nslookup test2.apps.okd.thekeunster.local
Server:         10.0.1.28
Address:        10.0.1.28#53

# should yield
Name:   test2.apps.okd.thekeunster.local
Address: 10.0.1.91
```
## Setup a CA Certificate

Follow the directions from official documentation (to a tee): 

https://docs.openshift.com/container-platform/4.4/installing/installing_rhv/installing-rhv-default.html#installing-rhv-setting-up-ca-certificate_installing-rhv-default

ultimately, you will have added your ca in the following location: `/etc/pki/ca-trust/source/anchors/ca.pem`

## Generate an SSH private key

Follow the directions from official documentation (to a tee): 

https://docs.openshift.com/container-platform/4.4/installing/installing_rhv/installing-rhv-default.html#ssh-agent-using_installing-rhv-default

By doing the above, this simply allows you to be able to ssh directly into your cluster nodes from your jumpbox (the computer your using to install openshift from). 

```bash
# i.e. passwordless logins to your nodes

ssh core@10.0.1.51	# worker node 00
ssh core@10.0.1.52  # worker node 01
ssh core@10.0.1.53  # worker node 02
```
## Obtain Openshift/OKD

You can download release distributions from either one of the following resources (Openshift or OKD):

**Openshift Releases** - https://openshift-release.svc.ci.openshift.org/

**Openshift Latest Stable (4.5)** - https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.5/

**OKD Release** - https://origin-release.svc.ci.openshift.org/

For all intents and purposes, the installations between the two above are virtually the same. 

For this example, we'll be using __Openshift 4.5.8__. 

Depending on your where you'll be running your installer from, in this case a linux box, make sure to grab the right client and installer. 

for example: 

```bash
openshift-client-linux-4.5.8.tar.gz
openshift-install-linux-4.5.8.tar.gz
```

## Extract the Client

Extract the client and make accessible in your PATH

```bash
tar zxvf openshift-client-linux-4.5.8.tar.gz
sudo mv kubectl /usr/local/bin
sudo mv oc /usr/local bin
```

## Extract the Installer

Extract the installer to any location of your choosing (i.e. `/home/user1/workspace/ocp` )

```bash
tar zxvf openshift-install-linux-4.5.8.tar.gz
mv openshift-installer /home/user1/workpace/ocp
```

A few observations about the installer. During installation of your cluster the following installation artifacts may be created: 

- `~/.ovirt`
- `~/.cache/openshift-installer`

you can safely delete these anytime after your installation is complete. 


## Create the Installation Configuration

We will create an installation configuration file before creating the cluster.

### Obtain pull secret

Go here to obtain a pull secret. 

https://cloud.redhat.com/openshift/install/pull-secret

You will need this in order to create an installation configuration

### Create installation configuration

run the following: 

```bash
./openshift-install create install-config --dir=install --log-level=debug
```

[![](http://img.youtube.com/vi/yO4-AYubgKY/0.jpg)](https://www.youtube.com/watch?v=yO4-AYubgKY)

This command will do the following: 
- ultimately will create an installation cofiguration file, inside of the specified installation directory, specified by argument `--dir`. The install directory will be created if it doesn't exist.  

```bash
.
├── install
│   └── install-config.yaml
```
- will create a hidden folder in your home directory: `~/.ovirt`. If for any reason you've foo-barred your installation config file, you can start fresh by deleting this directory. as well as the `install` directory. 

During the creation of the installation config, you may be asked to enter the contents of the cert you pulled down from RHVM 4.3 (or Ovirt 4.3). If so, you can enter the contents of that cert here. Otherwise, you can opt out. 

Upon successful creation of the installation configuration, open and examine the contents of the file `~/.ovirt/ovirt-config.yaml`. If you do not have the `ovirt_ca_bundle` parameter as shown below, add it. It will look similar to the following in it's entirety. 

```yaml
ovirt_url: https://labs.thekeunster.local/ovirt-engine/api
ovirt_username: admin@internal
ovirt_password: super-duper-password
ovirt_insecure: true
ovirt_ca_bundle: |
  -----BEGIN CERTIFICATE-----
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKoZIhvcNAQELBQAwUDELMAkGA1UEBhMCVVMxGjAYBgNVBAoM
  MIID2zCCAsOgAwIBAgICEAAwDQYJKo==
  -----END CERTIFICATE-----
```

the contents of the `ovirt_ca_bundle` parameter will come from the contents of the file: `/etc/pki/ca-trust/source/anchors/ca.pem`

## Create image template (optional)

The following will allow you to use a custom image template instead of the default 8gb templates. You may want to create a custom template to create nodes with larger disk sizes, allocate more RAM, and/or allocate more vCPUs per cluster master/worker node. 

### Read How

Read more about how to do this (scroll to heading "Customizing RHCOS template" pg. 13): https://access.redhat.com/sites/default/files/attachments/quickstart_guide_for_installing_ocp_on_rhv_1.4.pdf

### See How

In the video we start off with the intial template: `okd-jnw4r-rhcos`

And finish off by creating a new template: `ocp-template-50gb`

With the new template, master and worker nodes will have the following: 

- 50gb storage
- 16gb ram
- 4 vCPUs

make sure to set the new template in the active installation terminal before deploying the cluster. 

```bash
export OPENSHIFT_INSTALL_OS_IMAGE_OVERRIDE=ocp-template-50gb
```


[![](http://img.youtube.com/vi/kBOOfphT-Kw/0.jpg)](https://www.youtube.com/watch?v=kBOOfphT-Kw)

## Deploy Cluster

Run the following: 

```bash
./openshift-install create cluster --dir=install --log-level=debug
```

if all goes well, when this finishes you will have a running openshift cluster. 

## Post Operations

### Setup Persistent Storage

TODO

## Troubleshooting

### cluster deploy times out during bootstrapping

the following command will add an additional 30 minutes, to wait for the install to complete. you should not need to do this more then once. 

```bash
./openshift-install wait-for install-complete --dir=install --log-level=debug

```

### need to run a clean installation

run the following command: 

```bash
./openshift-install destroy cluster --dir=install --log-level=debug
```

delete the following directories: 

`~/.ovirt`

`~/.cache/openshift-installer`

in RHVM/Ovirt, delete the following: 
- all VMs created in previous install
- all templates created in previous install (only if you want to use a new template)