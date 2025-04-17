### KVM installation steps

- check for compatibility:

  `cat /proc/cpuinfo | egrep "vmx|svm"`

- install virtualization pakacges:

  `dnf groupinstall "Virtualization Host"
`

- install extra tools:

  `yum -y install virt-top libguestfs-tools virt-install virt-manager guestfs-tools`

- check for modules:

  `lsmod | grep kvm`

- A bridge interface is necessary for accessing VMs from outside of the hypervisor network. To create a bridge interface:

  `nmcli connection show
`
Delete the connection by name or UUID

  `nmcli connection delete [name_or_UUID]
`

- Add a new bridged connection called br0

  `nmcli connection add type bridge con-name br0 ifname br0
`

- Add a physical interface to the bridge:

 `nmcli connection add type bridge-slave ifname enp0s3 master br0
`

- Modify the connection details. To use DHCP for IP address assignment, run

  `nmcli connection modify br0 ipv4.method auto
`

- Alternatively, provide the data for a static IP address:

```
  sudo nmcli connection modify br0 ipv4.addresses [IP_address/subnet]
  sudo nmcli connection modify br0 ipv4.gateway [gateway]
  sudo nmcli connection modify br0 ipv4.dns [DNS]
  sudo nmcli connection modify br0 ipv4.method manual
```

- Activate the bridge and interface with:

  `sudo nmcli connection up br0
`

- test the setup:

 `sudo virt-install --name=ubuntu --ram=3072 --vcpus=2 --file=/var/lib/libvirt/images/ubuntu.img,size=20 --cdrom=Downloads/ubuntu-24.04.1-desktop-amd64.iso --network bridge=br0 --nographics
`
#Thanks to Marko Aleksic [](https://phoenixnap.com/kb/install-kvm-centos)

### Now moving to the KVM operational part

- to create a VM:

```
virt-install --name=rocky9-cli-vm --vcpus=1 --memory=2048 --location /data/iso/Rocky-9.5-x86_64-dvd.iso --disk path=/data/VMs/rocky9-cli-vm/centos7-ks.qcow2,format=qcow2,size=10,bus=virtio \
--os-variant linux --osinfo rocky9 --network network='default',model=virtio \
--extra-args 'console=ttyS0,115200n8 serial' --nographics
```

- to check VMs status
  `virsh list --all`
  
- to start a VM
  `virsh start rocky9-cli-vm`
  
- to shutdown a VM:

  `virsh shutdown rocky9-cli-vm`

- to remove a VM with including provisioned storage:

  `virsh undefine rocky9-cli-vm --remove-all-storage`

- to create a new pool(directory /data/VMs/DB_shared needs to be created manually):

  `virsh pool-define-as DB_shared dir - - - - "/data/VMs/DB_shared"
`

- to create 12GB disk in the pool DB_shared:

  ```
  virsh vol-create-as DB_shared disk1.qcow2 12G --format qcow2
  virsh attach-disk rocky9-cli-vm --source /data/VMs/DB_shared/disk1.qcow2 --target vdb --cache none --driver qemu --subdriver qcow2 --config
  ```

- to create a new network:

  `vim network-private.xml
`
```
<network>
        <name>private</name>
        <bridge name="virbr2"/>
        <ip address="192.168.11.1" netmask="255.255.255.0">
                <dhcp>
                        <range start="192.168.11.2" end="192.168.11.254"/>
                </dhcp>
        </ip>
        <ip family="ipv6" address="2002:c0a8:0b01::c0ab:0b01" prefix="64"/>
</network>
```

  `virsh net-define --file network-private.xml
`
- to list enabled networks:

  `virsh net-list
`

- to list all networks:

  `virsh net-list --all
`

- to enable the new network:

  `virsh net-start private
`

- to enable autostart:

  `virsh net-autostart private
`

- to check the network details as xml:
  
  `virsh net-dumpxml private
`

- to check the dhcp leases of network:

`virsh net-dhcp-leases private
`

- check the VM interface connectivity status:

  `virsh domiflist rocky9-cli-vm
`

- create a new interface for the VM rocky9-cli-vm, and attach the network interface to the new network private:
 
  `virsh attach-interface --domain rocky9-cli-vm --type network --source private --model virtio --config --live
`

- again, check the network details for the interface, such as below:

```
  virsh net-dhcp-leases private
  virsh domifaddr rocky9-cli-vm
```

- create a clone from VM:

  `virt-clone --original rocky9-cli-vm --name=rocky9-cli-vm-template --auto-clone`

- create a template from the clone, removing all user accounts except ansible user

  `virt-sysprep -d rocky9-cli-vm --enable user-account --keep-user-accounts ansible`
 
- create a clone from template

  `virt-clone --original=rocky9-cli-vm-template --name=rocky_clone --auto-clone`

- log/ssh to the VM and check the interface details from inside



