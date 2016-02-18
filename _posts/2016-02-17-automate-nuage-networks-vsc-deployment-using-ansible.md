---
layout: post
title: "Automate Nuage Networks VSC deployment using Ansible"
modified: 2016-02-17 16:22:25 -0800
tags: [sdn,ansible,nuage,vsc]
image:
  feature:
  credit:
  creditlink:
comments:
share: "Automate Nuage Networks VSC deployment using Ansible"
---
Today, I have released an **Ansible** role to automate the deployment of **Nuage Networks Virtualized Services Controller** (VSC)
on **KVM** hypervisors. The goal of this role is to install and configure one or several Nuage Networks VSCs in a few minutes.
For those of you who don't know what is a **Nuage Networks VSC**, here is a brief description.  
The **Virtualized Services Controller** is a **SDN controller** developed by **Nuage Networks**, it functions as a network control plane for the Nuage Network Virtualized Services Platform (VSP). This component of the Nuage Networks VSP supports several mode of deployments: standalone or clustered (2 to N). It is recommended to use a clustered deployment for production environment. The VSC cluster rely on MP-BGP (Multi Protocol BGP) to exchange information and keep the cluster synchronized. More information about the Nuage Networks VSP is available on the [Nuage Networks Official Website](http://www.nuagenetworks.net/products/virtualized-services-platform/).

Some of you may have already had the chance to play with VSC deployments on KVM, I am sure you remember that you had to do many steps to install and configure each controller (modify the VSC libvirt XML template, configure the VSC using the console, attach the VSC to the bridges, ...).  
To help you (and me :D) deploy a Nuage Networks VSC on KVM, I have developed an Ansible role: [**nuage-vsc-intaller**](https://galaxy.ansible.com/rvichery/nuage-vsc-installer)

For people who are not familiar with Ansible, I could recommend [https://www.ansible.com/how-ansible-works](https://www.ansible.com/how-ansible-works) as a quick way to get yourself familiar with some of the concepts such as inventory, playbooks and roles.

I will demonstrate below how to use the **nuage-vsc-installer**. It can be used for both **standalone** VSC installations or **clustered** ones (i.e. using BGP federation).

##Setting up your deployment environment

The first step is to install **Ansible**. Steps have been documented for a CentOS 7 environment but you can use any operating system supported by Ansible.
On your **CentOS 7 terminal**, install the **epel** repository and the **Ansible** package.

{% highlight shell %}
[centos@nuage-vsc-installer-post ~]$ sudo yum install http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
...
...
Complete!
[centos@nuage-vsc-installer-post ~]$ sudo yum install -y ansible
...
...
Complete!
{% endhighlight %}

If you want to install Ansible on Mac OSX, Ubuntu or any other operating system, please take a look at http://docs.ansible.com/ansible/intro_installation.html

Now, in order to be able to use the **Nuage VSC installer** role in your ansible playbooks, you have to install the **nuage-vsc-installer** role using **Ansible Galaxy**.  

{% highlight shell %}
[centos@nuage-vsc-installer-post ~]$ sudo ansible-galaxy install rvichery.nuage-vsc-installer
- downloading role 'nuage-vsc-installer', owned by rvichery
- downloading role from https://github.com/rvichery/nuage-vsc-installer/archive/master.tar.gz
- extracting rvichery.nuage-vsc-installer to /etc/ansible/roles/rvichery.nuage-vsc-installer
- rvichery.nuage-vsc-installer was installed successfully
{% endhighlight %}

Your deployment environment is ready. That's was quick isn't it ?!  
But before deploying a VSC using the installer, make sure to:

* have access to a KVM hypervisor with some linux bridges already configured.

* download the Nuage Networks VSC from the Nuage Networks support website. The Nuage Networks VSC is a licensed software and is not included in the Ansible role. Contact a Nuage Networks representative that can help you download the VSC qcow2 image.

* **read the [README file](https://galaxy.ansible.com/rvichery/nuage-vsc-installer/#readme) on Ansible Galaxy**

In the below examples, hypervisors are configured with two bridges: **br-mgmt** and **br-ctl** which carry respectively management (bof) and control traffic.

![Hypervisor network configuration]({{ site.url }}/images/2016-02-17-automate-nuage-networks-vsc-deployment-using-ansible_1.png)

##Example 1: Deploying a standalone VSC

In this first example, I will show you how to deploy a standalone Nuage Networks VSC on a single KVM hypervisor.

![VSC standalone deployment]({{ site.url }}/images/2016-02-17-automate-nuage-networks-vsc-deployment-using-ansible_3.png)

First, let’s create a new directory for your standalone deployment playbook.

{% highlight shell %}
[centos@nuage-vsc-installer-post ~]$  mkdir ansible-vsc-standalone
[centos@nuage-vsc-installer-post ~]$ cd  ansible-vsc-standalone
[centos@nuage-vsc-installer-post ansible-vsc-standalone]$
{% endhighlight %}

In this new directory, create a new **hosts** file that will be used by **Ansible** as an **inventory**.  
Inside **hosts**:

<figure class="lineno-container">
{% highlight yaml linenos %}
[vsc]
vsc_test01.nuage.lab
{% endhighlight %}
</figure>

Then, create the playbook YAML file.  
Using your favorite text editor create a new file **vsc-standalone-deployment.yml**.  
Inside **vsc-standalone-deployment.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
- hosts: vsc
  user: root
  gather_facts: no
  roles:
    - rvichery.nuage-vsc-installer
{% endhighlight %}
</figure>

The Ansible role requires to configure some variables for each VSC, so create a new directory **host_vars** to store the VSC information.  

{% highlight shell %}
[centos@nuage-vsc-installer-post ansible-vsc-standalone]$ mkdir host_vars
[centos@nuage-vsc-installer-post ansible-vsc-standalone]$ cd host_vars/
[centos@nuage-vsc-installer-post host_vars]$
{% endhighlight %}

Then, create a new configuration file for your standalone VSC, the filename **must** match the hostname in the hosts file (i.e. vsc_test01.nuage.lab in this example).  
Inside **host_vars/vsc_test01.nuage.lab.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
hypervisor: hypervisor01.nuage.lab
interfaces:
  mgmt:
    linux_bridge: br-mgmt
    ip: 10.21.1.40
    netmask_prefix: 24
  control:
    linux_bridge: br-ctl
    ip: 10.22.1.40
    netmask_prefix: 24
dns:
  servers:
    - 10.21.0.251
    - 10.21.0.252
  domain: nuage.lab
system_ip: 1.1.1.1
ntp_servers:
  - 10.21.0.251
  - 10.21.0.252
vsd_fqdn: “vsd.nuage.lab"
{% endhighlight %}
</figure>

Upload the **vsc_singledisk.qcow2** image on your deployment server and add the **absolute path** of the image (vsc_qcow2 variable) in **vsc-standalone-deployment.yml**.  
Inside **vsc-standalone-deployment.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
- hosts: vsc
  user: root
  gather_facts: no
  vars:
    vsc_qcow2: /home/centos/3.2r6/single_disk/vsc_singledisk.qcow2
  roles:
    - rvichery.nuage-vsc-installer
{% endhighlight %}
</figure>

Finally, launch your playbook using **ansible-playbook**.

{% highlight shell %}
[centos@nuage-vsc-installer-post ansible-vsc-standalone]$ ansible-playbook -i hosts vsc-standalone-deployment.yml
PLAY [vsc] ********************************************************************

TASK: [rvichery.nuage-vsc-installer | Pull facts on hypervisor] ***************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Install additional packages to modify VSC qcow image and run VSC as VM - RedHat Host] ***
skipping: [vsc_test01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Install additional packages to modify VSC qcow image and run VSC as VM - Debian Host] ***
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=qemu-kvm,libvirt-bin,bridge-utils,libguestfs-tools,python-libvirt)

TASK: [rvichery.nuage-vsc-installer | List the Virtual Machine] ***************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | fail msg="The VM {{ inventory_hostname }} is already defined on this hypervisor."] ***
skipping: [vsc_test01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Include OS-specific variables.] *********
ok: [vsc_test01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Creates VSC directory] ******************
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Copy over VSC qcow2 image] **************
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Setup VSC temporary configuration files] ***
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=bof.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=config.cfg)

TASK: [rvichery.nuage-vsc-installer | Copy temporary configuration files to the VSC image] ***
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=bof.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=config.cfg)

TASK: [rvichery.nuage-vsc-installer | Remove temporary configuration files] ***
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=bof.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=config.cfg)

TASK: [rvichery.nuage-vsc-installer | Get a list of VMs] **********************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Define VSC guest VM] ********************
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Start VSC guest VM] *********************
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | get guest info] *************************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | assert ] ********************************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

PLAY RECAP ********************************************************************
vsc_test01.nuage.lab       : ok=14   changed=7    unreachable=0    failed=0   
[centos@nuage-vsc-installer-post ansible-vsc-standalone]$
{% endhighlight %}

Your VSC has been deployed and configured, check in your Nuage Networks VSD that the new VSC appears.

![vsc_test01 VSD view]({{ site.url }}/images/2016-02-17-automate-nuage-networks-vsc-deployment-using-ansible_5.png){: .center-image }

##Example 2: Deploying a cluster of VSCs

This example assumes you have two KVM hypervisors available, one for each VSC.
![VSC clustered deployment - two hypervisors]({{ site.url }}/images/2016-02-17-automate-nuage-networks-vsc-deployment-using-ansible_2.png)

The cluster of VSCs can be deployed on a single KVM node but I don't recommended to do that in production.
![VSC clustered deployment - one hypervisor]({{ site.url }}/images/2016-02-17-automate-nuage-networks-vsc-deployment-using-ansible_4.png)

First, let’s create a new directory for your **clustered** deployment playbook.  

{% highlight shell %}
[centos@nuage-vsc-installer-post ~]$  mkdir ansible-vsc-clustered
[centos@nuage-vsc-installer-post ~]$ cd  ansible-vsc-clustered
[centos@nuage-vsc-installer-post ansible-vsc-clustered]$
{% endhighlight %}

In this new directory, create a new **hosts** file that will be used by **Ansible** as an **inventory**.  
Both VSCs must be part of the **same group** to automatically configure the **BGP** peering. If you want to use another group name, you have to overwrite the **list_of_vscs** variable (see [README file](https://galaxy.ansible.com/rvichery/nuage-vsc-installer/#readme)).  
Inside **hosts**:

<figure class="lineno-container">
{% highlight yaml linenos %}
[vsc]
vsc_test01.nuage.lab
vsc_test02.nuage.lab
{% endhighlight %}
</figure>

Then, create the playbook YAML file.  
Using your favorite text editor create a new file **vsc-clustered-deployment.yml**.  
Inside **vsc-clustered-deployment.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
- hosts: vsc
  user: root
  gather_facts: no
  roles:
    - nuage-vsc-installer
    {% endhighlight %}
    </figure>

The Ansible role requires to configure some variables for each VSC, so create a new directory **host_vars** to store all your VSC information.

{% highlight shell %}
[centos@nuage-vsc-installer-post ansible-vsc-clustered]$ mkdir host_vars
[centos@nuage-vsc-installer-post ansible-vsc-clustered]$ cd host_vars/
[centos@nuage-vsc-installer-post host_vars]$
{% endhighlight %}

Then, create a new file configuration file for each VSC, the filenames **must** match the hostnames in the hosts file (i.e. vsc_test01.nuage.lab.yml and vsc_test02.nuage.lab.yml in this example).  
Inside **host_vars/vsc_test01.nuage.lab.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
hypervisor: hypervisor01.nuage.lab
interfaces:
  mgmt:
    ip: 10.21.1.40
  control:
    ip: 10.22.1.40
system_ip: 1.1.1.1
{% endhighlight %}
</figure>

Inside **host_vars/vsc_test02.nuage.lab.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
hypervisor: hypervisor02.nuage.lab
interfaces:
  mgmt:
    ip: 10.21.1.41
  control:
    ip: 10.22.1.41
system_ip: 1.1.1.2
{% endhighlight %}
</figure>

As some variables are the same for both Nuage VSCs, create a new folder and a new file **group_vars/vsc.yml** in your playbook directory and add the redundant variables.  
These variables are defined at the group level.  
Create the directory **group_vars** first, then create the configuration file **vsc.yml**.  

{% highlight shell %}
[centos@nuage-vsc-installer-post ansible-vsc-clustered]$ mkdir group_vars
[centos@nuage-vsc-installer-post ansible-vsc-clustered]$ cd group_vars/
[centos@nuage-vsc-installer-post group_vars]$
{% endhighlight %}

Inside **group_vars/vsc.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
interfaces:
 mgmt:
   linux_bridge: br-mgmt
   netmask_prefix: 24
 control:
   linux_bridge: br-ctl
   netmask_prefix: 24
dns:
 servers:
   - 10.21.0.251
   - 10.21.0.252
 domain: nuage.lab
ntp_servers:
 - 10.21.0.251
 - 10.21.0.252
vsd_fqdn: “vsd.nuage.lab"
{% endhighlight %}
</figure>

Upload the **vsc_singledisk.qcow2** image on your deployment server and add the **absolute path** of the image (vsc_qcow2 variable) in **vsc-standalone-deployment.yml**  
Inside **vsc-clustered-deployment.yml**:

<figure class="lineno-container">
{% highlight yaml linenos %}
- hosts: vsc
  user: root
  gather_facts: no
  vars:
    vsc_qcow2: /home/centos/3.2r6/single_disk/vsc_singledisk.qcow2
  roles:
    - rvichery.nuage-vsc-installer
{% endhighlight %}
</figure>

Finally, launch your playbook using **ansible-playbook** .

{% highlight shell %}
[centos@nuage-vsc-installer-post ansible-vsc-clustered]$ ansible-playbook -i hosts vsc-standalone-deployment.yml

PLAY [vsc] ********************************************************************

TASK: [rvichery.nuage-vsc-installer | Pull facts on hypervisor] ***************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]
ok: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Install additional packages to modify VSC qcow image and run VSC as VM - RedHat Host] ***
skipping: [vsc_test01.nuage.lab]
skipping: [vsc_test02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Install additional packages to modify VSC qcow image and run VSC as VM - Debian Host] ***
ok: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=qemu-kvm,libvirt-bin,bridge-utils,libguestfs-tools,python-libvirt)
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=qemu-kvm,libvirt-bin,bridge-utils,libguestfs-tools,python-libvirt)

TASK: [rvichery.nuage-vsc-installer | List the Virtual Machine] ***************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]
ok: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | fail msg="The VM {{ inventory_hostname }} is already defined on this hypervisor."] ***
skipping: [vsc_test02.nuage.lab]
skipping: [vsc_test01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Include OS-specific variables.] *********
ok: [vsc_test01.nuage.lab]
ok: [vsc_test02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Creates VSC directory] ******************
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Copy over VSC qcow2 image] **************
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Setup VSC temporary configuration files] ***
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=bof.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=bof.cfg)
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=config.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=config.cfg)

TASK: [rvichery.nuage-vsc-installer | Copy temporary configuration files to the VSC image] ***
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=bof.cfg)
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=bof.cfg)
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=config.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=config.cfg)

TASK: [rvichery.nuage-vsc-installer | Remove temporary configuration files] ***
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=bof.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=bof.cfg)
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab] => (item=config.cfg)
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab] => (item=config.cfg)

TASK: [rvichery.nuage-vsc-installer | Get a list of VMs] **********************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]
ok: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Define VSC guest VM] ********************
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | Start VSC guest VM] *********************
changed: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]
changed: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | get guest info] *************************
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]
ok: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]

TASK: [rvichery.nuage-vsc-installer | assert ] ********************************
ok: [vsc_test02.nuage.lab -> hypervisor02.nuage.lab]
ok: [vsc_test01.nuage.lab -> hypervisor01.nuage.lab]

PLAY RECAP ********************************************************************
vsc_test01.nuage.lab       : ok=14   changed=7    unreachable=0    failed=0   
vsc_test02.nuage.lab       : ok=14   changed=7    unreachable=0    failed=0   

[centos@nuage-vsc-installer-post ansible-vsc-standalone]$
{% endhighlight %}

By now, your VSCs have been deployed and configured, check in your Nuage Networks VSD that both VSCs appear.

If you want to customize your VSC configuration, take a look at the README file [here](https://github.com/rvichery/nuage-vsc-installer/) to have a complete list of variables (AS number, xmpp credentials, kvm emulator, images path,…) that can be set in your playbooks.

You are more than welcome to contribute to this project on Github [https://github.com/rvichery/nuage-vsc-installer](https://github.com/rvichery/nuage-vsc-installer) or report issues [https://github.com/rvichery/nuage-vsc-installer/issues](https://github.com/rvichery/nuage-vsc-installer/issues). Please use the pull request mechanism to suggest changes.

I hope you found this post interesting and useful!

Thanks to [@jonas](https://github.com/jonasvermeulen) and [@philippe](https://github.com/pdellaert) for their help!
