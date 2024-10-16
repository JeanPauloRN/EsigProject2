# EsigProject2

### Implementação de Jenkins sendo mantido por Tomcat em Kubernetes com ambiente monitorado.

Aplicações Usadas/Versão:

- K3s
- Tomcat 9
- Jenkins Latest
- Helm 
- Kube Prometheus Stack

Para orquestração dos contêineres, utilizaremos o Kubernetes na distribuição K3s. Criaremos um cluster com apenas o master para realizar nosso laboratório dentro de uma máquina Debian 12.

Nesse cluster será feito deploy do nosso Tomcat com o Jenkins embutido. Usaremos o Helm para instalação do Kube Prometheus Stack que por sua vez já virá com as configurações padrão para monitoração com Prometheus e Grafana do nosso Cluster e consequentemente dos nossos Pods. 

---

### Instalação do K3s

Realizando atualização do sistema e instalando pacotes obrigatórios:

```bash
sudo apt update && sudo apt install ca-certificates curl -y
```

Após baixar a instalar os pacotes mais recentes no sistema, vamos fazer a instalação do K3s e iniciar o cluster com 1 node master com o seguinte comando:

```bash
curl -sfL https://get.k3s.io |  sh -s - server --cluster-init
```

Após finalizado, vamos verificar se nosso K3s está rodando corretamente:

```bash
k3s kubectl get nodes
```

---

### Instalação do Tomcat

Como indicado, vamos partir para criação dos nossos arquivos de deploy para nosso Tomcat, baixando o ``jenkins.war`` para dentro da pasta webapps do ``Tomcat`` de forma automatizada no deployment.

Para garantir o funcionamento do nosso ``Tomcat`` e ``Jenkins`` em modo persistente, precisamos que seja criado um pvc para armazenar as pastas ``/usr/local/tomcat/webapps`` do nosso ``Tomcat`` e a `` /root/.jenkins/`` do ``Jenkins``. A configuração anteriormente citada permite que após alterações e reinicialização do nosso container os dados sejam mantidos.

Iremos iniciar a criação do PVC criando um arquivo para o tomcat com nome de ``tomcat-pvc.yaml`` com o seguinte conteúdo:

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tomcat-pvc
spec:
  accessModes:
    - ReadWriteOnce 
  resources:
    requests:
      storage: 1Gi #Nesse caso Definimos 1Gb para armazenamento
```

Agora, vamos criar um outro PVC para manter os dados do nosso ``Jenkins``, ``jenkins-pvc.yaml``:

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi #Nesse caso Definimos 1Gb para armazenamento
```
Com os 2 arquivos para persistencia de dados da aplicação, agora podemos criar o arquivo de deploy do nosso ``Tomcat``, ``tomcat-deployment.yaml``:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat
spec:
  replicas: 1 #Especificamos 1 Réplica
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      initContainers: #Container de inicilização para download do arquivo do Jenkins
      - name: download-jenkins
        image: alpine:latest
        command: ['sh', '-c', 'wget -O /tmp/jenkins.war https://get.jenkins.io/war-stable/2.462.3/jenkins.war'] #Comando para download do Jenkins
        volumeMounts:
        - name: tomcat-volume 
          mountPath: /tmp
      containers:
      - name: tomcat
        image: tomcat:9.0 #Especificação de imagem do tomcat 9
        ports:
        - containerPort: 8080 #Porta do Container
        resources: #Definição de Recusos
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1"
        volumeMounts:
        - name: tomcat-volume
          mountPath: /usr/local/tomcat/webapps/ #Apontamento do Local para persistencia dos dados do tomcat
        - name: jenkins-volume
          mountPath: /root/.jenkins/  #Apontamento do Local para persistencia dos dados do Jenkins
      volumes:
      - name: tomcat-volume
        persistentVolumeClaim:
          claimName: tomcat-pvc #Apontamento do arquivo PVC criado anteriormente
      - name: jenkins-volume
        persistentVolumeClaim:
          claimName: jenkins-pvc #Apontamento do arquivo PVC criado anteriormente
```          
Com nosso arquivo de deployment criado, vamos agora especificar um arquivo de service para prover acesso as portas do ``Tomcat`` por meio de um ingress que será criado futuramente, ``tomcat-service.yaml``:

```bash
apiVersion: v1
kind: Service
metadata:
  name: tomcat
spec:
  type: ClusterIP
  ports:
  - port: 8080        # Porta interna do Tomcat
    targetPort: 8080  # Porta externa
  selector:
    app: tomcat  # Seleciona o pod do Tomcat com base
```

Com o arquivo de service devidamente criado, agora podemos avançar para o ingress, ``tomcat-jeankins-ingress.yaml``:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tomcat-jenkins-ingress
spec:
  ingressClassName: traefik
spec:
  rules:
  - host: jenkins.work
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat
            port:
              number: 8080
```

Após a criação, vamos subir cada arquivo do deploy para garantir que o funcionamento ocorra sem falhas na seguinte ordem e utilizando os comandos:

```bash
kubectl apply -f tomcat-pvc.yaml
```

```bash
kubectl apply -f jenkins-pvc.yaml
```

```bash
kubectl apply -f tomcat-service.yaml
```

```bash
kubectl apply -f tomcat-jenkins-ingress.yaml
```

```bash
kubectl apply -f tomcat-deployment.yaml
```

Podemos verificar o funcionamento de cada um com os respectivos comandos:

```bash
kubectl get pvc
```

```bash
kubectl services pvc
```

```bash
kubectl get ingress
```

```bash
kubectl get pods
```

Com o uso desses recursos o nosso container do tomcat com já estará acessível pela url ``https://jenkins.work``.

`Observação 1`: É necessário que a máquina usada para especifique em seu arquivo hosts que o nome ``jenkins.work`` seja resolvido para o IP do cluster, em nosso laboratório utilizamos o endereço ``192.168.0.19``.

`Observação 2`: Embora a porta do container do ``Tomcat`` seja a 8080, nesse caso vamos utilizar ingress e fazemos o acesso na porta 443.

Ao acessar a url será exibida a tela de primeiro acesso do ``Jenkins``, nesse momento existe a necessidade de acessar nosso container do tomcat para verificar qual é essa senha no arquivo ``initialAdminPassword``.

Antes de acessar o container é necessário verificar qual seu nome para realizar o apontamento no comando seguinte. Para visualizar o nome dos containers em execução, utilizamos o comando:

```bash
kubectl get pods
```

Após identificar o nome do container faremos a substituição no comando abaixo:

```bash
kubectl exec -it <Nome_do_Container> /bin/bash
```

Dentro do nosso ``Tomcat`` vamos visualizar o arquivo com a senha:

```bash
cat /root/.jenkins/secrets/initialAdminPassword
```
Após visualizar a senha é possível acessar o ``Jenkins`` e fazer as configurações iniciais. No nosso caso a senha foi trocada para ``J3nk1nS@eSiG2024``.

Com as configurações acima realizadas, será possível reiniciar nossos containers e as configurações feitas no Jenkins são mantidas.

---

### Instalação do Helm, Kube Prometheus Stack (Prometheus, Grafana e AlertManager) 

O ``Helm`` será para instalação de um pacote de aplicação chamada de Kube Prometheus Stack. Essa aplicação possui um conjunto de ferramentas que facilitará o monitoramento e observabilidade do nosso cluster e seus pods.

Para implementação o primeiro passo é a aquisição do ``Helm``:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Após executar o comando acima o Helm deverá ser instalado e poderemos verificar sua versão com o outro comando abaixo:

```bash
helm version
```
Agora, vamos instalar e atualizar os repositórios do ``Helm``:

```bash
helm repo add stable https://charts.helm.sh/stable
```

```bash
helm repo update
```

Prosseguindo, vamos adicicionar o repositório do ``Kube Prometheus Stack`` e atualizá-lo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
helm repo update
```

Por padrão o projeto do ``Kube Prometheus Stack`` utiliza um namespace diferente do Default para realizar a instalação das suas ferramentas, então, será necessário realizar a criação do namespace de nome ``monitoring``:

```bash
kubectl create namespace monitoring
```

Agora, estamos prontos para realizar a instalação:

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
```

Note que, a instalação das ferramentas, services, pvcs e pods pode demorar. É recomendável acompanhar essa instalação com o comando abaixo, quando os pods estiverem em status de running podemos prosseguir com a configuração:

```bash
kubectl get pods -n monitoring
```

Com os recursos devidamente instalados, agora podemos fazer as personalizações para que nosso grafana fique acessível externamente.

Para acessar a ferramenta externamente por nome, utilizaremos o ingress abaixo:

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring #Especifica nosso namespace criado para o Kube Prometheus
spec:
  ingressClassName: traefik
  rules:
    - host: grafana.work #Especifica o nome externo para acesso
      http:
        paths:
          - backend:
              service:
                name: kube-prometheus-grafana
                port:
                  number: 80 #Porta interna do Grafana
            path: /
            pathType: Prefix
```

Aplicando:

```bash
kubectl apply -f grafana-ingress.yaml -n monitoring
```

Após o comando de Apply o grafana estará acessível pelo endereço ``https://grafana.work``.

`Observação 1`:  É necessário que na máquina hospedeira seja especificado em seu arquivo hosts que o nome seja ``grafana.work``, para ser resolvido ao IP do cluster, em nosso laboratório utilizamos o endereço ``192.168.0.19``.

`Observação 2`: Embora a porta do container do ``Grafana`` seja a 80, nesse caso vamos utilizar ingress e fazemos o acesso na porta 443.

Após o passo a passo acima, podemos acessar o ``Grafana`` com o usuário ``admin`` e senha ``prom-operator``.

``OBS:`` Por padrão a instalação do ``Kube Prometheus Stack`` já cria os volumes persistentes das aplicações inclusas nele. Sendo assim, qualquer Dashboard adicionado ao ``Grafana`` ou alterações de usuário serão mantidos em possíveis reinicializações.

Para finalizar, não fiz uso dos dashboards padrão do ``Grafana`` contidos na instalação.

Fiz o uso dashboards da comunidade que podem ser encontrados nos links abaixo e importados via ID:

[Cluster Metrics](https://grafana.com/grafana/dashboards/15757-kubernetes-views-global/)

[Pods Metrics](https://grafana.com/grafana/dashboards/21548-k8s-pod-metrics/)

[Alerts](https://grafana.com/grafana/dashboards/20261-node-and-container-alerts/)


End.




