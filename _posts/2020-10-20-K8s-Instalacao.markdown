---
layout: post
title:  "Instalação do Kubernetes"
date:   2020-10-20 17:06:38 -0300
categories: jekyll update
---

Este documento foi desenvolvido em **março de 2020**, durante a disciplina DevOps na Residência em tecnologia da informação aplicada à área jurídica - turma JF/IMD, na área de especialidade de Redes e Infraestrutura, ministrado pelo Prof Me. André Solino.   

O objetivo deste tutorial é a instalação do Kubernetes.

Este tutorial abordará os tópicos listados abaixo:
1. Criação das instâncias
2. Container runtimes  - Docker
3. Iniciando Clusters com o kubeadm
4. Comandos interessantes
5. Referências


## Criação das instâncias

Foram criadas três máquinas, onde uma será o master e as outras duas os workers.
Todas elas com 2vCPU e o centOS7.
Para facilitar o trabalho, em casos de comandos repetidos usamos o tilix para digitar nos três terminais de uma vez.
Todas foram logadas como root.


## Container runtimes  - Docker

Para usar os containers nos pods, o K8s usa alguns Container runtimes, em nosso caso será o Docker, então, vamos para sua instalação. 
**Esse processo deve ser feito em todas as máquinas!**
Logue como root: sudo su -
```
# Install Docker CE
## Set up the repository
### Install required packages.
yum install -y yum-utils device-mapper-persistent-data lvm2

### Add Docker repository.
yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo

## Install Docker CE.
yum update -y && yum install -y \
  containerd.io-1.2.10 \
  docker-ce-19.03.4 \
  docker-ce-cli-19.03.4

## Create /etc/docker directory.
mkdir /etc/docker

# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
systemctl daemon-reload

systemctl restart docker

systemctl enable docker
```


##  Iniciando Clusters com o kubeadm

Como já foi dito nos outros documentos, o kubeadm  automatiza parte do processo de criação do cluster.

Como estamos trabalhando em um laboratório para ambiente de produção, iremos parar e desabilitar o firewalld. Caso você esteja em um ambiente de produção real é importante rever a documentação deste serviço para que ele fique habilitado e não abra falhas de segurança na sua implementação.

1. Parando o firewalld:
```
systemctl stop firewalld
systemctl disable firewalld
```
2. Portas que devem estar liberadas:
* Master: 6443*, 2379-2380, 10250, 10251 e 10252 .
* Workers: 10250 e 30000-32767.


### Instalação do kubeadm, kubelet and kubectl
Como dito acima, o **kubeadm** automatiza parte do processo de criação do cluster. O **kubelet** faz a interface com o docker, atua como agente em cada node e o **kubectl** é a interface de linha de comando do K8s.
**Esse processo deve ser feito em todas as máquinas!**
É necessário definir o SELinux no modo permissivo executando os comando abaixo. Isso irá acontecer até que o suporte ao SELinux seja aprimorado no kubelet.
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```
Alguns usuários do RHEL / CentOS 7 relataram problemas com o tráfego sendo roteado incorretamente devido ao desvio do iptables. Você deve garantir que net.bridge.bridge-nf-call-iptables esteja definido como 1 na sua configuração sysctl, por exemplo:
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
Para ter certeza que o módulo br_netfilter foi um módulo carregado no kernerl, utilizae o lsmod que é um comando para sistemas operacionais Linux que exibe quais módulos carregáveis do kernel estão atualmente carregados.
```
lsmod | grep br_netfilter
```
Para que você fique com a versão mais atualizada do kubeadm, rode:
```
yum update
```
Aqui, caso seja mostrado alguma atualização do docker, você não deve marcar para atualizar os pacotes: **docker-ce, docker-ce-cli ou containerd.io**. O repositório do docker atualiza mais rápido do que o do kubernetes, logo, não se deve ter a versão mais atualizada do docker pois o K8s pode não suportar.


### Creating a single control-plane cluster with kubeadm
Os objetivos desta seção serão: Instalar um cluster Kubernetes com control-plane e instalar um puglin de rede usada para que os pods possam se comunicar entre si.

**Esse processo  SÓ deve ser feito no MASTER! Apenas ele tem o Control Plane.**
Iniciar o control-plane node:
```
kubeadm init
```

Caso você use mais de uma interface de rede, é possível setar qual a interface você quer que seja usada nesta operação:
```
kubeadm init --apiserver-advertise-address $(hostname -i)
```

Note, os dois comandos são equivalentes caso você tenha apenas uma interface de rede!
A saída deste comando (kubeadmn init) deve ser similar a mostrada abaixo:

<p align="center">
<img src="/images/insk8s_1.png" alt="Saída esperada do kubeadm init" style="width:650px;height:250px;"/>
<figcaption align="center">Figura 1 - Saída esperada do kubeadm init. Fonte: Autoria própia </figcaption>
</p>

A partir de agora, você tem duas opções para fazer iniciar seu cluster:
1. Como um usuário comum.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
2. Como root.
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
O kubeadm join… mostrado acima, é o comando utilizado para ingressar seus workers no seu cluster.

Na parte de rede, existem plugins que permitem criar redes virtuais ou rede de overlay. Eles criam túneis entre os nodes, eles permitem que um pod que está no node A consiga se comunicar com um pode no node B, por exemplo.

Na documentação oficial, vimos alguns dos diferentes plugins disponíveis. Aqui foi usado o Weave, conforme está mostrado abaixo: 
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Para passar o tráfego IPv4 para as cadeias de iptables, se faz necessário:

```
sysctl net.bridge.bridge-nf-call-iptables=1
```
Depois que o pod network foi instalado, você pode confirmar se o pod CoreDNS está Running na saída do:

```
 kubectl get pods --all-namespaces
```

<p align="center">
<img src="/images/insk8s_2.png" alt="Saída esperada do kubectl get pods --all-namespaces (com alguns pods iniciando)" style="width:620px;height:210px;"/>
<figcaption align="center">Figura 2 - Saída esperada do kubectl get pods --all-namespaces (com alguns pods iniciando). Fonte: Autoria própia </figcaption>
</p>

Como mostrado na Figura 2, assim que se iniciam os pods ficam no status Pending para depois passar para Running. 

<p align="center">
<img src="/images/insk8s_3.png" alt="Saída esperada do kubectl get pods --all-namespaces (com todos os pods prontos)" style="width:650px;height:220px;"/>
<figcaption align="center">Figura 3 - Saída esperada do kubectl get pods --all-namespaces (com todos os pods prontos). Fonte: Autoria própia </figcaption>
</p>

Após todos subirem corretamente, você pode verificar que os nodes.
```
kubectl get nodes
```
<p align="center">
<img src="/images/insk8s_4.png" alt="Saída esperada do kubectl get nodes" style="width:300px;height:80px;"/>
<figcaption align="center">Figura 4.1 - Saída esperada do kubectl get nodes, você deverá ver essa! Depois que adicionar os nodes, verá a Figura 4.2. Fonte: Autoria própia </figcaption>
</p>


<p align="center">
<img src="/images/insk8s_5.png" alt="Saída esperada do kubectl get nodes" style="width:320px;height:100px;"/>
<figcaption align="center">Figura 4.2 - Saída esperada do kubectl get nodes com os workers adicionais. Fonte: Autoria própia </figcaption>
</p>


### Joining your nodes
**Esse processo deve ser feito nos workers!**
Aqui, você já deu o comando kubeadm init e já tem a saída, que inclue o kubeadm join, como no **EXEMPLO** abaixo. Se atente que você deve usar o kubeadm join que você obteve, não o do exemplo. 
```
 kubeadm join 10.128.0.3:6443 --token c3y122.omruf7ogbo4x34d7 \
    --discovery-token-ca-cert-hash sha256:4541d7661bdf610cca14500f00425de412e7773e8962c3ac0b218717bacce88c
```

Após pegar este comando, você deve inseri-lo nas instâncias que serão seus workers.

Abaixo é mostrado um exemplo dessa inserção, porém temos como retorno o seguinte erro: [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1.

<p align="center">
<img src="/images/insk8s_6.png" alt="Erro do kubeadm join" style="width:1000px;height:115px;"/>
<figcaption align="center">Figura 5 - Erro do kubeadm join. Fonte: Autoria própia </figcaption>
</p>

Após esse erro foi verificado que kubelet não estava funcionando e não foi possível reiniciá-lo.

<p align="center">
<img src="/images/insk8s_7.png" alt="Service kubelet não iniciado" style="width:680px;height:270px;"/>
<figcaption align="center">Figura 6 - Service kubelet não iniciado. Fonte: Autoria própia </figcaption>
</p>

Para solucionar foi necessário restartar o docker nos workers:
```
 systemctl restart docker 
```
Depois disso, o kubeadm join funcionou como o esperado e o kubelet subiu automaticamente.

<p align="center">
<img src="/images/insk8s_8.png" alt="Service kubelet iniciado" style="width:1000px;height:350px;"/>
<figcaption align="center">Figura 7 - Service kubelet iniciado. Fonte: Autoria própia </figcaption>
</p>

Caso você não tenho mais o token, é possível gerar outro. Todos os comandos relacionados a listagem/geração de tokens são feitas no master.
```
 kubeadm token list 
```
Os tokens expiram após 24 horas, caso você deseje gerar outro token:
```
kubeadm token create 
```
A saída será semelhante a isto:  5didvk.d09sbcov8ph2amjw
Caso você não tenha o valor do **--discovery-token-ca-cert-hash**, você pode conseguir por meio desse comando:
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```
Agora é só juntar as saídas anteriores e formar o comando completo do kubeadm join.

### Remove the node
Caso deseje remover um node, para fazer os comandos listados abaixo:
```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets

kubectl delete node <node name>

kubeadm reset
```

## Comandos interessantes
* Mostrar detalhes do Node com nome master
```
kubectl describe node master
```
* Mostrar algumas informações do cluster
```
kubectl cluster-info
```
<p align="center">
<img src="/images/insk8s_9.png" alt=" Informações do cluster" style="width:800px;height:90px;"/>
<figcaption align="center">Figura 8 -  Informações do cluster. Fonte: Autoria própia </figcaption>
</p>

* Mostrar status dos componentes
```
kubectl get componentstatuses
```
<p align="center">
<img src="/images/insk8s_10.png" alt=" Informações dos componentes" style="width:470px;height:70px;"/>
<figcaption align="center">Figura 9 -  Informações dos componentes. Fonte: Autoria própia </figcaption>
</p>



&nbsp;

#### Referências
1. NSIGHT, UFC. Introdução a Kubernetes. Disponível em:<https://docs.google.com/presentation/d/1weqpBWa9FNjKc1ugCUIpwYYquvoIOFbUvEcZ9ZYapAg/edit#slide=id.g654e6a820d_0_553>.Acessado em: 07 mar. 2020.
2. kubernets. Production environment. Disponível em:<https://kubernetes.io/docs/setup/production-environment>. Acessado em: 07 mar. 2020.
3. Google Cloud. Como usar regras de firewall. Disponível em:<https://cloud.google.com/vpc/docs/using-firewalls?hl=pt-br>. Acessado em: 07 mar. 2020.
