# kubernetes-public
This is a guide to provision a Highly Available Kubernetes Cluster (K8s) on v1.21

<h1>This is my Personal Homelab Kubernetes Cluster, and I've taken around 40 Bookmarked webpages and condensed them into one guide.</h1>

This Kubernetes cluster uses a Multi-Master stacked topology. I stuggled for a few days to find all the steps that would work for one installation on bare metal, and had to take some pieces from several places to get this to work the way it does. This cluster will be using [Calico](https://www.projectcalico.org/) as it's Container Network Interface. 

Please don't hesitate to raise any issues with the guide, I'll update it, and try to help where I can!


<h2> Installation Guide </h2>

  1. Provision 9-13 Ubuntu Server 20.04.1 (At the time of writing) virtual machines with an admin user called "kube" (optional, just personal preference)
      * 2 Load Balancer Servers with minimum 4 cores, and 4GiB of RAM, 32 GiB Storage
      * 3 Control Plane Servers with minimum 4 cores, and 4GiB of RAM, 32 GiB Storage
      * 4 Worker Node Servers with minimum 4 cores, and 8GiB of RAM, 32GiB Storage
      * (Optional) 4 Storage Node Servers with minimum 4 cores, 8GiB of RAM, 100GiB Storage


     These Values are just what work best in *MY* Homelab, and the resources I have available, you can modify them accordingly, just make sure they fall within 
     [Kubernetes minimum Requirements](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)

     The reason I say 9-13 Servers, is the extra 4 are Storage nodes. These nodes are setup with 100GiB each because I planned to use Longhorn as taught in TechnoTim's Guides, so you can do this without the additional storage nodes, or use them as more worker nodes if you don't plan on using storage nodes. 

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
     
     2. Access `keepalived` config at `/etc/keepalived/keepalived.conf` on both nodes. The API server port by default is `6443` Please check the notes below for information on the values.
     
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
        
        vrrp_instance ${VI_#} {
            state ${STATE} # value: MASTER
            interface ${INTERFACE} # value: ens18 
            virtual_router_id ${ROUTER_ID} #value: 51
            priority ${PRIORITY} # value: 101
            authentication {
                auth_type PASS
                auth_pass ${AUTH_PASS} # value: {RandomPassword}
            }
            virtual_ipaddress {
                ${APISERVER_VIP} # value: 192.168.x.x 
            }
            track_script {
                check_apiserver
            }
        }
        ```
        <hr>
        
        * ${VI_#} - This is the instance number. Change it on both load balancers. for simplicity I made mine `VI_1` on the "Master" and `VI_2` on the "Backup".
        * ${STATE} - This value Will be `MASTER` on the first loadbalancer and `BACKUP` on the second loadbalancer.
        * ${INTERFACE} - This is the physical network interface on the Ubuntu Server, change accordingly. Mine is `ens18`
        * ${ROUTER_ID} - This is a random value, just make sure the value on the "Master" loadbalancer is lower. e.g `51` and `52` on the "Backup".
        * ${PRIORITY} - This is a random value, just make the value on the "Master" load balancer is lower. e.g `101` and `200` on the "Backup".
        * ${AUTH_PASS} - This is a random password, use any generator and avoid special characters. This will be the same on both loadbalancers.
        * ${APISERVER_VIP} - This is the virtual IP address that both load balancers will be sharing. Change it based on your local network and available address'. Make sure this address is not within the DHCP servers range. 
        
      3. Add the script that `keepalived` uses at `/etc/keepalived/check_apiserver.sh` on both nodes. the `APISERVER_VIP` is the Virtual IP assigned to `keepalived` above and `APISERVER_DEST_PORT` is `6443`. (Thats the default API Server port)
   
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
        
       

     4. Modify the `HAProxy` configuration file located at `/etc/haproxy/haproxy.cfg` on both nodes

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
                server ${HOST2_ID} ${HOST2_ADDRESS}:${APISERVER_SRC_PORT} check fall 3 rise 2
                server ${HOST3_ID} ${HOST3_ADDRESS}:${APISERVER_SRC_PORT} check fall 3 rise 2
                # [...]
        ```
        <hr>
        
        * ${APISERVER_DEST_PORT} - By default the API Server's port is `6443`, so thats what I am using here.
        * ${HOST1_ID} - This is what the hostname is on the server, I called mine `kube-control-plane1`.
        * ${HOST1_ADDRESS} - This is the IP Address for the first control plane.
        * ${APISERVER_SRC_PORT} - This is the port for the API Server. (`6443`)

     5. Enable `Haproxy` & `keepalived` in `systemd` on both nodes
        
        ```
        sudo systemctl enable haproxy.service
        sudo systemctl enable keepalived.service
        ```

     6. Check to see if you can connect to the virtual IP address assigned by `keepalived if you get a connection refused that is fine because the Kubernetes api server is not yet running. But if you get a timeout error, then check your config. 
        
        `nc -v {VirtualIP} {PORT}`

  8. Prepare Each Node for Kubernetes Install

     1. Disable Firewall on all nodes

        `sudo ufw disable`

     2. Disable Swap on all nodes

        `sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab`  

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
    
     4. Install Docker engine on all the nodes and mark the packages as "held back".

        [Docker Install](https://docs.docker.com/engine/install/)

        
        `sudo apt-mark hold docker-ce docker-ce-cli`
        
     5. Set Docker to use `systemd` as the control group driver. 

        ```
        sudo mkdir /etc/docker
        cat <<EOF | sudo tee /etc/docker/daemon.json
        {
          "exec-opts": ["native.cgroupdriver=systemd"],
          "log-driver": "json-file",
          "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2"
        }
        EOF
        ```
        Then Restart Docker and enable on boot.
        
        ```
        sudo systemctl enable docker
        sudo systemctl daemon-reload
        sudo systemctl restart docker
        ```
        

     6. Install `Kubeadm`, `Kubelet`, and `kubectl` 

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

     `sudo kubeadm init --control-plane-endpoint "${LOAD_BALANCER_VIP}:${LOAD_BALANCER_PORT}" --upload-certs --pod-network-cidr 192.168.0.0/16`
     
      <hr>
      
      * ${LOAD_BALANCER_VIP} - This is the Virtual IP address of `keepalived` assigned above.
      * ${LOAD_BALANCER_PORT} - This is the API Server port assigned above (`6443`).




     ```
     Your Kubernetes control-plane has initialized successfully!

     To start using your cluster, you need to run the following as a regular user:

     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config

     Alternatively, if you are the root user, you can run:

     export KUBECONFIG=/etc/kubernetes/admin.conf

     You should now deploy a pod network to the cluster.
     Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
     https://kubernetes.io/docs/concepts/cluster-administration/addons/

     You can now join any number of the control-plane node running the following command on each as root:

     kubeadm join ${LOAD_BALANCER_IP}:${LOAD_BALANCER_PORT} --token <token> \
	   --discovery-token-ca-cert-hash <discovery-token> \
	   --control-plane --certificate-key <certificate key>

     Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
     As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
     "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

     Then you can join any number of worker nodes by running the following on each as root:

     kubeadm join ${LOAD_BALANCER_IP}:${LOAD_BALANCER_PORT} --token <token> \
	   --discovery-token-ca-cert-hash <discovery-token> 
     ```
     
     
     
     > This is the output fromt he Kubeadm init command. Copy this to a text file for later use. 
     
     <h3>Do not Install your CNI to the cluster yet. I've tried this multiple times and it seems to not let the other nodes join the cluster. If it works for you, great. For me, I had to wait until the whole cluster is bootstrapped before applying the CNI. (Shown in a later step)</h3>

 10. Using the first three lines copy the clusters new `~/.kube/config` to the first control plane's home directory for access 
     ```     
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
     ```
     

 11. Join the other two control planes with the first kubeadm command provided from the output. You will need to add `sudo` to these commands, it doesn't work without it. 

     ```
     sudo kubeadm join ${LOAD_BALANCER_IP}:${LOAD_BALANCER_PORT} --token <token> \
	   --discovery-token-ca-cert-hash <discovery-token> \
	   --control-plane --certificate-key <certificate key>
     ```

 12. Join the worker nodes and storage nodes with the second kubeadm command provided by the output. You will need to add `sudo` to these commands, it doesn't work without it.

     ```
     kubeadm join ${LOAD_BALANCER_IP}:${LOAD_BALANCER_PORT} --token <token> \
	   --discovery-token-ca-cert-hash <discovery-token> 
     ```

 12. Apply the CNI (Container Network Interface) for your cluster. 

     `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`

 13. Copy your kube config to your local machine from the original control plane.

     `sudo cat ~/.kube/config`

 14. You should now have a fully working kubernetes cluster, things to consider are downloading `metallb`, `rancher`, and `longhorn` to make your cluster a little more user friendly.  These Guides can be found from [TechnoTim](https://techno-tim.github.io/)

 15. If you are going to install Rancher on this cluster, I noticed that the schedular and conroller manager said "unhealthy". They seem to work as normal, but a fix for this is as follows:
     1. `sudo nano /etc/kubernetes/manifest/kube-scheduler.yaml` and clear the line (spec->containers->command) containing this phrase: `- --port=0`
     2. `sudo nano /etc/kubernetes/manifest/kube-controller.yaml` and clear the line (spec->containers->command) containing this phrase: `- --port=0`
     3. `sudo systemctl restart kubelet.service`

     Do this on all *THREE* Control Planes, and it should resolve the issue. 
