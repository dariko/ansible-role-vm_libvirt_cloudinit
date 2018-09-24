[![Build Status](https://travis-ci.org/dariko/ansible-role-vm_libvirt_cloudinit.svg?branch=master)](https://travis-ci.org/dariko/ansible-role-vm_libvirt_cloudinit)

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

# Deploy `myvm` as vm on `virtualizer` using a local copy of ubuntu 
# 16.04 image as a base.
# Manage a logical volume to use as backing device.
# Run a script on boot
-   hosts: myvm
    roles:
    -   role: vm_libvirt_cloudinit
        vm_hoster: virtualizer
        vm_drives:
        -   type: block
            source: /opt/xenial-server-cloudimg-amd64-disk1.img
            path: /dev/cinder-volumes/myvm_root
            managed: yes
            size_gb: 3
        vm_userdata: |
            #!/bin/bash
            # WARN: this is not secure
            echo 'root:password' | chpasswd
            
# Deploy `myvm` as vm on `virtualizer` using centos cloud image.
# Attach a first network interface as a macvlan sub-interface to `eth0`
# and a second one to a openvswitch switch named br-ex, trunking tags
# 30-40.
# Configure the vm via cloud-init
-   hosts: myvm
    roles:
    -   role: vm_libvirt_cloudinit
        vm_hoster: virtualizer
        vm_drives:
        -   type: block
            source: /opt/centos7-base.qcow2
            path: /dev/cinder-volumes/myvm_root
            managed: yes
            size_gb: 10
        vm_networks:
        -   type: direct
            parent: eth0
            address: 10.20.0.2/24
            gateway: 10.20.0.1
        -   type: openvswitch
            parent: br-ex
            trunk: "{{range(30, 41)|list}}"
        vm_userdata:
            users:
            -   name: username
                gecos: User Name
                sudo: ALL=(ALL) NOPASSWD:ALL
                groups: users, admin
                lock_passwd: true
                shell: /bin/bash
                ssh_authorized_keys:
                -   ssh-rsa AAA[...]
            write_files:
            -   path: /etc/hosts
                content: |
                    127.0.0.1 localhost
                    10.20.0.2 test
                owner: root:root
                permissions: '0644'
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

If `drive.managed==True` (the default) the role will need a key `size_gb`
to be set, and it will create an image for the drive:
- if `type=='file'`
  - if `path` is not set it will default to `{{vm_dir}}/drive_{{idx}}.img`
  - if `format` is not set it will default to `qcow2`
- if `type=='block'`
  - `path` is required with format `/dev/$vg/$lv`, the corresponding
    logical volume will be created
  - if `format` is not set it will default to `raw`

If `drive.source` is set the drive will be populated with the referenced
source image.

### Role variables
The role uses the following variables, here listed with their default
values:

`````
vm_hoster:
`````
This variable is mandatory. It must be set to the `ansible_hostname` of
the system on which this role will spawn the vm.

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
Cloud-init's user-data.
Can be give an as inline YAML or as a string.
(see [examples](#examples))

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
vm_domain_type: kvm
`````
The `type` attribute of the libvirt `<domain>`.

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
