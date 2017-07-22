# ICP_Local_Install
I followed the instructions and resources provided in "https://hub.docker.com/r/ibmcom/cfc-installer/?cm_mc_uid=34158262147114835231232&cm_mc_sid_50200000=" to build my very first ICP in local environment, and I decided to summarize the notes and walkthrough into the following sections so others can have a quick jumpstart.

Notation:<br/>
{boot-node-ip} --> to be replaced by boot-node's ip<br/>
$ --> command prompt, indicates start of a command

## Precondition
Have Vagrant(https://www.vagrantup.com/downloads.html) and VirtualBox(https://www.virtualbox.org/wiki/Downloads) installed.<br/>
Have your notebook connected to wifi.<br/>
The following commands can be applied to Mac or Linux straightaway and may need some modifications for Windows.<br/>
You have 16GB of RAM.

## Prepare Vagrant image and VirtualBox VM:
Under the "Vagrant" folder, there are 5 sub-folders namely:<br/>
microservice-builder-base: this is the base image for rest of the VirtualBox VM<br/>
boot-node: refer to "https://www.ibm.com/support/knowledgecenter/SSBS6K_1.2.0/getting_started/architecture.html" for Boot Node<br/>
master-node: refer to "https://www.ibm.com/support/knowledgecenter/SSBS6K_1.2.0/getting_started/architecture.html" for Master Node<br/>
proxy-node: refer to "https://www.ibm.com/support/knowledgecenter/SSBS6K_1.2.0/getting_started/architecture.html" for Proxy Node<br/>
worker-node: refer to "https://www.ibm.com/support/knowledgecenter/SSBS6K_1.2.0/getting_started/architecture.html" for Worker Node<br/>

Assuming "{ICP_Local_Install}" is our working directory already, first of all, we need to build the base image with the following commands:<br/>
$ cd {ICP_Local_Install}/vagrant/microservice-builder-base && vagrant up<br/>
This may take several to 1x minutes depends on the network speed.<br/>
$ vagrant package --base microservice-builder-base --output microservice-builder-base.box<br/>
$ vagrant box add microservice-builder-base microservice-builder-base.box<br/>
This will store the newly created VM as a vagrant image, and now it's safe to remove the transitional VM with the following command.<br/>
$ vagrant destroy -f<br/>

Next we can start building VMs for boot-node, master-node, proxy-node, and worker-node with following commands one at a time.<br/>
$ cd {ICP_Local_Install}/vagrant/boot-node && vagrant up<br/>
$ cd {ICP_Local_Install}/vagrant/master-node && vagrant up<br/>
$ cd {ICP_Local_Install}/vagrant/proxy-node && vagrant up<br/>
$ cd {ICP_Local_Install}/vagrant/worker-node && vagrant up<br/>

Now infrastructure is ready, and the only thing left to do before install ICP is to ensure the connectivity among these nodes.<br/>

### Ensure Connectivity Among Nodes
Reteive IP addresses of all four nodes by running the following commands one at a time and take note of the IP for eth1.<br/>
$ cd {ICP_Local_Install}/vagrant/boot-node && vagrant ssh -c "ifconfig"<br/>
$ cd {ICP_Local_Install}/vagrant/master-node && vagrant ssh -c "ifconfig"<br/>
$ cd {ICP_Local_Install}/vagrant/proxy-node && vagrant ssh -c "ifconfig"<br/>
$ cd {ICP_Local_Install}/vagrant/worker-node && vagrant ssh -c "ifconfig"<br/>

### Configure the IP addresses into boot-node and setup boot-node into a DNS server.<br/>
$ cd {ICP_Local_Install}/vagrant/boot-node && \\<br/>
vagrant ssh -c "sudo su -c 'echo {boot-node-ip} boot-node >> /etc/hosts'" && \\<br/>
vagrant ssh -c "sudo su -c 'echo {master-node-ip} master-node >> /etc/hosts'" && \\<br/>
vagrant ssh -c "sudo su -c 'echo {master-node-ip} master.cfc >> /etc/hosts'" && \\<br/>
vagrant ssh -c "sudo su -c 'echo {proxy-node-ip} proxy-node >> /etc/hosts'" && \\<br/>
vagrant ssh -c "sudo su -c 'echo {worker-node-ip} worker-node >> /etc/hosts'"

Restart the DNS server running on boot-node<br/>
$ vagrant ssh -c "sudo su -c 'systemctl restart dnsmasq'"

### Configure master-node, proxy-node, worker-node to resolve domain name with boot-node's DNS server<br/>
$ cd {ICP_Local_Install}/vagrant/master-node && \\<br/>
vagrant ssh -c "sudo su -c 'echo nameserver {boot-node-ip} > /etc/resolv.tmp'" && \\<br/>
vagrant ssh -c "sudo su -c 'cat /etc/resolv.conf >> /etc/resolv.tmp'" && \\<br/>
vagrant ssh -c "sudo su -c 'mv /etc/resolv.tmp /etc/resolv.conf'"<br/>
$ cd {ICP_Local_Install}/vagrant/proxy-node && \\<br/>
vagrant ssh -c "sudo su -c 'echo nameserver {boot-node-ip} > /etc/resolv.tmp'" && \\<br/>
vagrant ssh -c "sudo su -c 'cat /etc/resolv.conf >> /etc/resolv.tmp'" && \\<br/>
vagrant ssh -c "sudo su -c 'mv /etc/resolv.tmp /etc/resolv.conf'"<br/>
$ cd {ICP_Local_Install}/vagrant/worker-node && \\<br/>
vagrant ssh -c "sudo su -c 'echo nameserver {boot-node-ip} > /etc/resolv.tmp'" && \\<br/>
vagrant ssh -c "sudo su -c 'cat /etc/resolv.conf >> /etc/resolv.tmp'" && \\<br/>
vagrant ssh -c "sudo su -c 'mv /etc/resolv.tmp /etc/resolv.conf'"

### Retrieve SSH public key from boot-node and configure the key into master, proxy, worker node, so we won't be asked for password during ICP installation process<br/>
$ cd {ICP_Local_Install}/vagrant/boot-node<br/>
$ vagrant ssh -c "sudo su -c 'echo -e  \\"y\\n\\"|ssh-keygen -b 4096 -t rsa -f ~/.ssh/id_rsa -P \\"\\" -C \\"\\"'"<br/>
$ vagrant ssh -c "sudo su -c 'cat ~/.ssh/id_rsa.pub'"<br/>
Copy everything between "ssh-rsa" and "==" inclusive (aka. {SSH_PUBLIC_KEY}) into a notepad, we need this rightaway.<br/>

Configure boot-node's key into master node<br/>
$ cd {ICP_Local_Install}/vagrant/master-node<br/>
$ vagrant ssh -c "sudo su -c 'mkdir -p ~/.ssh/'"<br/>
$ vagrant ssh -c "sudo su -c 'echo {SSH_PUBLIC_KEY} >> ~/.ssh/authorized_keys'"<br/>
$ vagrant ssh -c "sudo su -c 'chmod 400 ~/.ssh/authorized_keys'"

Configure boot-node's key into proxy node<br/>
$ cd {ICP_Local_Install}/vagrant/proxy-node<br/>
$ vagrant ssh -c "sudo su -c 'mkdir -p ~/.ssh/'"<br/>
$ vagrant ssh -c "sudo su -c 'echo {SSH_PUBLIC_KEY} >> ~/.ssh/authorized_keys'"<br/>
$ vagrant ssh -c "sudo su -c 'chmod 400 ~/.ssh/authorized_keys'"

Configure boot-node's key into worker node<br/>
$ cd {ICP_Local_Install}/vagrant/worker-node<br/>
$ vagrant ssh -c "sudo su -c 'mkdir -p ~/.ssh/'"<br/>
$ vagrant ssh -c "sudo su -c 'echo {SSH_PUBLIC_KEY} >> ~/.ssh/authorized_keys'"<br/>
$ vagrant ssh -c "sudo su -c 'chmod 400 ~/.ssh/authorized_keys'"

Test Connectivity From Boot-Node<br/>
$ cd {ICP_Local_Install}/vagrant/boot-node<br/>
$ vagrant ssh -c "sudo su -c 'ssh root@master-node'"<br/>
$ exit<br/>
$ vagrant ssh -c "sudo su -c 'ssh root@proxy-node'"<br/>
$ exit<br/>
$ vagrant ssh -c "sudo su -c 'ssh root@worker-node'"<br/>
$ exit

## Install ICP From Boot-Node
SSH into boot-node as root and start to install ICP. The latest version we are using is 1.2.0 refered to "https://www.ibm.com/support/knowledgecenter/SSBS6K_1.2.0/installing/install_containers_CE.html"<br/>
$ cd {ICP_Local_Install}/vagrant/boot-node<br/>
$ vagrant ssh -c "sudo su -"<br/>
$ docker pull ibmcom/cfc-installer:1.2.0<br/>
$ docker run -e LICENSE=accept --rm -v "$(pwd)":/data ibmcom/cfc-installer:1.2.0 cp -r cluster /data<br/>
$ cp ~/.ssh/id_rsa cluster/ssh_key<br/>
$ echo [master] > cluster/hosts && \\<br/>
echo {master-node-ip} >> cluster/hosts && \\<br/>
echo [worker] >> cluster/hosts && \\<br/>
echo {worker-node-ip} >> cluster/hosts && \\<br/>
echo [proxy] >> cluster/hosts && \\<br/>
echo {proxy-node-ip} >> cluster/hosts

By entering the following (last) command, ICP installation is commenced:<br/>
$ cd cluster && docker run -e LICENSE=accept --net=host --rm -t -v "$(pwd)":/installer/cluster ibmcom/cfc-installer:1.2.0 install<br/>
After a successful installation, we should see something like following prompts up in terminal.
![Alt text](successful_install.gif "successful_install")

Now let's open up the "UI URL" in browser to take a look at ICP's dashboard
![Alt text](login.gif "login")
![Alt text](dashboard.gif "dashboard")
