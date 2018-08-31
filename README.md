# Ansible role vm_libvirt_cloudinit 
This role allow to deploy a vm on a target host while providing an
interface to customize it using cloud-init.

### Examples
`````
# Deploy `myvm` as vm on `virtualizer` using ubuntu cloud image as a base.
-   hosts: myvm
    gather_facts: no
    roles:
    -   role: vm_libvirt_cloudinit
        vm_hoster: virtualizer
        vm_drives:
        -   type: file
            source: https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img


# Deploy `myvm` as vm on `virtualizer` using ubuntu cloud image as a base.
# Manage a logical volume to use as backing device.
# sets root password to `secret`, allow a ssh key login as ubuntu
-   hosts: myvm
    roles:
    -   role: vm_libvirt_cloudinit
        vm_hoster: virtualizer
        vm_drives:
        -   type: block
            source: /opt/xenial-server-cloudimg-amd64-disk1.img
            path: /dev/cinder-volumes/myvm_root
            manage: yes
            size_gb: 10
            
# Deploy `myvm` as vm on `virtualizer` using centos cloud image.
# Attach a first network interface as a macvlan sub-interface to `eth0`
# and a second one to a br-provider openvswitch switch, trunking tags 30-40
-   hosts: myvm
    roles:
    -   role: vm_libvirt_cloudinit
        vm_hoster: virtualizer
        vm_drives:
        -   type: block
            source: https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
            path: /dev/cinder-volumes/myvm_root
            manage: yes
            size_gb: 10
`````



### Network Configuration
The roles variable `vm_drives` is a list of dictionaries used to define
the network interfaces which will be attached to the vm.
Each element must contain the key `type`, set to `bridge`, `direct` or
`openvswitch`.

The network interface will be attached to:
A linuxbridge identified by the `parent` key if `network.type=="bridge"`.
A macvlan sub-interface of the network device identified by the `parent`
key if `network.type=="direct"`.
An openvswitch switch identified by the `parent` key if 
`network.type=="openvswitch"`

If attaching to an openvswitch switch these keys can also be used:
- `trunk`: list of tags to trunk to the openvswitch port
- `access`: tag to apply to the openvswitch port

For any configuration the element can contain these keys:
- `mac`: statically assign a mac address to the interface
- `address`: network address, expressed as address/netmask
  (es. 192.168.0.1/24)
- `gateway`: gateway ip for the interface

Using the values in this list the role will generate the
`<interface>` sections of the libvirt domain and the cloud-init
`network-config` data.

### Storage Configuration
The roles variable `vm_drives` is a list of dictionaries used to define
the drives which will be attached to the vm.
Each element must contain the key `type`, set to `file` or `block`.

If `drive.type=="file"` these other keys are used:
- `source`: path/url of the image which will be cloned into the vm drive
            in `vm_dir`.  
            The image can be resized by setting `size_gb`
- `path`: path of an image which will be used as-is as the vm drive.

If `drive.type=="block"` these other keys are used:
- `path`: path of the block device to be used as vm drive.
- `source`: path of an image (file or block device) which will be cloned
            into `path`.
- `manage`: if true the module will try to create (and delete if
            `vm_delete==True`) a LVM2 logical volume as `path`.  
            (see [examples](#examples))

If the drive is created by this role an attribute `size_gb` can be used
to resize the `source` image into the drive. If not set `source` size
will be preserved.

### Role variables
The role uses the following variables, here listed with their default
values:

`````
vm_hoster:
`````
This variable is mandatory. It represent the ansible_hostname of the
system on which this role will spawn the vm.

`````
vm_hostname: "{{inventory_hostname}}"
`````
The hostname of the vm.

`````
vm_autostart: true
vm_start: true
vm_delete: false
vm_rebuild: false
`````
These variables controls the lifecycle of the vm.

`````
vm_cpus: 2
vm_memory_mb: 512
`````
Resources allocated to the vm

`````
vm_dir: "/opt/vms/{{vm_hostname}}"
vm_cloudinit_iso: "{{vm_dir}}/nocloud.iso"
`````
In `vm_dir` the role will place:
- `user-data`, `meta-data`, `network-data` and `nocloud.iso`: files used
  to configure the vm via `cloud-init`
- domain.xml: a dump of the vm libvirt domain definition
- drive_`N`.img: the vm drives created by this module

`````
vm_drives:
-   type: file
    format: qcow2
    source: https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img
    size_gb: 5
`````
List of drives which will be attached to the vm. See
[Strorage Configuration](#storage-configuration)

`````
vm_networks: []
`````
List of network interfaces which will be attached to the vm. See
[Network Configuration](#network-configuration)

`````
vm_nameservers:
-   8.8.8.8
`````
DNS servers configured on the vm.

`````
vm_userdata: {}
`````
Cloud-init's user-data as inline YAML (see [examples](#examples))

`````
vm_vnc_address: 127.0.0.1
vm_vnc_port: 0
`````
Address and port on which the libvirt domain vnc server will bind.
If `vm_vnc_port==0` no vnc server will be attached to the vm.
If `vm_vnc_port==-1` a port will be autoallocated by libvirt.

`````
vm_wait_for_tcp22: "{{vm_start}}"
vm_wait_for_tcp22_delegate: true
`````
If `vm_wait_for_tcp22==True` the role will wait until `vm_hoster` will
be able to reach tcp22 on the first address defined on `vm_networks`.
If `vm_wait_for_tcp22_delegate==False` this test will be performed from
the deployment host.

-----

`````
vm_allow_passthrough: false
`````
If true the libvirt domain cpu mode will be set to
`host-passthrough`, allowing the definition on nested vms.
If false the cpu model will follow the defaults for the `vm_hoster`
libvirt daemon.

`````
vm_do_bootstrap: true
`````
If true the role will manage the `vm_cloudinit_iso`, `vm_drives`
creation, otherwise only the domain will be defined.
`````
vm_cloudinit_iso: "{{vm_dir}}/nocloud.iso"
`````
Path where the NoCloud datasource for cloud-init will be created

`````
vm_emulator_path: "/usr/libexec/qemu-kvm"
`````
Path to the qemu-kvm executable on `vm_hoster`

`````
vm_identify_interfaces_by_parent: false
`````
If `true` the role will match libvirt<->cloudinit interfaces mapping by
`parent`, the default is to match by hwaddr.
This option is here only as a retrocompatibility workaround, and should
not be used.

`````
vm_network_config_version: "1"
`````
Version of the cloud-init network-config format to use.  
Support for version "2" is not working, so this option should not be used.
