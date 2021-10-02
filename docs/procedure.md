# Openshift installation 

## Blueprint deployment

Deploy the blueprint.   

Start the Deployment: it is going to take a while to boot up everything.

## ocp-web Provisioning 

As a first thing, we need to provision ocp-web in order to prepare the "installations" directory served by the NGINX web server. This is going to provide everything to bootstrab RHCOS machines and configure "bootstrap", "masters" and "workers"

To facilitate this job, an ansible playbook is ready to be used on the infra-server box.

### How to get your pullSecret from redhat

You need to register yourself on the Red Hat Cloud portal and retrieve your pullSecret from

- [user-provisioned-infrastructure](https://cloud.redhat.com/openshift/install/metal/user-provisioned)
- Selecy "Copy pull secret" or "Donwload pull secret"

### How to get UDF public key

You can find the UDF public key connecting via ssh to infra-server and 

    cat ~/.ssh/authorized_keys
    
end copy the first key of that file

### OCP Release

You are going to decide which Openshift Container Platform software release you are using.  
Check this [link](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/) and choose your release number

### RHCOS Release

You are going to decide which RedHat Core OS release you are using.  
Check this [link](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/) and choose your release number

### Inventory Hosts personalization 

Connect with SSH to infra-server and modify the inventory hosts file with 

- OCP Release version
- RHCOS Main Release version
- RHCOS Full release version
- Your RedHat Pull Secret
- The SSH public key from UDF

Edit the invetory file:

    cd ~/f5-udf-ocp-baremetal-blueprint/ansible
    vim inventory/hosts

and past there your collected information

```
ocp_release=4.7.7
rhcos_branch=4.7
rhcos_release=4.7.7
redhat_pull_secret='Enter the text from your Pull Secrets'
udf_access_public_key='UDF Access Public Key -> get it from a centos user on a centos box in UDF'
```

### Ansible Playbook to prepare ocp-web installations directory

Connect with SSH to infra-server and

    cd ~/f5-udf-ocp-baremetal-blueprint/ansible
    ansible-playbook playbooks/preparing-ocp-web-installations-directory.yaml
    
## ocp-boostrap setup

Connect via SSH to RHEL box named "ocp-bootstrap" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.9::10.1.1.1:255.255.255.0:ocp-bootstrap.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/bootstrap.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot

Now follow the reboot process on the console in the browser. It will take some time and a couple of reboot; when RHCOS will be ready, connect to it using the "SSH (core user)" method and verify that everything is good with

    journalctl -b -f -u release-image.service -u bootkube.service
    
you should read somthing like:

    Created "99_openshift-cluster-api_master-user-data-secret.yaml" secrets.v1./master-user-data -n openshift-machine-api
    Created "99_openshift-cluster-api_worker-user-data-secret.yaml" secrets.v1./worker-user-data -n openshift-machine-api
    Created "99_openshift-machineconfig_99-master-ssh.yaml" machineconfigs.v1.machineconfiguration.openshift.io/99-master-ssh -n
    Created "99_openshift-machineconfig_99-worker-ssh.yaml" machineconfigs.v1.machineconfiguration.openshift.io/99-worker-ssh -n

## ocp-master{1,2,3} setup

First of all, connect to the ocp-web box via SSH and change NGINX configuration:

    sudo su -
    cd /etc/nginx/
    rm nginx.conf
    ln -s nginx.conf-master nginx.conf
    systemctl restart nginx
    systemctl status nginx
    
### ocp-master1 setup

Connect via SSH to RHEL box named "ocp-master1" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.10::10.1.1.1:255.255.255.0:master-1.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/master.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot

The box reboots and is going to take some time and a couple of reboots more to get to a stable state.  
You can proceed to ocp-master2 

### ocp-master2 setup

Connect via SSH to RHEL box named "ocp-master2" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.11::10.1.1.1:255.255.255.0:master-2.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/master.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot

The box reboots and is going to take some time and a couple of reboots more to get to a stable state.  
You can proceed to ocp-master2 

### ocp-master3 setup

Connect via SSH to RHEL box named "ocp-master3" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.12::10.1.1.1:255.255.255.0:master-3.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/master.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot

The box reboots and is going to take some time and a couple of reboots more to get to a stable state.  

### Check ocp-masters status

Go back via SSH to ocp-web and:

    watch oc get nodes -A
    
until you get this output:

```
Every 2.0s: oc get nodes -A                                                                                                                                               ocp-web: Fri Apr 23 16:50:29 2021

NAME                      STATUS   ROLES    AGE     VERSION
master-1.ocp.f5-udf.com   Ready    master   9m25s   v1.20.0+c8905da
master-2.ocp.f5-udf.com   Ready    master   8m16s   v1.20.0+c8905da
master-3.ocp.f5-udf.com   Ready    master   102s    v1.20.0+c8905da
```

## ocp-worker{1,2,3} setup

First of all, connect to the ocp-web box via SSH and change NGINX configuration:

    sudo su -
    cd /etc/nginx/
    rm nginx.conf
    ln -s nginx.conf-worker nginx.conf
    systemctl restart nginx
    systemctl status nginx
    
### ocp-woker1 setup

Connect via SSH to RHEL box named "ocp-worker1" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4 
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.13::10.1.1.1:255.255.255.0:worker-1.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/worker.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot

The box reboots and is going to take some time and a couple of reboots more to get to a stable state.  
You can proceed to ocp-worker2

### ocp-woker2 setup

Connect via SSH to RHEL box named "ocp-worker2" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4 
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.14::10.1.1.1:255.255.255.0:worker-2.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/worker.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot

The box reboots and is going to take some time and a couple of reboots more to get to a stable state.  
You can proceed to ocp-worker3

### ocp-woker3 setup

Connect via SSH to RHEL box named "ocp-worker3" (normal SSH, not the one for "core" user) and, at the same time, open its "console" in the browser.

Issue the following commands (**CHANGE the RHCOS_RELEASE variable according to your configuration**)

    RHCOS_RELEASE=4.8.2
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-$RHCOS_RELEASE-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.15::10.1.1.1:255.255.255.0:worker-3.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-$RHCOS_RELEASE-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/worker.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
The box reboots and is going to take some time and a couple of reboots more to get to a stable state.  

### Approve CSR and Check ocp-masters status

Open two SSH connections to ocp-web; in the first one issue the following command:

    watch oc get nodes -A
    
while in the second one: 

    while true ; do oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve ; sleep 5 ; clear ; done
    
until you get this output:

```
Every 2.0s: oc get nodes -A                                        ocp-web: Fri Apr 23 17:04:13 2021

NAME                      STATUS   ROLES    AGE     VERSION
master-1.ocp.f5-udf.com   Ready    master   23m     v1.20.0+c8905da
master-2.ocp.f5-udf.com   Ready    master   22m     v1.20.0+c8905da
master-3.ocp.f5-udf.com   Ready    master   15m     v1.20.0+c8905da
worker-1.ocp.f5-udf.com   Ready    worker   2m34s   v1.20.0+c8905da
worker-2.ocp.f5-udf.com   Ready    worker   2m28s   v1.20.0+c8905da
worker-3.ocp.f5-udf.com   Ready    worker   60s     v1.20.0+c8905da
```
At this point you can stop the two commands and check the detailed status of the cluster with:

    oc get pods -A | grep -vi running | grep -vi completed
    
The cluster will be ready when you will not have any pod in a status different from "completed" or "ContainerCreating".  
You can watch this command until you get to "0":

    while true ; do oc get pods -A | grep -vi running | grep -vi completed | wc -l ; sleep 5 ; done
    
When all pods are in a "running" or "completed" status, you can get a general view of the cluster operator status with

    oc get co