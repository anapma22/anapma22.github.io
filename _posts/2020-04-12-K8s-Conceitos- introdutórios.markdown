---
layout: post
title:  "Conceitos básicos do Kubernetes"
date:   2020-04-12 17:06:38 -0300
categories: jekyll update
---
## Introdução

Este documento foi desenvolvido durante a disciplina DevOps na Residência em TI aplicada à área jurídica - turma JF/IMD, na especialidade de Redes e Infraestrutura. O docente foi o Prof Me. André Solino.   

<p align="center">
<img src="/images/kubernetes_logo.png" alt="Logotipo do kubernetes" style="width:200px;height:190px;"/>
<figcaption align="center">Figura 1 - Logotipo do kubernetes. Fonte: kubernetes.io.</figcaption>
</p>

Na figura 1, podemos vê à logotipo do Kubernetes (conhecido também como K8s), que sugere um leme de um navio. Mas sabemos que um navio pode levar praticamente qualquer coisa, porém em larga escala, seja lá o que for carregado dentro de um navio, está armazenado dentro de um container. 

Uma informação importante é que a palavra Kubernetes vem da palavra grega Kuvernetes, que representa a pessoa que pilota o navio. 

Com a inferência correta, já sabemos que o Kubernetes é responsável por fazer a orquestração e gerenciamento de containers! Ele é um sistema de orquestração open-source que automatiza a implantação, o dimensionamento e a gestão de aplicações em containers.

O Kubernetes faz o gerenciamento do cluster, que é composto por vários containers. O Docker é a tecnologia mais utilizada para  provisionar os containers O próprio Docker tem seu orquestrador, o Docker Swarm, porém ele não é tão maduro quanto o Kubernetes.

## Breve história
O K8s (trocadilho com o nome Kubernetes: k + 8 caracteres + s) é inscrito em GO, foi desenvolvido pela Google, para gerência dos serviços da G Suite. Em 2015 o Kubernetes foi doado para a “Cloud Native Computing foundation” e Linux Foundations, e se tornou um projeto Open Source, hoje é mantido pela comunidade. Isso é mais um incentivo a sua popularidade e grande utilização, visto que por já ser da comunidade, não fica restrito a nenhuma alteração drástica partindo da empresa que detivesse essa ferramenta.

## Conceitos importantes
Quando se implanta o Kubernetes, que é o **deploy** (esse conceito será detalhado abaixo), você passa a ter um **cluster**. Que é composto pelo  **master** e os **workers**, também conhecido como **nodes** ou **minions**. 

O master é responsável por controlar os nodes do Kubernetes, que são quem fazem as tarefas dentro dos containers. Cada cluster tem pelo menos um node ou worker. 
Dentro dos workers ficam os **pods**. Em um node é possível ter nenhum, um ou vários pods, a depender da sua aplicação.
 * Master: Executa o Control Plane.

 * Node: Executa os containers que estão dentro dos pods. É a menor unidade de hardware de computação no Kubernetes. 

 Também existem outros componentes que estão dentro do cluster, são eles:
* kubeadm: Automatiza parte do processo de criação do cluster.

* kubectl: Interface de linha de comando do K8s, ele opera o cluster gerenciando suas interações. Ainda temos o kubelet, mas ele será detalhado nos componentes do node.

Os pods são unidade do Kubernetes, dentro dele pode ter um ou mais containers e uma unidade de armazenamento. É característica dos pods serem voláteis e cada um tem um IP e id únicos. Eles podem ter diferentes estados:
* Pending: Estado inicial do pod.

* Running: Em execução.

* Success: Pod concluiu a tarefa e morreu.

* Unknown: Estado não reconhecido, o linux retornou qualquer coisa diferente de 0.

Dentro dos pods, temos os **services**, que são uma maneira abstrata de expor uma aplicação em execução em um conjunto de pods como um serviço de rede. 

O acesso aos clusters, é feito por meio do **ingress**. É uma coleção de regras de roteamento, responsáveis por estabelecer como usuários externos acessam serviços que estão em execução em um cluster Kubernetes.  

A rede no Kubernetes é configurada por meio de plugins que permitem criar redes virtuais ou rede de overlay. Eles criam túneis criptografados entre os nodes, que permitem que toda comunicação ocorra. Ou seja, um pod que está no node A consegue se comunicar com um pod no node B. 

Existem vários plugins, como Flannel ou Weave, porém a lista de opções é bem maior e pode ser conferida [aqui](https://kubernetes.io/docs/concepts/cluster-administration/networking/). 

Visando organizar objetos dentro do cluster, temos o conceito de namespace, que pode ser entendido como uma divisão lógica. Por padrão, kubectl interage com o namespace padrão (default).

Agora, que já temos alguns conceitos esclarecidos, vamos saber mais sobre a estrutura do Kubernetes.

### Control Plane e Nodes
O K8s possui um  software conhecido como **control plane**, nele é decido quando e onde executar os pods, além de gerenciar o roteamento do tráfego e escalar os pods de acordo com a utilização ou outras métricas definidas. 

<p align="center">
<img src="/images/Kubernetes_controlplane.png" alt=" Control Plane" style="width:750px;height:400px;"/>
<figcaption align="center">Figura 2 - Estrututura do kubernetes. Fonte: kubernetes.io.</figcaption>
</p>

Os componentes do Control Plane tomam decisões gerais sobre o cluster, além de detectar e responder a eventos do cluster, como iniciar um novo pod quando uma réplica de um **deployment** não inicia corretamente, por exemplo.  
Um deployment é uma implantação de uma aplicação dentro do Kubernetes, por exemplo um deploy do ngnix, vai subir um ngnix. Ao configurar um deploy, é possível setar quantos pods se deseja iniciar para determinada aplicação, a essa abstração damos o nome de réplicas.

Como é visto na figura 2, temos alguns componentes do Control Plane, que são:
* etcd: É uma base de dados de chave valor, que armazena os dados de configuração do cluster e o estado do cluster.

* kube-apiserver: Fornece API kubernetes usando Jason, baseado em REST. Os estados de objetos da API são armazenados no etcd, e o kubectl usa o kube-api-server para se comunicar com o cluster.

* kube-controller-manager: Controles responsáveis por monitorar continuamente o estado do cluster por meio da apiserver. Além de fazer mudanças de estado no cluster, alguns exemplos são o replication controller, endpoints controller, namespace controller, and serviceaccounts controller.

* cloud-controller-manager: Um daemon que incorpora loops de controle específicos da nuvem. Os provedores de nuvem se desenvolvem em um ritmo diferente do Kubernetes, sendo necessário o binário cloud-controller-manager para permitir que esses provedores abstraiam o código que seria para o kube-controller-manager.

* kube-scheduler: Responsável por executar as tarefas de agendamento, como execução de containers nos nodes com base na disponibilidade de recursos. Quando um pod está no estado pending, quem decide para onde o pod irá será este componente. É possível interferir nesta política, por exemplo indicando a máquina para onde os pods devem ir preferencialmente.

Agora, veremos os componentes dos nodes:
* kube-proxy: Quando se cria pods, ele pode ir pra qualquer máquina e um pod precisa se comunicar com o outro, ou seja, é necessário a comunicação entre os nodes. O kube-proxy é responsável por lidar com as regras de firewall.

* kubelet: Faz a interface com o docker, atua como agente em cada node. Lida com a execução dos pods. Sempre conversa com o kube-apiserver.


Para que o post não fique muito longo, não foram apresentados mais conceitos. Eles são introdutórios, porém importantes para entender todo o resto do Kubernetes!

&nbsp;

#### Referências
1. Tudo o que você precisa saber sobre Kubernetes – Parte 0. iMasters.  Disponível em: <https://imasters.com.br/desenvolvimento/tudo-o-que-voce-precisa-saber-sobre-ku,bernetes-parte-01>. Acessado em 23 de fev. 2020.
2. What is Kubernetes. Kubernetes. Disponível em: <https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/>. Acessado em 23 de fev. 2020.
3. Kubernetes Components. Kubernetes. Disponível em: <https://kubernetes.io/docs/concepts/overview/components/>. Acessado em 23 de fev. 2020.
4.  Alta disponibilidade com Kubernetes na prática. Tech@Grupo ZAP. Disponível em: <https://medium.com/tech-grupozap/alta-disponibilidade-com-kubernetes-na-pr%C3%A1tica-b9cf4261d2f7>. Acessado em 23 de fev. 2020.
5. Kubernetes na AWS. AWS. Disponível em: <https://aws.amazon.com/pt/kubernetes/>. Acessado em 23 de fev. 2020.
