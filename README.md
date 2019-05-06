# NFVIS Operations

This repo contains tooling to automate NFVIS operations:  It can:

* Create a PXE environment to install NFVIS on the test
* Create packages and install them onto the NFVIS host
* Deploy architectures on multiple NFVIS hosts 
* Test those architectures with packet flows with a [test harness](test_harness.md)
* Clean up architectures multiple NFVIS hosts

### Dependancies:
* [ansible-nfvis](https://github.com/CiscoDevNet/ansible-nfvis)

### Cloning

Since `ansible-nfvis` is included as a submodule, a recurse close is needed:

```yaml
git@wwwin-github.cisco.com:ciscops/nfvis-harness.git --recursive
```

>Note: See the [ansible-nfvis](https://github.com/CiscoDevNet/ansible-nfvis) Ansible Role for an explanation of the attributes.


### Building Packages
```yaml
ansible-playbook packages.yml
```

The playbook expects a data structure defined as follows:
```yaml
nfvis_package_list:
  - name: isrv
    image: isrv-universalk9.16.09.01a-vga.qcow2
    version: 16.09.01a
    template: isrv.image_properties.xml.j2
  - name: ubuntu
    image: ubuntu_18.04.img
    version: 18.04
    template: ubuntu.image_properties.xml.j2
    options:
      low_latency: no
  - name: vedge
    image: viptela-edge-18.4.0-genericx86-64.qcow2
    version: 18.4.0
    template: vedge.image_properties.xml.j2
  - name: asav
    image: asav9101.qcow2
    version: 9.10.1
    template: asav.image_properties.xml.j2
  - name: centos
    image: CentOS-7-x86_64-GenericCloud-1901.qcow2
    version: 1901
    template: centos.image_properties.xml.j2
```

Templates are found in `ansible-nfvis/templates`

##### Extra Vars:
* `package`: only build the package with the name specified
* `nfvis_package_dir`: location of image files used to build the packages in the directory specified by `nfvis_package_dir` (Default: `"{{ playbook_dir }}/packages"`)
* `nfvis_temp_dir`: temp location for building packages (Default: /tmp/nfvis_packages)
* `nfvis_package_dir`: location for storing built packages (Default: `"{{ playbook_dir }}/packages"`)

```yaml
ansible-playbook packages.yml -e package=asav
```

>Note: Since nfvis_deployment inject the config into the deployments, this task does not include any configuration.

### Provision the NFVIS hosts
```yaml
ansible-playbook -i inventory/isr_asa1.yml provision.yml
```
* Sets the hostname
* Sets trusted source
* Uploads and registers packages
* Creates VLANs, Bridges, and Networks

The playbook expects a data structure defined as follows:
#### Data Structure
##### VLANs:
```yaml
nfvis_vlans:
  - 10
  - 11
```

##### Bridges:

```yaml
nfvis_bridges:
  service:
    port:
      - 'GE0-1'
```

##### Networks:
```yaml
nfvis_networks:
  dmz1:
    vlan: 10
    trunk: 'false'
    bridge: lan-br
  internal1:
    vlan: 11
    trunk: 'false'
    bridge: lan-br
  service:
    vlan: 111
    trunk: 'false'
    bridge: service
```

##### Tags:
* 'system': Just run the play to set the system settings
* 'network': Just setup VLANs, bridges, and networks
* 'packages': Just run the play to upload the packages

```yaml
ansible-playbook -i inventory/isr_asa1.yml provision.yml --tags=upload
```

##### Extra Vars:
* 'package': only build the package with the name specified

```yaml
ansible-playbook -i inventory/isr_asa1.yml provision.yml --tags=upload -e package=asav
```

### Build a topology
```yaml
ansible-playbook -i inventory/isr_asa1.yml build.yml
```
* Creates VLANs on all hosts in the `encs` group
* Creates Bridges and Networks on all hosts in the `nfvis` group
* Deploys the VNFs in the `vnf` group

To limit to a single VNF:
```yaml
ansible-playbook -i inventory/isr_asa1.yml build.yml --limit=asav
```

##### Data Structure
```yaml
nfvis:
    name: "{{ inventory_hostname }}"
    host: "{{ hostvars['dut1'].ansible_host }}"
    image: ubuntu
    flavor: ubuntu-small
    bootup_time: 600
    port_forwarding:
      - proxy_port: "{{ ansible_port }}"
        source_bridge: 'wan-br'
    interfaces:
      - network: int-mgmt-net
      - network: lan-net
    config_data:
      - dst: meta-data
        data: "{{ lookup('template', 'ubuntu.meta-data.j2') }}"
      - dst: user-data
        data: "{{ lookup('template', 'ubuntu.user-data.j2') }}"
      - dst: network-config
        data: "{{ lookup('template', 'ubuntu.network-config.j2') }}"
```

##### Tags
* 'vlan': Just run the play to build the VLANs
* 'network': Just run the play to build the Networks
* 'vnf': Just run the play to deploy the VNFs



### Playbooks

#### Deploying the PXE server:
```yaml
ansible-playbook -i inventory/pxe.yml build.yml
```

Configuring the PXE server:
```yaml
ansible-playbook -i inventory/pxe.yml configure_pxe.yml
```





#### Check a Topology
```yaml
ansible-playbook -i inventory/isr_asa1.yml check.yml
```

* Waits for the VNFs to become reachable
* Checks connectivity from the hosts in the `control_host` group to the hosts in the `test_host` group

#### Clean up a Topology
```yaml
ansible-playbook -i inventory/isr_asa1.yml clean.yml
```
* Deletes the VNFs in the `vnf` group
* Deletes the Bridges and Networks on all hosts in the `nfvis` group
* Deletes the VLANs on all hosts in the `encs` group

To limit to a single VNF:
```yaml
ansible-playbook -i inventory/isr_asa1.yml clean.yml --limit=asav
```

##### Tags:
* 'vlan': Just run the play to clean the VLANs
* 'network': Just run the play to clean the Networks
* 'vnf': Just run the play to clean the VNFs

