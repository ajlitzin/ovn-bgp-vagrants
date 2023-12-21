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

* clone devstack repo

`
git clone https://opendev.org/openstack/devstack
cd devstack
`

* copy the default config file

`
 cp samples/local.conf .
`

* install devstack.  note I still had to add the GIT_CURL_VERBOSE env var or else it bombs out trying to clone openstack repos with
the GNU TLS error

`
GIT_CURL_VERBOSE=1 ./stack.sh
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