# Step-by-step Deployment
Environment information:
Role | OS | Unit | IP | Notes |
-----|----|------|----|-------|
Management server | CentOS 9 | 1 | 172.31.11.10 | Running Ansible |
Opennebula front-end server | Ubuntu 22.04 | 1 | 172.31.11.100 | Opennebula Admin: `oneadmin`/`ciscopass` |
KVM Host servers | Ubuntu 22.04 | 2 | 172.31.11.101; 172.31.11.102 | Hypervisor hosts |
NFS server | Ubuntu 22.04 | 1 | 172.31.11.200 | Network storage |
> 1. All servers set up with an admin account `sysadmin`.
> 2. Otherwise specified, all below commands are executed with `sudo`.

## P1 - Prepare a Management Server with Ansible
1. Prepare a management server for running Ansible software. 
2. Install Ansible and clone deploy scripts from the **one-deploy** git repository.
    ```
    apt install python3-pip
    pip3 install 'ansible-core<2.16'
    git clone git@github.com:OpenNebula/one-deploy.git
    cd ./one-deploy/ && make requirements
    ```

## P2 - Preapare Ohter Servers
1. Prepare the front-end server, KVM servers, and NFS server by installing OS (Ubuntu 22.04).
2. Setup passwordless SSH on those servers so that the management account `sysadmin` can remote login into those servers from the management server.
3. Additonal configuration on the NFS server.
4. Install NFS service in the NFS server.
   ```
   apt update
   apt install nfs-kernel-server
   systemctl start nfs-kernel-server.service  
   ```
5. Create a directory **/storage/one_datastores** and set its ownership to be 9869 as this is the UID/GID assigned to the oneadmin account during the Opennebula installation.
   ```
   mkdir -p /storage/one_datastores
   chmod 9869 9869 /storage/one_datastores
   ```
6. Configure the **one_datestores** directory to be exported by adding them to the **/etc/exports** file.
   ```
   # /etc/exports
   #
   # See exports(5) for more information.
   #
   # Use exportfs -r to reread
   # /export	192.168.1.10(rw,no_root_squash)
   /storage/one_datastores 172.31.11.0/24(rw,async)
   ```
7. Apply the new config.
   ```
   exportfs -a
   ```

## P3 - Running Scripts/Playbook to Deploy OpenNebula Services
1. Log into the management server as **sysadmin**.
2. Open relative yaml file (**shared.yml**).
   ```
   cd one-deploy
   nano inventory/shared.yml
   ```
3. Configure the (**shared.yml**) to meet our invironment.
   ```
   ---
   all:
     vars:
       ansible_user: sysadmin
       one_version: '6.10'
       one_pass: ciscopass
       ds:
         mode: shared
       vn:
         admin_net:
           managed: true
           template:
             VN_MAD: bridge
             PHYDEV: ens33
             BRIDGE: br0
             AR:
               TYPE: IP4
               IP: 192.168.0.50
               SIZE: 100
             NETWORK_ADDRESS: 192.168.0.0
             NETWORK_MASK: 255.255.255.0
             GATEWAY: 192.168.0.1
             DNS: 1.1.1.1
       # Mount NFS share.
       fstab:
         - src: 172.31.11.200:/storage/one_datastores

       features:
         passenger: true
       apache2_http:
         managed: false
       apache2_https:
         managed: true
         key: /etc/ssl/private/one.felixliang.local.key
         certchain: /etc/ssl/certs/one.felixliang.local.crt
       one_fqdn: one.felixliang.local
   
   frontend:
     hosts:
       f1: { ansible_host: 172.31.11.100 }
   
   node:
     hosts:
       n1: { ansible_host: 172.31.11.101 }
       n2: { ansible_host: 172.31.11.102 }
   ```
4. Verify the connection to servers of front-end and KVM hosts. When prompt, enter the password for `sysadmin`.
   ```
   ansible -i inventory/shared.yml all -m ping -b -k
   ```
6. Now we can run the playbook that install and configure OpenNebula services.
   ```
   ansible-playbook -i inventory/shared.yml opennebula.deploy.main -k
   ```
## P4 Verifying the Installation
By following the steps [here](https://github.com/OpenNebula/one-deploy/wiki/sys_verify), perform a quick check to verify your installation and your Opennebula system functions.


