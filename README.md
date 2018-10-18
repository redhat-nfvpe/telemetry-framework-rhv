# telemetry-framework-rhv
Automation for installation of telemetry framework on RHV. Purpose of this repository
is to provide documentation and automation for:

* installing a RHV engine
* setting up a virtual machine host server
* importing a RHEL 7.5 template
* instantiating RHEL 7.5 virtual machines

# Prerequisites

You'll need Ansible 2.6 or later along with a host machine already running
RHEL 7.5. These instructions are generally applicable to oVirt as well but
some modification of configuration will be required. Installing on top of
CentOS for oVirt is not in scope, but easily done with some extra effort.

You'll also need a bastion host (your laptop, or another place to run the
Ansible playbooks against your host server).

# Topology

TODO

# Host Setup

Host setup will be dealt with in the following sections.

## Download RHEL 7.5 cloud image

You're downloading the image and _running the following commands on the **host** itself_.
In the next section we'll swap over to our bastion host to automate the rest.

Get the download link for your RHEL 7.5 cloud image from https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.5/x86_64/product-software

You're looking for the latest _Red Hat Enterprise Linux 7.5 Update KVM Guest Image (20180925)_ (or newer).

Create a directory for the image to live.

    mkdir -p /home/images/engine/

Once you've done that, download it onto the host machine with `curl`. The image should live
in `/home/images/engine/`. We'll use this to spin up our initial `engine` virtual machine.

    curl https://access.cdn.redhat.com//content/origin/files/sha256/<unique_string>/rhel-server-7.5-update-4-x86_64-kvm.qcow2?_auth_=<unique_auth_from_link> \
      --output /home/images/engine/rhel-server-7.5-update-4-x86_64-kvm.qcow2

## Setup repositories and register RHEL

The following sections will be executed on your bastion host.

### Clone `telemetry-framework-rhv`

    mkdir -p ~/src/github/redhat-nfvpe/ && cd $_
    git clone https://github.com/redhat-nfvpe/telemetry-framework-rhv
    cd telemtry-framework-rhv

### Setup `telemetry-framework-rhv` configuration

We need to create an inventory file that we'll be able to run our
playbooks against to setup the nodes. Here is an example inventory file
for the HA lab.

    cat > inventory/hosts.yml <<EOF
    all:
      hosts:
        manager:
          ansible_host: 10.19.110.80
          ansible_private_key_file: /home/lmadsen/.ssh/halabdot7
          ansible_user: cloud-user
          password_file: engine.yml
        virthost:
          ansible_host: 10.19.110.7
          ansible_private_key_file: /home/lmadsen/.ssh/id_rsa
          ansible_user: root
          password_file: virthost.yml
      children:
        virthosts:
          hosts:
            manager:
            virthost:
        engine:
          hosts:
            manager:
    # vim: set tabstop=2 shiftwidth=2 smartindent expandtab :
    EOF

### Create password files for RHEL subscription

Create two files with `ansible-vault` which will hold our credentials
and Red Hat subscription information.

    ansible-vault create vars/engine.yml
    ansible-vault create vars/virthost.yml
    
Then we need to put in some configuration contents to these files.

**vars/engine.yaml**

    ansible-vault edit vars/engine.yml
    
    ovirt_engine_setup_admin_password: admin
    engine_password: "{{ ovirt_engine_setup_admin_password }}"

    ovirt_repositories_rh_username: "user@email.tld"
    ovirt_repositories_rh_password: "secretpassword"
    ovirt_repositories_pool_ids:
        - 8a85....26ba

**vars/virthost.yaml**

    ansible-vault edit vars/virthost.yml
    
    ovirt_engine_setup_admin_password: admin
    engine_password: "{{ ovirt_engine_setup_admin_password }}"

    ovirt_repositories_target_host: host
    ovirt_repositories_rh_username: "user@email.tld"
    ovirt_repositories_rh_password: "secretpassword"
    ovirt_repositories_pool_ids:
        - 8a85....39e0

### Subscribe and setup repositories

With our subscription information all setup, we can run the `bootstrap.yml`
playbook to subscribe our server and setup the repositories.

    ansible-playbook -i inventory/hosts.yml --ask-vault-pass playbooks/bootstrap.yml

# RHV Engine virtual machine creation

Next up is creating our virtual machine for the engine. These commands are also
run on your bastion host.

## Setup `base-infra-bootstrap` inventory and vars

We'll leverage the work done in the `base-infra-bootstrap` repository to make
creating the initial virtual machine a little bit less tedious. We're running
the commands below on our bastion host.

    mkdir -p ~/src/github/redhat-nfvpe/ && cd $_
    git clone https://github.com/redhat-nfvpe/base-infra-bootstrap
    cd base-infra-bootstrap
    ansible-galaxy install -r requirements.yml

Create our `inventory` to point at the host.

    mkdir inventory/rhv_engine
    cat > inventory/rhv_engine/inventory <<EOF
    virt_host ansible_host=10.19.110.7 ansible_ssh_user=root

    [virthosts]
    virt_host

    [all:vars]
    ansible_ssh_common_args='-o StrictHostKeyChecking=no'
    EOF

Then create the `vars` for configuring the virtual machine.

    cat > inventory/rhv_engine/vars <<EOF
    centos_genericcloud_url: file:///home/images/engine/rhel-server-7.5-update-4-x86_64-kvm.qcow2
    image_destination_name: rhel-server-7.5-update-4-x86_64-kvm.qcow2
    host_type: "centos"
    images_directory: /home/images/engine
    spare_disk_location: /home/images/engine
    spare_disk_size_megs: 20480
    ssh_proxy_user: root
    ssh_proxy_host: 10.19.110.7
    vm_ssh_key_path: /home/lmadsen/.ssh/halabdot7
    install_openshift_ansible_deps: false

    # network configuration for bridged network on virtual machine
    bridge_networking: true
    bridge_name: br1
    bridge_physical_nic: "eno1"
    bridge_network_name: "br1"
    bridge_network_cidr: 10.19.110.0/24
    domain_name: dev.nfvpe.site

    # network configuration for spinup.sh in redhat-nfvpe.vm-spinup role
    system_network: 10.19.110.0
    system_netmask: 255.255.255.0
    system_broadcast: 10.19.110.255
    system_gateway: 10.19.110.254
    system_nameservers: 10.19.110.9
    system_dns_search: dev.nfvpe.site

    # list of virtual machines to create
    virtual_machines:
      - name: engine
        node_type: other
        system_ram_mb: 8192
        static_ip: 10.19.110.80

## Create engine virtual machine

    cd ~/src/github/redhat-nfvpe/base-infra-bootstrap
    ansible-playbook -i inventory/rhv_engine/inventory -e "@./inventory/rhv_engine/vars" playbooks/virt-host-setup.yml

# Engine Configuration

Engine configuration will be dealt with in the following sections.
