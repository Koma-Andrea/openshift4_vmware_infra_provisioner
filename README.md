Openshift 4 vmware infra provisioner
=========

This Role provides the simplest way to install openshift 4 taking care of all the network configuration but DNS.

Features:
------------
- **You don't need DHCP**
- check for your environment
- Validate your configuration
- Validate your DNS
- Configure an HaProxy (not deploy it, only configure ATM)
- Configure an Nginx web server (not deploy it, only configure ATM)
- Build the iPXE iso image
- Create the VM
- Boot the vm injecting network configuration
- Install the cluster
- Test all three methods with OKD
- Test all three methods with FCOS (OKD) 

TODO:
------------
- ~~Check for yout environment~~
- ~~Validate your configuration~~
- ~~Validate your DNS~~
- ~~Configure an HaProxy (not deploy it, only configure ATM)~~
- ~~Configure an Nginx web server (not deploy it, only configure ATM)~~
- ~~Build the iPXE iso image~~
- ~~Create the VM~~
- ~~Boot the vm injecting network configuration~~
- ~~Install the cluster~~
- ~~Calculate signature for fcos/rhcos~~
- ~~Test all three methods with FCOS (OKD)~~ 
- Test all three methods with RHCOS (Openshift)
- Miltiple interfaces transpile
- Determine and download images automatically and build a fetch url
- Proxy implementation
- Validate minimal requirements
- Deploy VM for HaProxy and NginX
- Build HA/Vrrp for HaProxy/NginX



Requirements
------------

A Bastion node where execute thos tasks and where install a temporary webserver
Rhel/Centos 8
Ansible 2.9

Role Variables
--------------

```bash
# Please lookup the variables in 
echo defaults/main.yml
```

Supported methods
------------
| Name  | Require OVA| Require DHCP    | Require WebServer  | Need ISO |  Support FCOS | Support RHCOS | Support Baremetal | Description  |
|---|---|---|---|---|---|---|---|---|
| grub   |  ✅ | 🚫  |  ✅  | ✅| ✅  | ✅  | 🚫 | I will generate a iPXE iso and chainload once a grub pxe to inject IP parameters. |
| template   |  ✅ | ✅*  | 🚫  |🚫  | ✅  | ✅  | 🚫 | I will generate and inject an ignition and use the ova template for vmware installation The DHCP is needed for the image to start but the injected parameters in ignition will fix them at first reboot. |
| netinstall   | 🚫|  🚫|  ✅  |  ✅   | ✅  | ✅| 🚫 | I will install the OS via network and inject the ip via kernel parameters |

*In the template mode you need dhcp at first boot because of a bug in coreos/rhcos <br>See [GitHub Issue](https://github.com/coreos/fedora-coreos-tracker/issues/358) and [Fedora COS Docs](https://docs.fedoraproject.org/en-US/fedora-coreos/static-ip-config/)

# Methods description and **backdraft**

Method Grub:
------------
PRO: 
- You don't need to set up DHCP reservation
- You don't need to setup transpile the installer detect the parameter at boot and staticize it.
- Don't leave *trace* in the installation as it just compile for you the parameter for the kernel to stacizie the IP.
- You are using an OVA so you don't need to download a ton of files.

CONS:
- You need a webserver to chainload a grub
- You need to boot from an ISO to chainload Grub
- This method is still not supported by redhat. (But at least you don't leave any trace).
- Still buggy, sometime the lexer cannot interpretate the script at first try.<br>
This will send you to the grub prompt<br>
In this case you need to run the command `reboot` and select boot from CDROM to try again.


Method Template:
------------

PRO: 
- You don't need to set up DHCP reservation
- You don't need a webserver to host PXE informationss
- You are using an OVA so you don't need to download a ton of files.

CONS:
- You need to transpile the network configurations
- Red Hat does not document in any way that transpiling desn't broke your support.

Method netinstall:
------------

PRO:
- You don't need to set up DHCP reservation
- You don't need to transpile the network confiuguration

CONS:
- You need a webserver to host the ignition configuration
- You need an ISO to start the PXE installation
- Red Hat does specify that the installation on vmware is done via OVA (AFAIK does not change anything installing via network or OVA in the final image).



Dependencies
------------
This playbook is tested on RHEL8 and Centos8 with Ansible 2.9
All the dependencies are managed via dnf/yum.


Example Playbook
----------------
```yaml
---
- hosts: all
  remote_user: root
  roles:
    - openshift4_infra_provisioner
```

Example Inventory
----------------
```yaml
---
haproxy:
  hosts:
    bastion.ocp.okd01.lab.local
webserver:
  hosts:
    bastion.ocp.okd01.lab.local
all:
    vars:
      enable_reservation: False
      dns_fail_is_not_fatal: False
      ptr_fail_is_not_fatal: True
      checks_only: False
      #... look for the entire set of variables in defaults/main.yml ...
```

# To run the installation please fill the variables

### Check Installation status:
```bash
openshift-install \
  --dir={{playbook_dir}}/tmp/cluster_conf wait-for bootstrap-complete \
  --log-level=info
```

### To configure the OC client export this variable:  '
```bash
export KUBECONFIG={{playbook_dir}}/tmp/cluster_conf/auth/kubeconfig

oc login \
  -h api.{{ocp_installer.clustername}}{{ocp_installer.basedomain}} \
  -u kubeadmin \
  -p {{ lookup('file', playbook_dir + '/tmp/cluster_conf/auth/kubeadmin-password') }}
```

### To approve the pending nodes you should run this (you may need to do this multiple times):  "
```bash
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
```
### To configure the registry for non production cluster:'
```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'`
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```
