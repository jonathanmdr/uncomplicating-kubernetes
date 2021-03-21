## PROVISIONANDO INSTANCIAS EC2 NA AWS

- Requisitos mínimos:
	- vCPU: 2
	- memória: 2GB

- Sugestão de instância:
	- tipo: t3.small
	- armazenamento: SSD 8GB
	- OS: Linux Ubuntu 18.04 LTS | 20.04 LTS

 > Vale ressaltar que as instâncias EC2 que vão compor o cluster kubernetes devem estar com a comunicação entre elas habilitada, isso pode ser feito através do security group, incluindo todas as máquinas no mesmo grupo

 > Os serviços expostos através do cluster ficam no range de portas: 30000 - 32767

 > Para contralizar a comunicação através de uma única rota é necessário criar um ELB



## INSTALANDO O DOCKER COMO CONTAINER RUNTIME
 > deve ser feito em todos os nodes do cluster

```bash	
$ curl -fsSL https://get.docker.com | bash
$ sudo usermod -aG docker $USER
$ newgrp docker
```



## CRIAR/EDITAR O ARQUIVO daemon.json PARA DEFINIRMOS O DRIVER SENDO O systemd
 > deve ser feito em todos os nodes do cluster

```bash	
$ cat > /etc/docker/daemon.json <<EOF
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




## CRIAR O DIRETORIO DE CONTROLE DE ARQUIVOS DO DOCKER
 > deve ser feito em todos os nodes do cluster

 > este passo se faz necessário pois nós mudamos a configuração padrão do docker para utilizar o cgroupdriver sendo o systemd

```bash	
$ mkdir -p /etc/systemd/system/docker.service.d
```



## RELOAD DO daemon E REINICIANDO O docker PARA QUE ELE ASSUMA AS CONFIGURAÇÕES feitoS ACIMA
 > deve ser feito em todos os nodes do cluster

```bash	
$ systemctl daemon-reload
$ systemctl restart docker
$ docker info | grep -i cgroup
```



## ADICIONANDO A CHAVE GPG PARA PERMITIR BAIXAR AS FERRAMENTAS NECESSARIAS
 > deve ser feito em todos os nodes do cluster

```bash	
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```



## ADICIONANDO O REPOSIORIO DO KUBERNETES NO GERENCIADOR DE PACOTES DO LINUX
 > deve ser feito em todos os nodes do cluster

```bash	
$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
$ apt update
```



## INSTALANDO AS FERRAMENTAS NECESSARIAS PARA GESTAO DO CLUSTER K8S
 > deve ser feito em todos os nodes do cluster

```bash	
$ apt install -y kubeadm kubelet kubectl
$ kubeadm config images pull
```



## CRIANDO O CLUSTER
 > deve ser feito somente no node master

```bash	
$ kubeadm init
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



## HABILITANDO MODULOS DO KERNEL LINUX NECESSARIOS PARA O SUNCIONAMENTO DO CLUSTER (Distribuição Ubuntu já vem inicializado por default)
 > deve ser feito em todos os nodes do cluster

```bash	
$ modprobe br_netfilter ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4 ip_vs
```



## HABILITANDO A COMUNICAÇÃO ENTRE OS PODS DE DIFERENTES NODES
 > deve ser feito somente no node master

 > existem algumas alternativas de ferramentas para realizar este tipo trabalho, aqui nós usaremos o weave net dado a simplicidade de uso do mesmo

```bash	
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```



## GERANDO O TOKEN PARA VINCULAR NODES
> deve ser feito somente no node master

```bash	
$ kubeadm token create --print-join-command
```



## VINCULANDO NODES COM O MASTER PARA FORMAR O CLUSTER
 > este comando deve ser executado no node em que se deseja incluir ao cluster

```bash	
$ kubeadm join {ADRESS_NODE_MASTER}:{PORT_NODE_MASTER} --token {TOKEN} --discovery-token-ca-cert-hash sha256:{CERT_HASH}
```



## MONTANDO O SERVICO DE NFS NOS NODES
 > deve ser feito somente no node master
	
```bash
$ mkdir /opt/dados
$ chmod 1777 /opt/dados
$ nano /etc/exports #Incluir o seguinte valor: /opt/dados *(rw,sync,no_root_squash,subtree_check)
$ exportfs -ar
```

- Visualizando o diretório NFS criado no master
```bash
$ showmount -e {ADRESS_NODE_MASTER}
```
