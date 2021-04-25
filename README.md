# kubernetes-public
This is a guide to provision a Highly Available Kubernetes Cluster (K8s) on v1.21

<h1>This is my Personal Homelab Kubernetes Cluster, and I've taken around 40 Bookmarked webpages and condensed them into one guide.</h1>

This homelab uses a Multi-Master stacked topology. 


<h2> Installation Guide </h2>

  1. Provision 9-13 Ubuntu Server 20.04.1 (At the time of writing) virtual machines with an admin user called "kube" (optional, just personal preference)
      * 2 Load Balancer Servers with minimum 4 cores, and 4GiB of RAM, 32 GiB Storage
      * 3 Control Plane Servers with minimum 4 cores, and 4GiB of RAM, 32 GiB Storage
      * 4 Worker Node Servers with minimum 4 cores, and 8GiB of RAM, 32GiB Storage
      * (Optional) 4 Storage Node Servers with minimum 4 cores, 8GiB of RAM, 100GiB Storage


     These Values are just what work best in *MY* Homelab, and the resources I have available, you can modify them accordingly, just make sure they fall within 
     [Kubernetes minimum Requirements](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)

     The reason I say 9-13 Servers, is the extra 4 are Storage nodes. These nodes are setup with 100GiB each because I planned to use Longhorn as taught in TechnoTim's Guides, so you can do this without if you don't plan on having storage nodes. 

  2. Setup an additional user on each Ubuntu Server with Sudo privelages (so that the machine can be managed by ansible):

     `sudo adduser serveradmin`
 
     Followed by:
     
     `sudo usermod -aG sudo serveradmin`
  
  3. Setup SSH Key based authentication, and copy the ID's to each server
 
     `ssh-keygen #Run this on your Local Development Machine`
      
     Followed by: 
      
     `ssh-copy-id {user}@{IP Address} # Do this for both kube & serveradmin`

  4. Install Ansible on your Local Machine:

     ```
     sudo apt update
     sudo apt install ansible
     sudo apt install sshpass # This is needed if you are using password based authentication.
     ``` 

     [Ansible Galaxy Community General](https://galaxy.ansible.com/community/general)

     `ansible-galaxy collection install community.general`

     [Ansible Galaxy Gantsign.OhMyZsh](https://galaxy.ansible.com/gantsign/oh-my-zsh)

     `ansible-galaxy install gantsign.oh-my-zsh`

  5. Provision the 13 Servers using ansible using TechnoTim's Ansible Files. the Host file `kuber` has all my kubernetes hosts. [TechnoTim Ansible](https://github.com/techno-tim/ansible-homelab)

     ```
     ansible-playbook ~/ansible/playbooks/apt.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber
     ansible-playbook ~/ansible/playbooks/apt-dist.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber
     ansible-playbook ~/ansible/playbooks/qemu-guest-agent.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber
     ansible-playbook ~/ansible/playbooks/timezone.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber
     ansible-playbook ~/ansible/playbooks/zsh.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber
     ansible-playbook ~/ansible/playbooks/oh-my-zsh.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber # also do this for kube user if desired
     ansible-playbook ~/ansible/playbooks/resize-lvm.yml --user serveradmin --ask-become-pass -i ~/ansible/inventory/kuber
     ```
     > This should provision all the Ubuntu Servers and have them prepped for a Kubernetes cluster installation


  6. Install Kubectl on your local dev machine: [Install Kubernetes Tools](https://kubernetes.io/docs/tasks/tools/)

  7. Create the highly-available load-balancer with a virtual IP address for access to the kubernetes API Server endpoint

     1. ssh into both load balancer nodes, and install `keepalived` and `haproxy`
       
        `sudo apt install keepalived haproxy`
     
     2. Access `keepalived` config at `/etc/keepalived/keepalived.conf` on both nodes. The API server port by default is `6443`
     
        ```
        global_defs {
            router_id LVS_DEVEL
        }
        vrrp_script check_apiserver {
          script "/etc/keepalived/check_apiserver.sh"
          interval 3
          weight -2
          fall 10
          rise 2
        }
        
        vrrp_instance VI_1 {
            state ${STATE} # value: MASTER
            interface ${INTERFACE} # value: ens18 (this is the physical network interface on the Ubuntu Server, change accordingly)
            virtual_router_id ${ROUTER_ID} #value: 51
            priority ${PRIORITY} # value: 101
            authentication {
                auth_type PASS
                auth_pass ${AUTH_PASS} # value: {RandomPassword}
            }
            virtual_ipaddress {
                ${APISERVER_VIP} # value: 192.168.x.x (This is dependent on the local network, and available address'. Make sure this Address is not within the DHCP servers address' that it will assign ) 
            }
            track_script {
                check_apiserver
            }
        }
        ```
   
        Add the script that `keepalived` uses at `/etc/keepalived/check_apiserver.sh` on both nodes. the `APISERVER_VIP` is the Virtual IP assigned to `keepalived` above
   
        ```
        #!/bin/sh
        
        errorExit() {
            echo "*** $*" 1>&2
            exit 1
        }
        
        curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
        if ip addr | grep -q ${APISERVER_VIP}; then
            curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
        fi
        ```

     3. Modify the `HAProxy` configuration file located at `/etc/haproxy/haproxy.cfg` on both nodes

        ```
        # /etc/haproxy/haproxy.cfg
        #---------------------------------------------------------------------
        # Global settings
        #---------------------------------------------------------------------
        global
            log /dev/log local0
            log /dev/log local1 notice
            daemon
        
        #---------------------------------------------------------------------
        # common defaults that all the 'listen' and 'backend' sections will
        # use if not designated in their block
        #---------------------------------------------------------------------
        defaults
            mode                    http
            log                     global
            option                  httplog
            option                  dontlognull
            option http-server-close
            option forwardfor       except 127.0.0.0/8
            option                  redispatch
            retries                 1
            timeout http-request    10s
            timeout queue           20s
            timeout connect         5s
            timeout client          20s
            timeout server          20s
            timeout http-keep-alive 10s
            timeout check           10s
        
        #---------------------------------------------------------------------
        # apiserver frontend which proxys to the masters
        #---------------------------------------------------------------------
        frontend apiserver
            bind *:${APISERVER_DEST_PORT}
            mode tcp
            option tcplog
            default_backend apiserver
        
        #---------------------------------------------------------------------
        # round robin balancing for apiserver
        #---------------------------------------------------------------------
        backend apiserver
            option httpchk GET /healthz
            http-check expect status 200
            mode tcp
            option ssl-hello-chk
            balance     roundrobin
                server ${HOST1_ID} ${HOST1_ADDRESS}:${APISERVER_SRC_PORT} check fall 3 rise 2 # for me ${HOST1_ID} is kube_control_plane1. Add all your control Planes here. 
                # [...]
        ```

     4. Enable `Haproxy` & `keepalived` in `systemd` on both nodes
        
        ```
        sudo systemctl enable haproxy.service
        sudo systemctl enable keepalived.service
        ```

     5. Check to see if you can connect to the virtual IP address assigned by `keepalived if you get a connection refused that is fine because the Kubernetes api server is not yet running. But if you get a timeout error, then check your config. 
        
        `nc -v {VirtualIP} {PORT}`

  8. Prepare Each Node for Kubernetes Install

     1. Disable Firewall on all nodes

        `sudo ufw disable`

     2. Disable Swap on all nodes

        `swapoff -a; sed -i '/swap/d' /etc/fstab`  

     3. Make sure IPTables can see bridged Traffic on all nodes

        ```
        cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
        br_netfilter
        EOF
       
        cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        EOF
        sudo sysctl --system
        ``` 
    
     4. Install Docker engine on all the nodes and mark the packages as "held back"

        [Docker Install](https://docs.docker.com/engine/install/)

        
        `sudo apt-mark hold docker-ce docker-ce-cli`

     5. Install `Kubeadm`, `Kubelet`, and `kubectl` 

        ```
        sudo apt-get update
        sudo apt-get install -y apt-transport-https ca-certificates curl

        sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

        echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

        sudo apt-get update
        sudo apt-get install -y kubelet kubeadm kubectl
        sudo apt-mark hold kubelet kubeadm kubectl

        ```
  9. Intiliaze the first control plane on the cluster and copy the contents of the output to a text file for later use.

     `sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs --pod-network-cidr 192.168.0.0/16`

 10. Join the other two control planes and then join the worker nodes with the commands provided by the output, and also make sure you copy the clusters kube config using the commands

 11. Apply the CNI for your cluster 

     `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`

 12. Copy your kube config to your local machine from the original control plane.

     `sudo cat ~/.kube/config`

 13. You should now have a fully working kubernetes cluster, things to consider are downloading `metallb`, `rancher`, and `longhorn` to make your cluster a little more user friendly.  These Guides can be found from [TechnoTim](https://techno-tim.github.io/)

 14. If you are going to install Rancher on this cluster, I noticed that the schedular and conroller manager said "unhealthy". They seem to work as normal, but a fix for this is as follows:
     1. `sudo nano /etc/kubernetes/manifest/kube-scheduler.yaml` and clear the line (spec->containers->command) containing this phrase: `- --port=0`
     2. `sudo nano /etc/kubernetes/manifest/kube-controller.yaml` and clear the line (spec->containers->command) containing this phrase: `- --port=0`
     3. `sudo systemctl restart kubelet.service`

     Do this on all *THREE* Control Planes, and it should resolve the issue. 
