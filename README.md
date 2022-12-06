# Despliegue-de-cluster-de-kubernetes-con-la-herramienta-kubespray
En el siguiente documento se realizara la instalacion de kubespray la cual es muy usada en ambiente industrial, esta es montada sobre la VM en la cual se despliega


# Herramientas necesarias

Antes de proceder con la instalacion se necesitan de unas herramientas previas las cuales son:

- [helm-install](https://github.com/mgodll/Despliegue-y-gestion-de-contenedores-usando-la-herramienta-helm)
- [Kubectl](https://github.com/mgodll/Despliegue-y-gestion-de-contenedores-usando-la-herramienta-helm)
- [Docker](https://github.com/mgodll/Despliegue-y-gestion-de-contenedores-usando-la-herramienta-helm)

# Requisitos de maquina

Requisitos minimos para la instalacion

| Estructura       | Valores            |
|------------------|--------------------|
|OS usado          | Ubuntu 20.04 LTS   |
|vCPU              | 8                  |
|RAM (GB)          | 16                 |
|Disk (GB)         | 140                |
|Home user         | mateo              |
|Number of NICs    | 2 (ens160, ens192) |
|Kubespray Version | v2.19.1            |



> **NOTA** se recomienda habilitar dos interfaces de red para vincular la maquina a un PVC o un volumen

# Exportacion de variables de entorno

Se exportaran unas variables para hacer el montaje mas comodo

```bash
export NODE_USER="mtoror"
export NODE_HOSTNAME_PREFIX="mateo"
export NODE_IP="10.182.0.10"
export KUBESPRAY_TAG="v2.19.1"
export CLUSTER_NAME="idt-cluster"
```

> **NOTA** se recomienda usar los siguientes comandos para saber el NODE_USER (whoami) y NODE_HOSTNAME_PREFIX (hostname), el node ip es la ip con acceso a internet de la maquina

# Configurando servidor

Como primer paso se actualizara la interfaz y se realizara una modificacion de privilegios

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install vim
sudo visudo
```

Siguiente se abrira una ventana y se anexara

```bash
# User privilege specification
root    ALL=(ALL:ALL) ALL
+ $NODE_USER  ALL=(ALL) NOPASSWD:ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
```

> **NOTA** Para todo los procesos a continuacion se recomiendo estar dentro de super usuario con el comando `sudo su` para tener privilegios de `root`

Para crear una key con auto-acces en el servisor se asignaran un usuario y contraseña de la siguiente manera

```bash
ssh-keygen
# Generating public/private rsa key pair.
# Enter file in which to save the key (/root/.ssh/id_rsa):
# Enter passphrase (empty for no passphrase):
# Enter same passphrase again:
# Your identification has been saved in /root/.ssh/id_rsa.
# Your public key has been saved in /root/.ssh/id_rsa.pub.
# The key fingerprint is:
# SHA256:t0qr5YF1pg9GqE2MvwVkjzamiNYUIvLUsc/KX47znis root@kubernetes
# The key's randomart image is:
# +---[RSA 2048]----+
# |    .            |
# |   . o           |
# |o...o o          |
# |oo. .B +         |
# |  ... % S +      |
# | .oo X * = .     |
# |....= + X .      |
# |.    .E@ B       |
# |      =*X..      |
# +----[SHA256]-----+
ssh-copy-id -i ~/.ssh/id_rsa $NODE_USER@$NODE_IP
# /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
# /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
# /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
# ubuntu@10.1.3.152's password:
# Number of key(s) added: 1
# 
# Now try logging into the machine, with:   "ssh 'ubuntu@10.1.3.152'"
# and check to make sure that only the key(s) you wanted were added.
```

> **NOTA** si en llegado caso al tirar el comando `ssh-copy-id -i ~/.ssh/id_rsa $NODE_USER@$NODE_IP` retorna `$NODE_USER@localhost: Permission denied (publickey)`, se recomienda ingresar lo siguiente a la terminar con el fin de copiar manual las key `cat ~/.ssh/id_rsa.pub >> /home/$NODE_USER/.ssh/authorized_keys`

ahora se accede al cluster o se inicia sesion en el servidor creado

```bash
ssh $NODE_USER@$NODE_IP
```

> **NOTA** En este paso al acceder a la maquina nos sacara de super usuario, volver acceder y seguin con el proceso como root

Se instala Python como requisito para la maquina

```bash
sudo apt install python3-pip
sudo pip3 install --upgrade pip
```

Clonar el repositorio de Kuberspray

```bash
# Clone Kubespray repository
cd && git clone https://github.com/kubernetes-sigs/kubespray.git ~/kubespray
cd ~/kubespray && git checkout tags/$KUBESPRAY_TAG -b $KUBESPRAY_TAG
# Install dependencies from requirements.txt
sudo pip3 install -r requirements.txt
# Copy inventory/sample as inventory/$CLUSTER_NAME
cp -rfp inventory/sample inventory/$CLUSTER_NAME
# declare IPs of the ubuntu nodes
declare -a IPS=($NODE_IP)
# Update the ansible inventory with the variables related to the nodes
HOST_PREFIX=$NODE_HOSTNAME_PREFIX CONFIG_FILE=inventory/$CLUSTER_NAME/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

> **NOTA** En la parte de `${IPS[@]}` no tiene necesidad de reemplazar la direccion ip

Se creara un .yaml con el fin de desplegar la maquina y darle la informacion necesaria para ello se ingresa lo siguiente

```bash
sudo vim ~/kubespray/inventory/$CLUSTER_NAME/deploy_ansible_vars.yaml
```

Y anexamos lo siguiente

```bash
# Ansible variables
ansible_user: $NODE_USER
bootstrap_os: $NODE_USER

# App variables
helm_enabled: true
kubevirt_enabled: false
local_volume_provisioner_enabled: true

# Networking configuration
kube_network_plugin: calico
kube_network_plugin_multus: true
multus_version: stable
kube_pods_subnet: 10.233.64.0/18
kube_apiserver_port: 6443
kube_network_node_prefix: 24
kube_service_addresses: 10.233.0.0/18
kube_proxy_mode: ipvs
kube_apiserver_node_port_range: 30000-36767

# Kubernetes configuration
kubeadm_control_plane: true
etcd_kubeadm_enabled: true
kube_config_dir: /etc/kubernetes
kube_version: v1.23.7
dashboard_enabled: true

# cGroup configuration
docker_cgroup_driver: systemd
kubelet_cgroup_driver: systemd

# DNS configuration
dns_mode: coredns
resolvconf_mode: host_resolvconf
dns_domain: cluster.local
enable_nodelocaldns: true
nameservers:
  - 8.8.8.8
  - 8.8.4.4
upstream_dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# Swap configuration
disable_swap: true
```

verificamos haciendo un cat a la configuracion del .yaml `cat ~/kubespray/inventory/$CLUSTER_NAME/hosts.yml`

```bash
all:
  hosts:
    kubernetes1:
      ansible_host: 10.1.3.152
      ip: 10.1.3.152
      access_ip: 10.1.3.152
  children:
    kube_control_plane:
      hosts:
        kubernetes1:
    kube_node:
      hosts:
        kubernetes1:
    etcd:
      hosts:
        kubernetes1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

# Despliegue del cluster

Como ultimo paso se desplegara el cluster de kubernetes usando

```bash
cd ~/kubespray
ansible-playbook -b -i inventory/$CLUSTER_NAME/hosts.yml cluster.yml --user $NODE_USER -e @inventory/$CLUSTER_NAME/deploy_ansible_vars.yaml
```

Se espera a que el proceso termine, normalmente suele durar entre 10 a 15 minutos

> **NOTA** Para los siguientes pasos se sale del root, con el comando exit

Se configura el archivo Kubeconfig para el servidor con las siguientes lineas

```bash
# In the server:
mkdir -p ~/.kube
sudo cat /root/.kube/config > ~/.kube/config
sudo chown $NODE_USER:$NODE_USER ~/.kube/config
sudo chmod go-r ~/.kube/config
```

Y por ultimo se verifica que el cluster de kubernetes este en linea con kubectl

```bash
kubectl get pods -n kube-system
NAME                                          READY   STATUS             RESTARTS        AGE
calico-kube-controllers-6dd874f784-klbvw      1/1     Running            1 (3m52s ago)   3m54s
calico-node-xc6zt                             1/1     Running            0               4m31s
coredns-76b4fb4578-q48sh                      1/1     Running            0               3m22s
dns-autoscaler-7874cf6bcf-ddhws               1/1     Running            0               3m18s
etcd-pruebas1                                 1/1     Running            0               5m31s
kube-apiserver-pruebas1                       1/1     Running            1               5m31s
kube-controller-manager-pruebas1              1/1     Running            1               5m31s
kube-multus-ds-amd64-52k97                    0/1     CrashLoopBackOff   7 (43s ago)     3m58s
kube-proxy-5cwjd                              1/1     Running            0               4m34s
kube-scheduler-pruebas1                       1/1     Running            1               5m31s
kubernetes-dashboard-584bfbb648-8rgz2         1/1     Running            0               3m14s
kubernetes-metrics-scraper-5dc755864d-nx2vz   1/1     Running            0               3m14s
local-volume-provisioner-2j8x6                1/1     Running            0               3m39s
nodelocaldns-6chf2                            1/1     Running            0               3m17s
```
