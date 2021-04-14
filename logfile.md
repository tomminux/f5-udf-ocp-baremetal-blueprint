# IMPORTANT --> UPDATE THE BLUEPRINT

- nginx conf files --> OK
- timezone on ocp-web -> OK
- resolv.conf on ocp-web -> OK
- clean installation directory --> OK
- configure BIG-IP and apply AS3 configMap for udf-services -> OK

- update DNSMASQ config as per Ulises instructions for *.apps.ocp.f5-udf.com wildcard record
- core user on bootstrap, masters and cores 
- remote installations directory on OCP-WEB
- move installations directory creation and downloading file to "preparing..." ansible playbook



# UDF Blueprint: codename-thor - LogFile

## UDF Deployment Machines Configuration

| VM UDF Name    | Template                | Capacity                   | Mgmt Address   | External Address | Internal Address|
|----------------|-------------------------|--------------------------- |----------------|------------------|-----------------|
| infra-server   | Ubuntu 20.04 LTS Server | 4 vCPU, 16GB RAM,100GB vHD | 10.1.1.4       |                  | 10.1.20.4       |
| bigip-cis      | BIGIP 16.0.0-0.0.12     | 4 vCPU, 16GB RAM,105GB vHD | 10.1.1.5       | 10.1.10.5        | 10.1.20.5       |
| ocp-web        | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.6       |                  | 10.1.20.6       |
| ocp-bootstrap  | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.7       |                  | 10.1.20.7       |
| master-1       | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.8       |                  | 10.1.20.8       |
| master-2       | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.9       |                  | 10.1.20.9       |
| master-3       | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.10      |                  | 10.1.20.10      |
| worker-1       | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.11      |                  | 10.1.20.11      |
| worker-2       | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.12      |                  | 10.1.20.12      |
| worker-3       | Centos 8                | 4 vCPU, 16GB RAM, 50GB vHD | 10.1.1.13      |                  | 10.1.20.13      |
| linux-jumphost | Ubuntu 20.04 LTS Server | 2 vCPU, 8GB RAM, 100GB vHD | 10.1.1.14      | 10.1.10.14       | 10.1.20.14      |

## ..:: infra-server Configuration ::..

## ..:: ocp-web Configuration ::..
```
sudo su -  
mkdir .kube  
cp /usr/share/nginx/html/installations/auth/kubeconfig ~/.kube/config  
cp /usr/share/nginx/html/installations/oc /usr/local/bin/.  
mkdir /home/ubuntu/.kube  
cp ~/.kube/config /home/ubuntu/.kube/config  
chown -R ubuntu:ubuntu /home/ubuntu/.kube  
```


## ..:: ocp-web Configuration ::..

    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.7.0-x86_64-live-initramfs.x86_64.img  
    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.7.0-x86_64-live-kernel-x86_64  
    wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img  
    
## ..:: ocp-bootstrap Configuration ::..

    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-4.7.0-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-4.7.0-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.9::10.1.1.1:255.255.255.0:ocp-bootstrap.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/bootstrap.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
## ..:: masters Configuration ::..

    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-4.7.0-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-4.7.0-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
    IFACE=enp0s4
    
### master-1
    
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.10::10.1.1.1:255.255.255.0:master-1.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/master.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
### master-2
    
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.11::10.1.1.1:255.255.255.0:master-2.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/master.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
### master-3
    
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.12::10.1.1.1:255.255.255.0:master-3.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/master.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
## ..:: workers Configuration ::..

    sudo timedatectl set-timezone Europe/Rome
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-kernel-x86_64
    curl -O -L -J http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-initramfs.x86_64.img
    sudo mv rhcos-4.7.0-x86_64-live-kernel-x86_64 /boot/vmlinuz-rhcos
    sudo mv rhcos-4.7.0-x86_64-live-initramfs.x86_64.img /boot/initramfs-rhcos.img
        
### worker-1
   
    IFACE=enp0s4 
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.13::10.1.1.1:255.255.255.0:worker-1.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/worker.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
### worker-2

    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.14::10.1.1.1:255.255.255.0:worker-2.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/worker.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
### worker-3

    IFACE=enp0s4
    sudo grubby --add-kernel=/boot/vmlinuz-rhcos --args="ip=10.1.1.15::10.1.1.1:255.255.255.0:worker-3.ocp.f5-udf.com:$IFACE:none nameserver=10.1.1.4 rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.live.rootfs_url=http://10.1.1.8:8080/installations/rhcos-4.7.0-x86_64-live-rootfs.x86_64.img coreos.inst.ignition_url=http://10.1.1.8:8080/installations/worker.ign" --initrd=/boot/initramfs-rhcos.img --make-default --title=rhcos
    sudo reboot
    
