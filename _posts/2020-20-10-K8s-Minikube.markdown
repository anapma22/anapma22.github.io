---
layout: post
title:  "Tutorial de como instalar o minikube em uma instância do google cloud"
date:   2020-20-10 17:06:38 -0300
categories: jekyll update
---

Este documento foi desenvolvido em **março de 2020**, durante a disciplina DevOps na Residência em tecnologia da informação aplicada à área jurídica - turma JF/IMD, na área de especialidade de Redes e Infraestrutura, ministrado pelo Prof Me. André Solino.   

O objetivo deste tutorial é a instalação do minikube numa instância da Google Cloud Platform. 
Este tutorial abordará desde o início a criação do projeto na Google Cloud Platform até a finalização da instalação do minikube. Os tópicos detalhados estão listados abaixo:
1. Definição teórica
2. Criação do projeto
3. Criação da instância de VM
4. Instalação do kubectl
5. Instalação do minikube 
6. Deploy
7. Fontes

## Definição teórica
Mas afinal, o que é minikube?

Primeiro, vamos falar do Kubernets, ele foi criado e desenvolvido pelos engenheiros da Google, uma das pioneiras no desenvolvimento da tecnologia de containers. Em 2015 o Kubernetes foi doado para a “Cloud Native Computing foundation” e Linux Foundations, e se tornou um projeto Open Source.

Sua estrutura conta com o Master e os Workes. Onde o primeiro é responsável por controlar os nós do Kubernetes e os Workers são hosts do cluster (nós ou nodes), são as máquinas que realizam as tarefas solicitadas. Essa organização é bem parecida com o Docker Swarm. 

Apesar dessa semelhança o setup do Kubernets é mais complexo do que o do Docker Swarm. Mesmo para um ambiente de testes, o mínimo exigido eram cinco máquinas para simular um cluster com três Masters e dois Workes/Nós. Para melhorar isso, a comunidade criou o minikube, que permite subir rápido um ambiente, no qual é possível conhecer e ver o Kubernetes funcionando.

O minikube simula um cluster real de um único nó, ou seja todos os processos e serviços que deveriam rodar no master e no slave rodam no node do minkube. 

## Criação do projeto
1. Acessar https://console.cloud.google.com/
2. Clique em “Selecione um projeto” no canto superior esquerdo.
3. Clique em “Novo projeto”. Digite o nome do projeto e finalize esta etapa.

##  Criação da Instância de VM
4. No terminal digite $ssh-keygen no computador que será usado para acessar a instância de VM.
5. Clique em “Compute Engine’’, na seção “Computação’’. Essa opção é encontrada no menu de navegação, localizado na lateral esquerda. 

Caso não veja esse menu ele está oculto, clique nas três barras no canto superior esquerdo que o menu será mostrado. 

<p align="center">
<img src="/images/google_platform.png" alt="Barras que ocultam / mostram o menu de navegação" style="width:200px;height:190px;"/>
<figcaption align="center">Figura 1 - Google Cloud Platform. Fonte: console.cloud.google.com </figcaption>
</p>

6. A mensagem que o Google Compute Engine está sendo preparado será exibida, poderá levar alguns minutos e após isso será possível criar a instância.
7. Clique em “Criar’’, esta opção abre uma série de opções que podem ser personalizadas de acordo com o objetivo. Abaixo será mostrado apenas o que precisa ser alterado, as demais opções continuam com o valor padrão.
    * Nome: minikube.
    * Tipo de máquina: n1-standard-2
    * Disco de inicialização: CentOS 7. 
    * Clicar em: “Gerenciamento, segurança, discos, rede, locatário único“. 
        * Clicar na aba “Segurança’’.
        * Inserir a chave SSH pública gerada no ponto 4. Sempre é possível editar essas chaves, inserir novas ou remover alguma já configurada.
    * Clicar em “Criar’’ para finalizar esta etapa.

## Instalação do kubectl
8. Ligar a instância e acessá-la.
    * Selecione a instância que foi criada, em seguida clique em “Iniciar’’ na parte superior.
    * No terminal digite $ssh user@IPexterno, esse IP é mostrado depois que a instância foi iniciada.
9. Instalar kubectl.
    * Download do último release: 
    ```
    curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
    ```
    * Conceda a permissão de execução para o binário do kubectl
    ```
    chmod +x ./kubectl
    ```
    * Mova o binário para o PATH indicado:
    ```
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```
    * Teste para verificar se a instalação ocorreu corretamente e na versão indicada:
    ```
    kubectl version --client
    ```
    * A saída esperada está mostrada abaixo:
    <p align="center">
    <img src="/images/minikube_1.png" alt="Saída do comando para verificação do kubectl" style="width:200px;height:190px;"/>
    <figcaption align="center">Figura 2 - Saída do comando para verificação do kubectl. Fonte: Autoria própria </figcaption>
    </p>


## Instalação do minikube
A documentação oficial foi usada como base, o link está no final deste documento.
10. Download do binário:
    ```
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
    ```
11. Adicionar o binário para o PATH indicado:
    ```
    sudo mkdir -p /usr/local/bin/
    sudo install minikube /usr/local/bin/
    ```
12. O minikube precisa do docker:
    ```
    sudo yum install docker
    ```
13. O --vm-driver=none precisa da permissão de root:
    ```
    sudo su -
    ```
Como estamos trabalhando em um ambiente de testes, iremos parar o firewalld e desabilitar o SELinux. Caso você esteja em um ambiente de produção é importante rever a documentação desses serviços para que eles fiquem habilitados e não abram falhas de segurança na sua implementação.

14. Parando o firewalld:
    ```
    systemctl stop firewalld
    systemctl disable firewalld
    ```
14. Parando o SELinux:  em seguida alterar a linha “SELINUX=enforcing’ confirme está mostrado abaixo
    ```
    vim /etc/sysconfig/selinux
    ```                    
    * Altere a linha "SELINUX=enforcing" para:
    ```
    SELINUX=disabled
    ```  
16. Agora é necessário reiniciar a máquina para que a alteração do SELinux seja feita.
17. Caso você deseje conferir, após reiniciar a máquina digite $sestatus, você verá que o SELinux foi desabilitado.
18. Verifique a instalação iniciando o minikube.    
Execute o seguinte comando no seu terminal para concluir a criação do cluster, um cluster de nó único será criado. Observe que esse comando pode demorar um pouco:
    ```
    minikube start --vm-driver=none
    ```  
19. Após a finalização do minikube start, é hora de checar se o minikube está rodando conforme o esperado:
    ```
    minikube status
    ```      
    * A saída esperada está mostrada abaixo:
    <p align="center">
    <img src="/images/minikube_2.png" alt="Saída esperada do comando para verificação do minikube" style="width:200px;height:190px;"/>
    <figcaption align="center">Figura 3 - Saída esperada do comando para verificação do minikube. Fonte: Autoria própria </figcaption>
    </p>
20. Como já foi dito o minikube simula um cluster real de um único nó, é possível vê esse comportamento por meio do comando $kubectl get nodes. A saída deve ser conforme está mostrado abaixo.
    <p align="center">
    <img src="/images/minikube_3.png" alt="Saída esperada do comando para verificação de nodes do kubernetes" style="width:200px;height:190px;"/>
    <figcaption align="center">Figura 4 - Saída esperada do comando para verificação de nodes do kubernetes. Fonte: Autoria própria </figcaption>
    </p>

## Deployment 
21. Agora, vamos subir um deployment do ngnix para rodar em nosso minikube.
* Modo imperativo
```
kubectl run ngnix --generator=run-pod/v1 --image ngnix
```
* Modo declarativo
```
apiVersion: v1
kind: Pod
metadata:
  name: ngnix-pod
spec:
  containers:
  - name: ngnix-container
    image: ngnix
```
22. Vamos verificar o deployment, por meio do comando $kubectl get deployment. A saída deve ser similar a mostrada abaixo:
    <p align="center">
    <img src="/images/minikube_4.png" alt="Saída esperada da verificação do deployment" style="width:200px;height:190px;"/>
    <figcaption align="center">Figura 5 - Saída esperada da verificação do deployment. Fonte: Autoria própria </figcaption>
    </p>
23. Verificar os pods: 
    ``` 
    kubectl get pods
    ```
    * A saída esperada está mostrada abaixo:
    <p align="center">
    <img src="/images/minikube_5.png" alt="Saída esperada da verificação dos pods" style="width:200px;height:190px;"/>
    <figcaption align="center">Figura6 - Saída esperada da verificação dos pods. Fonte: Autoria própria </figcaption>
    </p>    

&nbsp;

#### Referências
1. Tudo o que você precisa saber sobre Kubernetes – Parte 0. iMasters. Acessado em 23 de fev. 2020. Disponível em: <https://imasters.com.br/desenvolvimento/tudo-o-que-voce-precisa-saber-sobre-kubernetes-parte-01>. 
2. Install Minikube. Kubernetes. Acessado em 23 de fev. 2020. Disponível em: <https://kubernetes.io/docs/tasks/tools/install-minikube/>
3. #01 - O que é o KUBERNETES e INSTALANDO o MINIKUBE | Descomplicando o Kubernetes. Acessado em 23 de fev. 2020. Disponível em: <https://www.youtube.com/watch?v=pV0nkr61XP8>
