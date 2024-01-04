# Summary
Starting with Ubuntu 22.04 remote vm as lab device.  To run devstack/ovn-bgp lab setup uses vagrant with libvirt.  https://vagrant-libvirt.github.io/vagrant-libvirt/

* Log in to your Ubuntu 22.04 base machine
* install libvirt and other requirements

`sudo apt install libvirt-clients libvirt-daemon-system libvirt-dev ebtables`

* install vagrant

`sudo apt install vagrant`

* clone repo and checkout branch

`
git clone https://github.com/ajlitzin/ovn-bgp-vagrants.git
cd ovn-bgp-vagrants
git checkout config-via-ansible
cd spine-leaf-test
`

* install python virtual env package, create a virtual environment, activate it and install ansible

`
sudo apt install python3.10-venv
python3 -m venv ~/venv/ansible-openstack
source ~/venv/ansible-openstack/bin/activate
pip3 install -r requirements.txt
`

* start up the vagrant vms.  Note that the cumulus vms will take 20+ minutes to fully come up before you can log into them or configure them with ansible.  Come back to those later after we install devstack on rack-1-host-1

`vagrant up`

* run ansible playbooks to configure ip interfaces and bgp

`ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory configure-base`

* Next we are going to take elements of this ovn-bgp [blog](https://ltomasbo.wordpress.com/2023/12/19/deploying-ovn-bgp-agent-with-devstack/) entry and the [devstack install guide](https://docs.openstack.org/devstack/latest/guides/single-vm.html) to install devstack on single node, rack-1-host-1.  Eventually we'll try to extend this devstack install to multiple compute node vms

* log into rack-1-host-1

` vagrant ssh rack-1-host-1`

* add vagrant user to sudoers file.  Add `vagrant ALL=(ALL) NOPASSWD:ALL`

`
sudo su
EDITOR=vi visudo
`

* make /opt/stack dir and change ownership

`
sudo mkdir /opt/stack
sudo chown vagrant:root /opt/stack
`

* clone ovn-bgp-agent repo

`
cd /opt/stack/
git clone https://opendev.org/openstack/ovn-bgp-agent
`

* clone devstack repo

`
cd
git clone https://opendev.org/openstack/devstack
`

* copy the default config file from the ovn-bgp-agent repo into the devstack repo

`
cp /opt/stack/ovn-bgp-agent/devstack/local.conf.sample devstack/local.conf
`

* make updates to local.conf

For example, enable Horizon by changing disable_service Horizon to enable_service Horizon
* install devstack

`
cd devstack
./stack.sh
`

You may run into errors during the install.  I often ran into GnuTLS errors while trying to clone openstack component repos.  These all appear to be transient in nature.

In short, the install scripts are great but can be a little flaky.  If you run into errors, they may be transient.  I recommend you try to unstack and then restack before you dig into any install error too deeply:

`
./unstack.sh
./stack.sh
`

Using the local.conf from the ovn-bgp-agent repo

## accessing Horizon
Let's assume you have started with a virtual machine as your base host and used vagrant to launch more virtual machine(s) to run devstack. Your local client will not have direct access to the vagrant vm that is hosting the Horizon web management portal.  In this case you will need a chain of ssh tunnels in order to load Horizon from your local machine.

local client -> base server -> vagrant vm hosting Horizon

one way is to start two separate ssh tunnels, the first from your local client toward your base server, the 2nd from the base server to the vm hosting Horizon.

By default Horizon listens on HTTP port 80.  We will listen locally on port 8080 and forward that on.

from client to base server:
ssh -L8080:localhost:8080 user@base-server

from base server to vagrant vm hosting Horizon:
ssh -L 8080:localhost:80 vagrant@10.255.1.233 -i ~/ovn-bgp-vagrants/spine-leaf-test/.vagrant/machines/rack-1-host-1/libvirt/private_key

Note: I'm not sure there is a good way to do this in one command with the -J flag or defining it in your ssh conf file because you then would need the vagrant machine private key local to your client server and this can change if you rebuild your vagrant vm.

Now in your browser on your local client you should be able to go to http://<ip or FQDN of base server>:8080/dashboard

## Notes when running with Ubuntu 22.04

The generic/ubuntu2204 Vagrant box has ipv6 disabled.  Devstack relies on IPv6 being enabled.  The updated Vagrant file has an ansible job that updates the vm to enable ipv6.  If IPv6 is disabled you will get a confusing permission denied error when the stack.sh script attempts to apply an IPv6 address to the br-ex bridge:

```
+lib/neutron_plugins/services/l3:_neutron_configure_router_v6:416  sudo ip -6 addr replace 2001:db8::2/64 dev br-ex
RTNETLINK answers: Permission denied
```

You can check if ipv6 is enabled by running:

```text
sysctl -a 2>/dev/null | grep disable_ipv6
```

If ipv6 is disabled you will see values of "1" for the output above, like this:

```text
vagrant@rack-1-host-1:~$ sysctl -a 2>/dev/null | grep disable_ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.br-ex.disable_ipv6 = 1
net.ipv6.conf.br-int.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.eth1.disable_ipv6 = 1
net.ipv6.conf.eth2.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.ovs-system.disable_ipv6 = 1
net.ipv6.conf.vagrant.disable_ipv6 = 1
net.ipv6.conf.virbr0.disable_ipv6 = 1
```

You can temporarily fix this manually via the following.  note that this method will NOT survive a restart of the vagrant vm:

`
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
`

## History - trying to run on Ubuntu 20.04

When using the ansible provider, make sure you are in your intended ansible environment (e.g. python venv) before you run vagrant up
The blog guide runs into trouble installing devstack on centos8.  The openstack devstack site only mentions support for Ubuntu and two other Linux distriubtions, but not centos.  So I decided to try and use ubuntu.  In the end I ran into the same error using Ubuntu.  It appears something in the pre-install script that Luis wrote is no longer working.  I am now attempting to just follow the devstack wiki to complete the install of devstack and then see if I can incorporate the blog settings.

I am attempting to get the hosts working with ubuntu.  The first issue I ran into is trying to use box vagrant/jammy64.  This was because that box doesn't support the provider libvirt.  Next i found generic/ubuntu2204.  Unfortunately this box does not work either- at least when running Vagrant 2.6. The box would download and attept to start it's config process, but reach a point where vagrant is trying to ssh into the box and fail.  this appears due to the version of vagrant that was installed on my host system which is running Ubuntu 20.04.  The packaged vagrant version included with 20.04 is 2.2.6 which doesn't support generic/ubuntu2204 because Vagrant 2.2.6 relies on RSA keys and the version of OpenSSH included with 22.04 disables them by default.  This is described [here|https://github.com/hashicorp/vagrant/pull/13179] .  Workarounds include manually installing a supported version of Vagrant, or changing my box to use Ubuntu 20.04 instead of 22.04.  for now I am choosing to use 20.04


* This is required for installs on Ubuntu

`
sudo apt install python3.8-distutils
`

* I ran the install_devstack master script from the blog which failed, but did produce an edited version of local.conf .  Following the devstack wiki i cloned the devstack repo.  Instead of hand editing a new local.conf, i just copied over the one that ahd been previously generated and ran the stack.sh script.  This errored out because it claimed it had not been tested on ubuntu Focal (20.04) and that if I wanted to continue I had to set the Force env var

`
export FORCE=yes
`

* Ubuntu 20.04 has some sort of TLS issue with either git or the curl library version git relies on.  If you run the stack.sh script vanilla you will run into errors like:

`+functions-common:git_timed:688            timeout -s SIGINT 0 git clone --no-checkout https://opendev.org/openstack/neutron.git /opt/stack/neutron
Cloning into '/opt/stack/neutron'...
error: RPC failed; curl 56 GnuTLS recv error (-54): Error in the pull function.
fatal: the remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed`

Strangely, you can often avoid these errors by turning on a debug flag for git.  so run the command like so:

`
GIT_CURL_VERBOSE=1 ./stack.sh
`

or for repated failures, turn off ssl verification (yuck):

`
git config --global http.sslVerify false
git config --global http.postBuffer 1048576000
`

* next issue is neutron didn't start 