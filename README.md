# Startando e stoppando o minikube

O minikube é uma "VMzinha" que simula um cluster Kubernetes, sem inúmeros worker nodes para economizar recursos.

- **minikube start** – Starta a VM do minikube. Com a flag **–driver=[DRIVER_NAME]** podemos especificar o driver (docker, virtualbox, etc)
- **minikube stop** - Stoppa a VM do minikube.
- **minikube status** – Verifica o status da VM criada. Útil para verificar se está rodando.
- **minikube delete** – Deleta a VM criada.
- **minikube dashboard** – Abre uma aba no browser com um dashboard do Kubernetes.

# Listando recursos com o kubectl get

```
kubectl get <RECURSO>
```

# Criando Deployments com o kubectl create no modo imperativo

```
kubectl create deployment '<DEPLOYMENT_NAME>' --image='<IMAGE>'
```

# Deletando Deployments com o kubectl delete

```
kubectl delete deployment '<DEPLOYMENT_NAME>'
```

Ou, caso o deployment tenha sido criado na forma declarativa (arquivo), também é possível deletar apontando para o arquivo

```
kubectl delete -f=deployment.yaml -f=service.yaml -f=...
```

# Criando um Service para expor uma aplicação com o kubectl expose

```
kubectl expose deployment '<DEPLOYMENT_NAME>' --port=<PORT> --type=<TYPE>
```

Para **\<TYPE>**, temos:

  1. ClusterIP (só será ‘reachable’ de dentro do cluster)
  2. NodePort (‘reachable’ de fora do cluster)
  3. LoadBalancer (Gera um IP único para todas as pods daquele deployment e expõe ele, além de distribuir o tráfego)

Após executar o comando, o comando **kubectl get services** demonstrará a columa **EXTERNAL IP** como \<pending>.

![image](https://user-images.githubusercontent.com/80921933/210461779-380ec446-57a1-4839-aabe-1d8099752feb.png)


Isso acontece porque um cloud-provider já forneceria tal IP, mas rodando o Kubernetes no minikube, ainda precisamos rodar o comando

```
minikube service <SERVICE>
```

# Escalando Pods com o kubectl scale

```
kubectl scale deployment/<DEPLOYMENT> --replicas=<DERIRED_NUMBER_OF_PODS>
```

Antes do escalonamento:

![image](https://user-images.githubusercontent.com/80921933/210464647-32c9dc80-c107-4276-816c-79230ac0d41c.png)

Comando rodado:

```
kubectl scale deployment/spring-app --replicas=5
```

Depois do escalonamento:

![image](https://user-images.githubusercontent.com/80921933/210464692-93118384-fb87-46a2-b5a2-13b8bc3cbb7c.png)

# Atualizando a docker-image de um deployment

```
kubectl set image deployment/<DEPLOYMENT> <CONTAINER>=<NEW_IMAGE>
```

**\<CONTAINER>** é a **base da imagem** utilizada, sem a tag e **sem o nome do usuário de repositório**, ex: Na imagem **azold6/imagemteste:6**, a base da imagem seria **imagemteste**

Para verificar o progresso das mudanças, executar 

```
kubectl rollout status deployment/<DEPLOYMENT>
```

Caso a seguinte saída aconteça, os Pods foram atualizados

![image](https://user-images.githubusercontent.com/80921933/210465852-881647b1-1dc7-4bea-bcad-29add2ed49ab.png)

# Realizando o rollout de uma alteração em um Deployment

Com o seguinte comando, podemos verificar o histórico de alterações em um deployment

```
kubectl rollout history deployment/<DEPLOYMENT>
```

Teremos uma saída nos informando sobre todas as alterações

![image](https://user-images.githubusercontent.com/80921933/210469570-512f78e3-12fb-4598-a775-934f0df94bde.png)

Podemos detalhar uma revision adicionando a flag **--revision=<REVISION_NUMBER>**

```
kubectl rollout history deployment/<DEPLOYMENT> --revision=2
```

Após escolher uma revision de destino para realizar o rollback, executamos

```
kubectl rollout undo deployment/<DEPLOYMENT> --to-revision=<REVISION_NUMBER>
```

Pronto! Para verificar o progresso das mudanças, executar 

```
kubectl rollout status deployment/<DEPLOYMENT>
```

# Exemplo 1 e explicação de um Deployment declarativo básico

O primeiro **spec** fornece informações ao **Deployment**, como quantidade de Pods (replicas), qual label uma Pod deve ter para ser gerenciada por aquele Deployment, e o template da(s) Pod(s) que serão criadas com aquele deployment.

O **template** faz referência ao "perfil" da Pod. A aba **metadata** dentro do template define as labels da Pod, e o segundo **spec** define as especificações da Pod, como imagem utilizada e nome do contêiner.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec: 
  replicas: 1
  selector: 
    matchLabels:                                   
      app: spring-app ############################## 
  template:                                       ## Essas labels devem bater.
    metadata:                                     ## Está informando para o deployment que ele deve gerenciar
      labels:                                     ## Pods com a label "app: spring-app"
        app: spring-app ############################
    spec:
      containers:
        - name: spring-application
          image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
```

# Exemplo 1 e explicação de um Service declarativo básico

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-spring-app 
spec:
  selector:
    app: spring-app ######### Selector deve bater com o label da(s) Pod(s) para expor a porta 
  ports:
    - port: 80
      targetPort: 8081
      protocol: 'TCP'
  type: LoadBalancer
```

# Exemplo 2 e a utilização do Liveness Probe

O Liveness Probe existe para termos mais controle sobre os healthchecks do Kubernetes. Útil caso tenhamos que apontar para um endpoint específico da aplicação.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: with-livenessprobe
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app
          image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            periodSeconds: 3
            initialDelaySeconds: 3
```

# Explicação sobre os tipos de volumes e persistent volumes

**emptyDir** - Fica vinculado a uma Pod, ou seja, não morre enquanto a Pod existir, mas se a Pod for substituida, o volume "reinicia" (fica fazio novamente). Usado somente quando precisamos de dados temporários <br>
**hostPath** - Funciona exatamente como um bind-mount. Escolhe um caminho na máquina host para "bindar" a um caminho dentro de um worker-node (Confirmar isso!).
**Persistent Volumes** - São volumes independentes de Pods e worker-nodes. Eles usam outros volumes para funcionar. Para que uma Pod possa utilizá-lo, ela deve usar um **Persistent Volume Claim**

# Exemplo 3 e a utilização do volume type emptyDir

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    containers:
      - name: spring-app
        image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
        volumeMounts:
          - mountPath: /path/in/container/here
            name: my-emptydir ##
    volumes:                   # Match
      - name: my-emptydir ######
        emptyDir: {}
```

# Exemplo 4 e a utilização do volume type hostPath

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-hostPath
  template:
    metadata:
      labels:
        app: app-hostPath
    containers:
      - name: spring-app
        image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
        volumeMounts:
          - mountPath: /path/in/container/here ###
            name: my-hostpath ##                 # 
    volumes:                   # Match           # Esse volume-type é como um bind-mount
      - name: my-hostpath ######                 # "Binda" um path na Pod com um path no host
        hostPath:                                #
          path: /path/in/host/machine/here #######
          type: DirectoryOrCreate # Se o path 1 linha acima não existir, será criado.
```

# Exemplo 5 e a criação de um Persistent Volume

```yaml
apiVersion: apps/v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem # Block ou Filesystem. Consultar documentação.
  storageClassName: standard
  accessModes:
    - ReadWriteOnce # Quantos WN podem montar o volume. Consultar documentação.
  hostPath:
    path: /path/in/host/machine/here
    type: DirectoryOrCreate # Se o path 1 linha acima não existir, será criado.
```
# Exemplo 6 e a criação de um Persistent Volume Claim

Para esse exemplo, foi usado o o PV do exemplo 5 (acima)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  volumeName: my-pv ##### Nome do PV
  accessModes:
    - ReadWriteOnce 
  storageClassName: standard
  resources: ############ Especificação de recursos "claimados"
    requests:
      storage: 1Gi 
```

# Exemplo 7 e a montagem de um PVC em um Deployment

Para esse exemplo, foi usado o PVC do exemplo 6 (acima)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: application
          image: azold6/jenkins-with-spring:jenkins-spring-pipeline-53
          volumeMounts:
            - mountPath: /path/in/container/here
              name: my-volume ###
      volumes:                  # Match
        - name: my-volume #######
          persistentVolumeClaim:
            claimName: my-pvc ### Nome do PVC

```

# Exemplo 8 e utilização de environment variables no Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    containers:
      - name: spring-application
        image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
        env: # Definimos as env aqui...
          - name: MY_ENV_1
            value: 'hello'
          - name: MY_ENV_2
            value: 'world'
```

# Exemplo 9 e a criação de ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  MY_ENV_1: 'hello'
  MY_ENV_2: 'world'
  # key: value...
```

# Exemplo 10 e a utilização de ConfigMaps em Deployments

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-application
          image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
          env:
            - NAME: ENV_HELLO
              valueFrom:
                configMapKeyRef:
                  name: my-configmap # Name of my ConfigMap
                  key: hello # Key name inside my ConfigMap
```

# Comunicação entre dois contêineres na mesma Pod

Exemplo de dois contêineres na mesma Pod:

```yaml
  # ...
  spec:
    containers:
      - name:  cont-users
        image:  azold6/kub-demo-users:latest
      - name: cont-auth
        image: azold6/kub-demo-auth:latest
```

Quando dois contêineres sobem na mesma Pod e precisamos que eles se comuniquem, podemos usar **localhost** dentro do código para que eles consigam se comunicar.

# Comunicação entre dois contêineres em Pods diferentes

**Método 1**

Quando dois contêineres sobem em Pods diferentes (aka Deployments diferentes), podemos usar variáveis de ambiente geradas pelo Kubernetes no seguinte padrão:

```
<NOME_SERVICE>_SERVICE_HOST
```

Para dúvidas, consultar a aula 232 (Maximilian Schwarzmüller)

**Método 2**

O Kubernetes usa um serviço chamado CoreDNS, que permite que usemos o nome de um **Service** para referenciar outra aplicação (de forma similar ao docker-compose, quando dois contêineres estão na mesma network, e podemos referênciá-los por seus nomes. No Kubernetes, os nomes são substituidos pelo nome do **Service**)

Para referenciarmos outra aplicação, utilizamos a seguinte sintaxe

```
<NOME_SERVIÇO>.<NAMESPACE>
```

Por padrão o \<NAMESPACE> terá o valor de **default**

Então, caso a aplicação 1 precise mandar um request HTTP para a aplicação cujo Service chama-se **app2** e roda na porta 9090, inseriríamos, no código da aplicação 1, o seguinte domínio:

```
app2.default:9090
```

Geralmente, o domínio é fornecido através de variáveis de ambiente, configurados no próprio Deployment.

# Informações namespaces

Os namespaces "isolam" recursos do cluster virtualmente, para permitir que muitos times trabalhem de forma paralela. Também podem ser usados para limitar recursos para certos namespaces, etc...

# Criando namespaces

Para criar um novo namespace, podemos usar um arquivo yaml no seguinte formato:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here>
```

alternativamente, podemos usar o comando 

```
kubectl create namespace <insert-namespace-name-here>
```

# Listando namespaces disponíveis

```
kubectl get namespaces
```

# Inserindo recursos em namespaces

Caso tenhamos um recurso em um arquivo yaml, e queiramos inserí-lo em um namespace, usamos o seguinte comando

```
kubectl apply -f=<ARQUIVO> -n=<NAMESPACE>
```

Também é possível especificar o namespace no próprio arquivo, da seguinte forma:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  namespace: alex # Namespace deve ser especificado no campo metadata
spec:
# ...               Pods já serão criadas no namespace especificado acima
```

# Buscando recursos contidos em alguns ou em todos os namespaces

Para buscar Pods, Deployments, ou qualquer outro recurso em um namespace, podemos executar o comando

```
kubectl get pods -n=<NAMESPACE>
```

Para buscar recursos de todos os namespaces, podemos executar

```
kubectl get pods --all-namespaces
```

# Habilianto o auto completion para comandos do Kubernetes

Trocar para o root user

```
sudo su -
```

Executar

```
kubectl completion bash > /etc/bash_completion.d/kubectl
```

Para não precisar reiniciar o terminal, executar

```
source <(kubectl completion bash)
```

# Detalhando recursos com o kubectl describe

Podemos usar o comando **kubectl describe** para detalhar recursos (Pods, Deployments, Services, etc...) do Kubernetes

Cada recurso oferece diferentes informações

# Personalizando o formato de saída de recursos

Podemos especificar o formato de saída (yaml, json, wide, etc...) ao usar o comando **kubectl get ...** com a flag **-o \<FORMATO>**

```
kubectl get deployment/my-dep -o <FORMATO>
```

# Definindo Session Affinity em um Service

O campo **sessionAffinity** pode ser usado como um sticky-session. Por enquanto, conheço somente a opção **ClientIP**, que determina que um cliente com o mesmo IP vai sempre bater na mesma Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
  namespace: alex
spec:
  selector:
    app: my-app
  ports:
    - port: 8081
      targetPort: 8081
      protocol: 'TCP'
  type: LoadBalancer
  sessionAffinity: ClientIP ## Definição do sessionAffinity
```

# Definindo dnsPolicy de um Deployment

o campo **dnsPolicy** se refere para onde as Pods "olharão" primeiro ao se deparar com um DNS. o value **ClusterFirst** denota que DNSes internos do cluster serão consultados primeiramente.

```yaml
# ...
template:
  metatada:
    labels:
      app: my-app
  spec:
    containers:
      - image: xxx
        name: yyy
    dnsPolicy: ClusterFirst
# ...
```


# Definindo restartPolicy de um Deployment

o campo **restartPolicy** se refere à ação que acontecerá quando um Pod não estiver healthy. O restartPolicy: Always define que Pods não-saudáveis sempre serão reiniciados.

```yaml
# ...
template:
  metatada:
    labels:
      app: my-app
  spec:
    containers:
      - image: xxx
        name: yyy
    restartPolicy: Always
# ...
```

# Exemplo 12 e a limitação de recursos de uma Pod com o campo resources

O campo "limits" define o máximo que o Kubernetes irá fornecer de recursos
O campo "requests" define a quantidade de recursos que o Kubernetes **garantirá** aos Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-dep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-application
          image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
          livenessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 30
          resources: # Requests de recursos são definidos nas config do contêiner
            limits:
              memory: 512Mi
              cpu: "500m"
            requests:
              memory: 256Mi
              cpu: "250m"
```
