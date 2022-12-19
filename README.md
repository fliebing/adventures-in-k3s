# Adventures in k3s
As of Dec 5 2022, Versions of:

SW  Used |     version     | Notes   
---------|----------|---------
Ubuntu server | 22.04.1 | minimal install
Longhorn | latest version | 
k3s  | v1.24.8 | Longhorn is incompatible with anything over 1.24
Cert-manager  | 1.10.1  | says it is compatible with kuberentes 1.24)
Metallb | latest | 

---

# Change Control
Please wait for the code to be tested before using
Date | Changes | Tested
---------|----------|---------
 DEC/18/2022 | Initial deploy | YES
 DEC/18/2022 | V1.0 Install script | NO
 
> **_WARNING: Always examine scripts downloaded from the internet before running them locally!!_** > 

---
---

## Why K3s?
This is due to the minimal overhead this kubernetes distro has, I also wanted to make sure to keep my options open in regards to the orchestrator I would use. Rancher is interesting, but it is a bit heavy for my very light 3 node cluster at this time.(I need the resources to run the rest of my tests), so fo rthe time being I will use command line and possibly Portainer so that I can manage my Kubernetes as well as my docker from the same place.
Longhorn is possibly the one that sold me on K3s, as it is done by the same team, I wanted to ensure minimal incompatibilities with such an important piece as my block storage. It is easy to deploy and has a simple, clean and straight-forward UI.
Longhorn has some downfalls though.. It is only block storage. So file and object storage would need to be handled in some other way. My preferred solution in my Proxmox setup is using Ceph (old, yes, but incredibly reliable). Its a great solution, but it requires I devote entire disks to this, which at this time I cannot do for this build as I am using 3 bare metal tiny PC's.
As for reliability I am using old Lenovo Thinkcentre units I picked up some years ago in a going out of business sale, although now you should be able to find them cheap, since the older models are out of support and cannot run Windows 11, so supply of these should be there and we do our bit to reduce e-waste. Their Processor and memory allows for them to be incredibly capable worker nodes (although they have NO upgradeability in regards to disk, you are stuck using the highest size SSD you can get them to recognize). 

## Kubernetes install using K3s in a 3 node cluster
I decided to write this after going through the process of installing a non-standard k3s cluster for my home lab.
Most people are using virtualization and in my case I had a couple (3) of spare machines I could use for this.
The process of going from 0 experience in installing and running a Kubernetes cluster to the final result was a road requiring a lot of reading and troubleshooting every step of the process.
I never thought that moving from docker to Kubernetes would take me as long as it did to accomplish.


### My original plan
The original plan was to set up a 3 node cluster (1 master and 2 workers), but as I am self hosting certain things and I plan to continue going down the microservices develpoment training path, so I decided to change the solution for an HA setup that I would grow in time.
Overall there was nothing wrong with this plan. I start with the 3 nodes as a Lab, then as i get more nodes, I leave them as Master nodes, taint them, and move all workloads to the additional workers I would build over time.
Well, this was the beginning of the journey in getting this set up properly...

## Main decisions
I decided to have 3 master-worker nodes running with an etcd database to ensure there was no problems with having to manage a database replica outside of the cluster, and since etcd is better suited for Kubernetes than using MySQL, this was the way to go.
The decision to use k3s was due to the small footprint and light weight of this solution vs using a full Rancher deployment as at this time the memory requirements of the cluster were my main concern.
etcd has the advantage of being streamlined for this task, but it does require more memory from the nodes than a single database setup and an archive.
The chattiness of the nodes was also something unexpected.

# K3s Component Fault tolerance

## ETCD
Fault tolerance requires an odd number of nodes to function. And it requires that more than half stay up. (See table below)

Total Number of nodes | Failed Node Tolerance
---|---
1|0
3|1
5|2

At 6 I would make 3 master and 3 worker to ensure that I wouldnt accidentally do something that could bring down the cluster by playing around with the different configs and installing more apps.

## Traefik
I have not had a problem with the way Traefik is deployed in the K3s suite, but I do not like that the K3s system overwrites the configs of the pre installed packages, but it is either re-installing this component completely or making a copy of the yaml file and using a different name so k3s does not overwrite it.

​# Installation of K3S HA cluster

## Pre requisites
This is what you don’t figure out until the 3rd or 4th time you run the setup….
1. make sure your hostnames are just alphanumeric, or else there may be some issues with kubernetes, I had some strangeness until I went with simple alphanumeric names, no special characters like hyphens or underscores
2. You ALWAYS need to place a secret passphrase for the cluster install
    1. The instructions don’t tell you this first hand, as they even allow you to get the secret from the first node if you forgot… but the first node generates a K10 token that will not work for the secondary node installs. 
3. Prepare your network, have the kuberentes IP set aside.
    1. This is important since you will want the IP be the one you cluster responds to from your local network.
4. Decide if you will want Longhorn for distributed file system support, (because the current version of Longhorn v1.3.2 does not support kuberentes versions higher than v1.24. The support will start from v1.4.0 of Longhorn) so you will need to install an older version of K3s if you want Longhorn, in my case I will push the envelope and go with 1.24.8.


## Install the following :
Start off by disabling the swap in ubuntu, especially important for these low capacity systems as Kubernetes is not designed to work with swap files
Also preparing for Longhorn and wireguard, and since I like nano to edit the files, it is being installed, and preemptively installing Helm.

```bash
sudo apt update && \
sudo apt upgrade -y && \
sudo swapoff -a -v &&\
sudo rm /swap.img &&\
#back up /etc/fstab: &&\
sudo cp /etc/fstab /etc/fstab.bak &&\
sudo sed -i '/\/swapfile/d' /etc/fstab &&\
sudo sed -i '/\/swap.img/d' /etc/fstab &&\
sudo apt install dialog open-iscsi jq nfs-common tgt -y && \
sudo apt install wireguard nano -y && \
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null && \
sudo apt install apt-transport-https -y && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list && \
sudo apt update && \
sudo apt install helm -y
```


## Install the First Server
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.8+k3s1 \
sh -s - server \
--write-kubeconfig-mode 644 \
--disable servicelb \
--disable local-storage \
--token=YOUR-SECRET \
--tls-san CLUSTER-DNS-NAME --tls-san CLUSTER-LOAD BALNCER-IP \
--cluster-init
```
NOTE: 
	If you want the latest version (and don’t want to install Longhorn in the short term),  remove: INSTALL_K3S_VERSION=v1.23.8+k3s2 \
Here are the definitions of the rest of the options:
   - server - Makes this node a Server node
   —token is for your passphrase 
    –tls-san - Creates a SAN entry that allows the node to be accessible via DNS. You can have as many —tls-san blocks as needed to cover any aliases your load balancer may use (these go into the self signed cert the installation generates for the cluster) 
  –write-kubeconfig-mode Sets write permissions
   —cluster-init tells the system you will be using etcd for the install, this only needs to be placed in the First Server
 

## Now you are ready for the next 2 servers:
Here the config changes slightly, but we will reuse all the command up to the load balancer IP.

The command to use will look like this:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.8+k3s1 \
sh -s - server \
--write-kubeconfig-mode 644 \
--token=YOUR-SECRET \
--tls-san CLUSTER-DNS-NAME --tls-san CLUSTER-LOAD BALNCER-IP \
--server https://IP-OF-THE-FIRST-SERVER:6443
```

Note: 
	If you need to make sure the first line is exactly the same as the one on your first server, you need to install the same version of K3s in all the machines.

    —server is added at the end to specify the server that you will register the new nodes to. As we don’t have any pre existing cluster, it will be the first server we built.

Now check if the nodes came up and registered properly:
```bash
sudo kubectl get nodes -o wide
```
Hopefully you show all the nodes listed, if not, there are several reasons why this may happen, the most common for me was that the certificate did not match (a typo on a long passphrase). Other causes may be related to network issues preventing the nodes from registering one with the other. Please ensure that you are using a hub or a switch that is directly plugged into the nodes to avoid this.
If at all possible use static IPs on the systems, if the DHCP lease changes the node’s IP then you may need to start over, as the certs will not work (you cannot just run the command again to issue a new cert, you will need to re-install)

## Helm
Helm is already installed as you installed the pre-reqs.
It is important to note that you need to run the following, otherwise you will get an “Error: INSTALLATION FAILED: Kubernetes cluster unreachable: ...“ Error message when using helm.

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml 
```
NOTE: You will likely see a warning when using helm that the k3s.yaml is group and world viewable, DO NOT PANIC...the moment you reboot the settings for the k3s file get reverted back to the original, so once you are done installing, you just need to reboot the node and you will not have that security issue.

## cert-manager and Metallb
Traefik actually has Let's Encrypt support built in, so you may be wondering why we are installing another package to the do same thing. Traefik's Let's Encrypt support retrieves certificates and stores them in files. cert-manager retrieves certificates and stores them in Kubernetes secrets. secrets can be simply referenced by name and therefore easier to use.
```bash
helm repo add jetstack https://charts.jetstack.io &&\
helm repo update &&\
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.1 \
  --set startupapicheck.timeout=5m \
  --set installCRDs=true &&\
helm repo add metallb https://metallb.github.io/metallb &&\
kubectl create namespace metallb-system &&\
helm install metallb metallb/metallb --namespace metallb-system
```
To test cert manager you can run the following:
```bash
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```
This will make a test-resources file that we will use to test cert-manager works as expected.

Then make sure cert-manager pods are fully deployed:
```bash
kubectl get pods --namespace cert-manager
```
You should see 3 pods running and ready.

Then we can apply the yaml we just wrote
```bash
kubectl apply -f test-resources.yaml
```
Wait a couple of seconds for the cert to be created and then run:
```bash
kubectl describe certificate -n cert-manager-test
```
The end of that output should show that the certificate was issued successfully 
If so, you can clean up the test resources
```bash
kubectl delete -f test-resources.yaml
```

For the next piece, make sure you place the address range you want metallb to assign to the exposed applications, the simplest to use is V2 (as per metallb site), so this is what I am going with:

```bash
cat <<EOF > metallb-cr.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - YOUR_DEFINED_POOL_START_IP-YOUR_DEFINED_POOL_END_IP
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
EOF
```
Then apply the yaml
```bash
kubectl apply -f metallb-cr.yaml
```

## Longhorn
As I am preparing the environment for my lab, I always end up needing additional storage for some reason or another, and a great way to aggregate the space we have left in the servers is to use software like Longhorn (also from Rancher, so I don’t need to worry too much about compatibility issues, I hope).

Turns out that even after doing the pre-install of the open-iscsi, the Ubuntu 22.04 distro needs some additional configuration to be able to install Longhorn.

Here is a great script to test if ALL of the systems in your cluster are ready for Longhorn: https://github.com/longhorn/longhorn/blob/v1.3.2/scripts/environment_check.sh

(If for some reason it complains it is not executable, just run ```bash sudo chmod +x environment_check.sh```)

With this I figured out my iscsid was not running, so I needed to do the following (on each node)

```bash
sudo nano /etc/iscsi/initiatorname.iscsi 
```
This file will show you a line with an id for the initiator, you need to change it to one that identifies the source.(like this)

Initiator name: iqn.2022-12.YOUR-SUBDOMAIN.SOMETHINGELSE:THIS_NODENUMBER:84de25ddfc37

(Hope that makes sense) Basically change it to be unique to the node.

THEN….
```bash
sudo nano /etc/iscsi/iscsid.conf 
```

Here, you will uncomment the following line and make sure it says “automatic” and not “manual”

```
#*****************
# Startup settings
#*****************
.......
node.startup = automatic
```

After this it should be smooth sailing by running:
```bash
sudo systemctl restart iscsid open-iscsi 
sudo systemctl status iscsid
```

And you should see it be up.. 
An additional item is that my system started crying about multipath being enabled. So preemptively I ran the following in each node:
sudo cat << EOF >> /etc/multipath.conf 
blacklist {
  devnode "^sd[a-z0-9]+"
}
EOF &&\
systemctl restart multipathd

One final check with the script…. And you should be good to go.


## Install longhorn
```bash
helm repo add longhorn https://charts.longhorn.io &&\
helm repo update &&\
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
```
This should end up with a “Longhorn is now installed on the cluster!” message.

You will need to give this a few minutes to finish its process of deploying all the pods

## Where to go from here?
Well it depends on if you are low on resources (like in my case) you would install portainer , if you can spare the resources, you are better off installing Rancher (keeping everything in the same family). Although if I was installing rancher, I would probably install it before Longhorn, just to make sure I can take advantage of the UI and make things easier.


_NOTE:_ I have not tested the compatibility with Rancher.



