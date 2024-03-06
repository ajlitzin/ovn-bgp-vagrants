# Summary

Starting with Ubuntu 22.04 remote vm as lab device.  To run devstack/ovn-bgp lab setup uses [vagrant with libvirt]( https://vagrant-libvirt.github.io/vagrant-libvirt/)

* Log in to your Ubuntu 22.04 base machine

* Update packages with apt

`sudo apt update`

* Install libvirt and other requirements

`sudo apt install -y libvirt-clients libvirt-daemon-system libvirt-dev ebtables`

* Install vagrant

`sudo apt install -y vagrant`

* Install git

`sudo apt install -y git`

* Clone repo and checkout branch

```bash
git clone https://github.com/ajlitzin/ovn-bgp-vagrants.git
cd ovn-bgp-vagrants
git checkout multi-node-devstack
cd spine-leaf-test
```

* Install python virtual env package.  Note you may need to update package version.  You can find out which version to install by trying to run `python3 -m venv test-venv` first and if it should fail it should tell you which venv python package to install. e.g.:
Note this is assuming python3.10 version

`
sudo apt install -y python3.10-venv
`

* Create a virtual environment. You should only need to do this step once.

```bash
python3 -m venv ~/venv/ansible-openstack
```

* Activate the virtual env.  You will need to do this once per new terminal session before you can run ansible playbooks. and install ansible.

```bash
source ~/venv/ansible-openstack/bin/activate
```

* Install ansible into the virtual env

```bash
pip3 install -r requirements.txt
```

* Start up the vagrant vm to use as the Devstack node. Vagrant reads the Vagrantfile to know how to configure each defined virtual machine and how to connect their interfaces.  Most of the original Vagrantfile is from Luis Tomas, but I have modified it to use Ubuntu 22.04 and to include a no-op ansible play who's sole purpose is to trigger vagrant to create an ansible inventory file for the vms it creates.  Later we will uses this inventory file to configure our vagrant vms.  (!) Sometimes if you vagrant down a vm and then bring it back up vagrant gives the vm a new IP, but doesn't upgrade the inventory file.  In those cases you must update the IP in the inventory yourself (after logging into the vm to find out what it is.)

`vagrant up rack-1-host-1`

* If the above errors out with a permission denied error trying to access /var/run/libvirt/libvirt-sock then you should try rebooting your server and trying again.  It may be helpful to follow this suggestion, but this didn't help me, at least not without a restart too. [Troubleshoot permission denied](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-troubleshooting-common_libvirt_errors_and_troubleshooting#sect-The_URI_failed_to_connect_to_the_hypervisor-permission_denied)

* Test logging into the vagrant vm to use as the Devstack node.

`vagrant ssh rack-1-host-1`

* Assuming the vagrant ssh works, exit and return to your base vm inside your python virtual environment

`vagrant@rack-1-host-1~$ exit`

* Run ansible play to configure rack-1-host-1.  This will add network interface configuration for the loopback interface and the eth1 and eth2 uplinks to the leaf switches.  Note that only eth1 is in use as of now.  It also prepares the server for devstack installation by creating the target directory, clones the ovn-bgp-agent and devstack repos, and copies over a modified version of the devstack local.conf config file.

```bash
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory configure-cumulus.yaml -l rack-1-host-1 --diff
```

* Next we are going to take elements of this ovn-bgp [blog](https://ltomasbo.wordpress.com/2023/12/19/deploying-ovn-bgp-agent-with-devstack/) entry and the [Devstack install guide](https://docs.openstack.org/devstack/latest/guides/single-vm.html) to install Devstack on the controller node, rack-1-host-1.  Once the controller is built we'll build a 2nd compute node, install Devtack and configure it.

* Log into rack-1-host-1

`vagrant ssh rack-1-host-1`

* Install Devstack.  Note that this is a pretty involved set of shell scripts that I'm not sure would run well via ansible playbook

```bash
cd devstack
./stack.sh
```

You may run into errors during the install.  I often ran into GnuTLS errors while trying to clone openstack component repos.  These all appear to be transient in nature.

In short, the install scripts are great but can be a little flaky.  If you run into errors, they may be transient.  I recommend you try to unstack and then restack before you dig into any install error too deeply:

```bash
./unstack.sh
./stack.sh
```

* Validate ovn-bgp-agent operation

ovn-bgp-agent for Devstack installs FRR for you with an opinated config that relys upon the default openstack network configuration.  After Devstack install completes you should see that FRR is running and be able to interact with it via vtysh.  However, the configuration doesn't include any upstream bgp peers so validating that ovn-bgp-agent is impacting BGP route advertisement is difficult at this point.

You can validate the changes that ovn-bgp-agent makes to kernel interfaces when it detects events from the Southbound OVN database.  FRR's BGP config has a policy to redistribute connected networks and a policy restricting route advertisements to host routes (/32, /128).  This Devstack build creates a dummy inteface named bgp-nic on a vrf named bgp-vrf and that is what FRR is using to know which connected routes to distribute.

Note that in the running config of FRR you can see that the BGP definition is tied to bgp-vrf, e.g. `router bgp 64999 vrf bgp-vrf`. This is important to the config and how it ties to the kernel routing, but you may notices that is not present in the [frr.conf](https://opendev.org/openstack/ovn-bgp-agent/src/branch/master/etc/frr/frr.conf) file that is included in the ovn-bgp-agent Devstack.  The ovn-bgp-agent checks for its existance and adds if if it's missing via calls to functions in [frr.py](https://opendev.org/openstack/ovn-bgp-agent/src/branch/stable/2023.2/ovn_bgp_agent/drivers/openstack/utils/frr.py)

Using the default ovn-bgp-agent config the agent will react to events that include virtual machines being attached to public network or ports and floating IPs.  To prove this you can spin up instances and attach them to those resources and then check that the /32 IP address ends up attached to the bgp-nic kernel interface:

`ip addr show bgp-nic`

If you follow the supplemental instructions in the [Luis Tomas blog entry](https://ltomasbo.wordpress.com/2023/12/19/deploying-ovn-bgp-agent-with-devstack/) to enable exposing hosts on tenant self-service networks you can also spin up a private network and attach a virtual machine to it and it's /32 address should also appear in the output of `ip addr show bgp-nic`

* Adding an upstream Leaf switch

It's much more interesting to prove that BGP route advertising actually works than to assume it works by watching the changes to the bgp-nic interface and trusting that the configuration of the FRR on the host would properly advertise to an upstream peer if it had one.  Let's take advantage of the fact that the repo this is based on was intended to spin up a mini-lab which included a leaf switch upstream to the host.

Our local vagrant vm for rack-1-host-1 includes two interfaces that are virtually cabled to the virtual leafs in rack1, eth1 and eth2.  We need to configure the interfaces and make some small changes to the FRR conf file to take advantage of them and configure BGP to neighbor with the upstream leaf switches.  Fortunately we already have an ansible playbook to accomplish this.

* From your python virtual env on your host machine, run the ansible playbook to configure rack-1-host-1 to use rack-1-leaf-1 as a BGP neighbor

`
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory configure-cumulus.yaml -l rack-1-host-1 --tags frr --diff
`

* Configure rack-1-leaf-1

From your earlier python virtual env on your host machine, spin up the vagrant vm for rack-1-leaf-1.  Note that it takes a looong time, sometimes 20 minutes, for it to completely come online so be patient.  Once you can ssh into rack-1-leaf-1 use ansible to configure it.

`vagrant up rack-1-leaf-1`

* Use ansible to configure rack-1-leaf-1

From within your python virtual env on your host machine, run the configuration playbook against the leaf:

`
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory configure-cumulus.yaml -l rack-1-leaf-1
`

* Check BGP status and routes

Once the configuration completes you should see rack-1-host-1 and rack-1-leaf-1 form a BGP session.  Check its status and routes exchanged.  You should see that every host IP bound to the bgp-nic interface of rack-1-host-1 becomes a /32 BGP route advertised from rack-1-host-1 to rack-1-leaf-1

```bash
sudo vtysh
show ip route bgp
```

* Add iptables SNAT rule

The Geneve tunnels will use the loopback as destinations, but they also need to use the loopback IP as their src IP when communicating with the remote Geneve TEP.  We use an iptables rule to force this to happen.  (!) This was not in Luis's blog, but I found my Geneve tunnels would not come up if I didn't do it.

```bash
sudo iptables -t nat -A POSTROUTING -o eth1 -d 99.99.1.2 -p all -j SNAT --to 99.99.1.1
```

* Cycle the interface

May not be strictly required, but I did have at least one occastion where the server ignored the NAT rule until I did this.

```bash
sudo ip link set dev eth1 down
sudo ip link set dev eth1 up
```

* Check status of Geneve tunnel

It should show that state=down because we haven't configured the 2nd node yet

```bash
sudo ovs-vsctl show | grep geneve -C 3
```

You should see something like this:

```bash
vagrant@rack-1-host-1:~$ sudo ovs-vsctl show | grep geneve -C 3
            Interface tapa4c4cc51-79
        Port ovn-16e9c0-0
            Interface ovn-16e9c0-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="99.99.1.2"}
                bfd_status: {diagnostic="Control Detection Time Expired", flap_count="11", forwarding="true", remote_diagnostic="Neighbor Signaled Session Down", remote_state=down, state=down}
        Port tapb356b4aa-29
```

* Begin steps to add 2nd compute node, rack-1-host-2

* Start up the vagrant vm to use as the second Devstack compute node.

`vagrant up rack-1-host-2`

* Test logging into the vagrant vm to use as the Devstack node.

`vagrant ssh rack-1-host-2`

* Assuming the vagrant ssh works, exit and return to your base vm inside your python virtual environment

`vagrant@rack-1-host-2~$ exit`

* Run ansible play to configure rack-1-host-2

```bash
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory configure-cumulus.yaml -l rack-1-host-2 --diff
```

* Log into rack-1-host-2

`vagrant ssh rack-1-host-2`

* Add iptables SNAT rule

```bash
sudo iptables -t nat -A POSTROUTING -o eth1 -d 99.99.1.1 -p all -j SNAT --to 99.99.1.2
```

* Cycle the interface

May not be strictly required, but I did have at least one occastion where the server ignored the NAT rule until I did this.

```bash
sudo ip link set dev eth1 down
sudo ip link set dev eth1 up
```

* Install Devstack. Troubleshoot as necessary using notes from earlier Devstack install on the first host.

```bash
cd devstack
./stack.sh
```

* Validate ovn-bgp-agent operation

Use the same methods you used on the intial node

* From controller node, use tool to map new compute node to nova cell.

Log back into the controller node, e.g. rack-1-host-1

* Source OpenStack
  
This will load the environment required to run the OpenStack CLI commands.
From your controller vm, rack-1-host-1:

```bash
cd /opt/stack/devstack
source openrc admin demo
```

* Run script to add new compute node to the cell

```bash
tools/discover_hosts.sh
```

* Check status of Geneve tunnel

Now it should show state=up because we both endpoints are configured

```bash
sudo ovs-vsctl show | grep geneve -C 3
```

You should see something like this:

```bash
vagrant@rack-1-host-1:~$ sudo ovs-vsctl show | grep geneve -C 3
            Interface tapa4c4cc51-79
        Port ovn-16e9c0-0
            Interface ovn-16e9c0-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="99.99.1.2"}
                bfd_status: {diagnostic="Control Detection Time Expired", flap_count="11", forwarding="true", remote_diagnostic="Neighbor Signaled Session Down", remote_state=up, state=up}
        Port tapb356b4aa-29
```

* Use the Openstack CLI from controller node to Launch vms
Alternatively you could use the Horizon dashboard
  * Source OpenStack
  
This will load the environment required to run the OpenStack CLI commands.
From your controller vm, rack-1-host-1:

```bash
cd /opt/stack/devstack
source openrc admin demo
```

* Open security group to allow our access to it
Note that in production or non-virtual lab you wouldn't want to open your SG like this

```bash
openstack security group create -c id -f value sg-testing
openstack security group rule create --protocol tcp sg-testing
openstack security group rule create --protocol icmp sg-testing
```

* Build a vm on rack-1-host-1 hypervisor on the public provider network

```bash
openstack port create --security-group sg-testing --network public vm1-port
openstack server create --flavor m1.nano --nic port-id=$(openstack port list | grep vm1-port | awk {'print $2'}) --availability-zone nova:rack-1-host-1 --image cirros-0.6.2-x86_64-disk vm1-provider
```

* Build a vm on rack-1-host-2 hypervisor on the public provider network

```bash
openstack port create --security-group sg-testing --network public vm2-port
openstack server create --flavor m1.nano --nic port-id=$(openstack port list | grep vm2-port | awk {'print $2'}) --availability-zone nova:rack-1-host-2 --image cirros-0.6.2-x86_64-disk vm2-provider
```

## Common installation problems

### Error: TLS is not enabled

The configure-cumulus playbook should properly disable TLS requirement for ovn-bgp-agent.  If you still get an error you may need to manually disable it and re-run stack.sh.  Look at the task in the playbook to see how to disable TLS for ovn-bgp-agent.

### RTNETLINK answers: Permission denied aka make sure IPv6 is enabled

The generic/ubuntu2204 Vagrant box has ipv6 disabled.  Devstack relies on IPv6 being enabled.  The configure-cumulus ansible playbook has a task that updates the vm to enable ipv6.  If IPv6 is disabled you will get a confusing permission denied error when the stack.sh script attempts to apply an IPv6 address to the br-ex bridge:

```text
+lib/neutron_plugins/services/l3:_neutron_configure_router_v6:416  sudo ip -6 addr replace 2001:db8::2/64 dev br-ex
RTNETLINK answers: Permission denied
```

You can check if ipv6 is enabled by running:

```bash
sysctl -a 2>/dev/null | grep disable_ipv6
```

If ipv6 is disabled you will see values of "1" for the output above, like this:

```bash
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

You can temporarily fix this manually via the following.  Note that this method will NOT survive a restart of the vagrant vm:

```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
```

### Remove a compute node

Once a compute node registers with the controller, you can no longer just unstack.sh or even clean.sh to rebuilt the compute node.  If you do need to re-stack, perhaps to enable a plugin, you have to first remove the existing references from the controller node.  This [guide from Mirantis](https://docs.mirantis.com/mcp/q4-18/mcp-operations-guide/openstack-operations/manage-compute-nodes/remove-compute-node.html#:~:text=To%20remove%20a%20compute%20node%3A&text=Log%20in%20to%20an%20OpenStack%20controller%20node.&text=The%20command%20output%20should%20display,perform%20live%20or%20cold%20migration) is helpful

```bash
sudo openstack --os-cloud devstack-admin --os-region RegionOne compute service list
sudo openstack --os-cloud devstack-admin --os-region RegionOne compute service delete <id>
```

Note that i had to add the --os-cloud devstack-admin to the token request to get some sort of admin token or else my GET requests to the provider endpoint always got a 403

```bash
sudo openstack --os-cloud devstack-admin token issue -c id -f value

curl -i -X GET <placement-endpoint-address>/resource_providers?name=<target-compute-host-name> -H \
'content-type: application/json' -H 'X-Auth-Token: <token>'

curl -i -X DELETE <placement-endpoint-address>/resource_providers/<target-compute-node-uuid> -H \
'content-type: application/json' -H 'X-Auth-Token: <token>'
```

Fwiw, my controller node had a response from the resource_provider curl GET call, but my compute node did not

## Common operational problems

### Viewing Logs

#### Use journalctl to view ovn-bgp-agent logs

```bash
sudo journalctl -u devstack@ovn-bgp-agent
```

#### openstack plugin logs, e.g. neutron, nova-compute, etc

These logs can be found in /var/log/syslog.  There does not appear to be a way to use a journalctl unit to view them individually

#### ovn-controller, ovsdb, NB DB, SB DB, stack.sh logs

These are all written to files in /opt/stack/logs/

### Geneve tunnel does not come up

Verify you have the iptables SNAT rule in place that rewrites the source address of every packet destined for the remote node loopback IP with the local node's loopback IP:

`sudo iptables --numeric -L -t nat`

Use tcpdump to make sure the iptables SNAT rule is working as expected.

`sudo tcpdump -nn -i eth1 geneve`

You should see geneve packets being set using the loopback addresses.  If the SNAT is not working you will see geneve packets sourced from one of the BGP p2p interface addresses.

### Ansible gets a timeout when trying to connect to a vagrant virtual machine

This usually happens if you destroy a vm and then recreate it. It may happen if you shut down and then later bring up a vm.  In these scenarios Vagrant might give the vm a new IP address, but it's not kind enough to update it's ansible inventory file.  Log into the new vm and gather it's IP address.  Then edit the inventory file, .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory, and update the IP for the vm.

### ovn-bgp-agent logs Unexpected exception while running the sync: (100, 'Network is down')

This often means that for some reason the ovs bridges are down.  Check their status with `ip link show br-ex` and `ip link show br-int`.  You may be able to bring them back up like

```bash
sudo ip link set br-ex up
sudo ip link set br-int up
```

### ovn-bgp-agent logs Exception: Could not retrieve schema from tcp

This means there is a connectivity issue trying to reach the controller node hosting the OVN SB DB.  Check your routing/firewall

### VMs on same network, but different hypervisors cannot communicate

By default OpenStack assumes everything is L2 adjacent.  Therefore it assumes that when two VMs are on the same subnet, but different hypervisors it just needs to send the packets out the provider bridge, br-ex which sends on to the upstream switching layer via physical server ports.  But when using the ovn-bgp-agent setup, br-ex does not get assigned any physical interfaces.  Instead it needs to hand packets off to kernel routing for them to be delivered.  Part of the glue that makes this work is a flow rule that ovn-bgp-agent daemon creates for br-ex that causes it to re-write the destnation MAC address from any packet it receives from the integration-bridge (br-int) with the MAC address of the provider bridge.  This causes the provider bridge to hand off the packet to the kernel which will then forward it using one of the ECMP BGP routes.

If this flow is missing on either hypervisor, the packet does not get delivered.

If you find that the flows are missing, you can test connectivity by manually adding the flows to each hypervisor.

* Discover the MAC address of br-ex

```bash
vagrant@rack-1-host-1:/opt/stack/devstack$ ip link show br-ex
8: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 5e:b7:7b:db:61:43 brd ff:ff:ff:ff:ff:ff
```

* Discover the name of the patch that connects br-ex to br-int

```bash
vagrant@rack-1-host-1:/opt/stack/devstack$ sudo ovs-vsctl show
<snip>
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port patch-provnet-23731af5-e966-4e75-8b41-0dc2e941c633-to-br-int
            Interface patch-provnet-23731af5-e966-4e75-8b41-0dc2e941c633-to-br-int
                type: patch
                options: {peer=patch-br-int-to-provnet-23731af5-e966-4e75-8b41-0dc2e941c633}
```

You appear to only need the first number after the patch name, e.g. patch-provnet-2

* Add the missing flow to br-ex

```bash
vagrant@rack-1-host-1:/opt/stack/devstack$ sudo ovs-ofctl --protocols=OpenFlow13 add-flow br-ex ip,priority=900,in_port="patch-provnet-2",actions=mod_dl_dst:5e:b7:7b:db:61:43,NORMAL
```

* Repeat process for the other hypervisor

Now your vms should be able to communicate with each other.  Note that in my experience, despite the man pages indicating that manually added flows have an infinite idle timeout I have found that the flows will timeout and disappear if not traffic is traversing them for some relatively small amount of time; a few minutes.  

As ovn-bgp-agent is meant to be adding these flows you should examine the ovn-bgp-agent logs for any errors.

## Using SSH port forwarding to Access Horizon

Let's assume you have started with a virtual machine as your base host and used vagrant to launch more virtual machine(s) to run Devstack. Your local client will not have direct access to the vagrant vm that is hosting the Horizon web management portal.  In this case you will need a chain of ssh tunnels in order to load Horizon from your local machine.

local client -> base server -> vagrant vm hosting Horizon

One way is to start two separate ssh tunnels, the first from your local client toward your base server, the 2nd from the base server to the vm hosting Horizon.

By default Horizon listens on HTTP port 80.  We will listen locally on port 8080 and forward that on.

From client to base server:

```bash
ssh -L8443:localhost:8443 user@base-server
```

From base server to vagrant vm hosting Horizon:

```bash
ssh -L 8443:localhost:80 vagrant@<vagrant vm IP> -i ~/ovn-bgp-vagrants/spine-leaf-test/.vagrant/machines/rack-1-host-1/libvirt/private_key
```

Note: I'm not sure there is a good way to do this in one command with the -J flag or defining it in your ssh conf file because you then would need the vagrant machine private key local to your client server and this can change if you rebuild your vagrant vm.

Now in your browser on your local client you should be able browse to the [Horizon portal](https://localhost:8443/dashboard)

## How to get TLS working on a multi-node devstack installation

It is possible to get TLS to work on a multi-node devstack installation.  Essentially to get it working you distribute the CA root and intermediate certificates from the controller node to all other nodes in the cluster. However, there are some tricks in the timing.

First, if you are using the configure-cumulus playbook, before you run it the first time to configure your compute node you'll need to comment out or otherwise skip the task "Disable tls-proxy requirement for ovn-bgp-agent"

All the following steps need to occur before you run stack.sh on the 2nd+ compute nodes.

Next, get a copy of /opt/stack/data/ca-bundle.pem, /opt/stack/data/devstack-cert.pem and /etc/ssl/certs/ca-certificates.crt from your controller node and copy it to the same location on your compute node.  Either via cut and paste or other method.

By default when you run stack.sh it creates it's own local CA and updates the files you just updated with the CA certificate files from the controler. So we need to stop that by editing stack.sh as follows:

```text
vagrant@rack-1-host-2:~/devstack$ git diff stack.sh
diff --git a/stack.sh b/stack.sh
index dce15ac0..416fde38 100755
--- a/stack.sh
+++ b/stack.sh
@@ -605,7 +605,7 @@ uname -a
 
 # Reset the bundle of CA certificates
 SSL_BUNDLE_FILE="$DATA_DIR/ca-bundle.pem"
-rm -f $SSL_BUNDLE_FILE
+##rm -f $SSL_BUNDLE_FILE
 
 # Import common services (database, message queue) configuration
 source $TOP_DIR/lib/database
@@ -895,9 +895,10 @@ fi
 # we don't run into problems with missing certs when apache
 # is restarted.
 if is_service_enabled tls-proxy; then
-    configure_CA
-    init_CA
-    init_cert
+    #configure_CA
+    #init_CA
+    #init_cert
+    echo_summary "let's try skipping this..."
 fi
 ```

Now if you run stack.sh it should complete and properly configure ovn-bgp-agent to use TLS

## History - trying to run on Ubuntu 20.04

When using the ansible provider, make sure you are in your intended ansible environment (e.g. python venv) before you run vagrant up
The blog guide runs into trouble installing Devstack on centos8.  The Openstack Devstack site only mentions support for Ubuntu and two other Linux distriubtions, but not centos.  So I decided to try and use ubuntu.  In the end I ran into the same error using Ubuntu.  It appears something in the pre-install script that Luis wrote is no longer working.  I am now attempting to just follow the Devstack wiki to complete the install of Devstack and then see if I can incorporate the blog settings.

I am attempting to get the hosts working with ubuntu.  The first issue I ran into is trying to use box vagrant/jammy64.  This was because that box doesn't support the provider libvirt.  Next i found generic/ubuntu2204.  Unfortunately this box does not work either- at least when running Vagrant 2.6. The box would download and attept to start it's config process, but reach a point where vagrant is trying to ssh into the box and fail.  this appears due to the version of vagrant that was installed on my host system which is running Ubuntu 20.04.  The packaged vagrant version included with 20.04 is 2.2.6 which doesn't support generic/ubuntu2204 because Vagrant 2.2.6 relies on RSA keys and the version of OpenSSH included with 22.04 disables them by default.  This is described [here|https://github.com/hashicorp/vagrant/pull/13179] .  Workarounds include manually installing a supported version of Vagrant, or changing my box to use Ubuntu 20.04 instead of 22.04.  for now I am choosing to use 20.04

* This is required for installs on Ubuntu

```bash
sudo apt install python3.8-distutils
```

* I ran the install_devstack master script from the blog which failed, but did produce an edited version of local.conf .  Following the Devstack wiki I cloned the Devstack repo.  Instead of hand editing a new local.conf, i just copied over the one that ahd been previously generated and ran the stack.sh script.  This errored out because it claimed it had not been tested on ubuntu Focal (20.04) and that if I wanted to continue I had to set the Force env var

```bash
export FORCE=yes
```

* Ubuntu 20.04 has some sort of TLS issue with either git or the curl library version git relies on.  If you run the stack.sh script vanilla you will run into errors like:

```text
+functions-common:git_timed:688            timeout -s SIGINT 0 git clone --no-checkout https://opendev.org/openstack/neutron.git /opt/stack/neutron
Cloning into '/opt/stack/neutron'...
error: RPC failed; curl 56 GnuTLS recv error (-54): Error in the pull function.
fatal: the remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```

Strangely, you can often avoid these errors by turning on a debug flag for git.  so run the command like so:

```bash
GIT_CURL_VERBOSE=1 ./stack.sh
```

or for repated failures, turn off ssl verification (yuck):

```bash
git config --global http.sslVerify false
git config --global http.postBuffer 1048576000
```
