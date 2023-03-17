# Startando e stoppando o minikube

O minikube é uma "VMzinha" que simula um cluster Kubernetes, sem inúmeros worker nodes para economizar recursos.

- **minikube start** – Starta a VM do minikube. Com a flag **–driver=[DRIVER_NAME]** podemos especificar o driver (docker, virtualbox, etc)
- **minikube stop** - Stoppa a VM do minikube.
- **minikube status** – Verifica o status da VM criada. Útil para verificar se está rodando.
- **minikube delete** – Deleta a VM criada.
- **minikube dashboard** – Abre uma aba no browser com um dashboard do Kubernetes.

# Pingando o host do minikube

Podemos pingar e referenciar o host do minikube com o DNS **host.minikube.internal**, da mesma forma que utilizávamos o **host.docker.internal** no Docker

```bash
ping host.minikube.internal
```

# Listando recursos com o kubectl get

```bash
kubectl get <RECURSO>
```

# Criando Deployments com o kubectl create no modo imperativo

```bash
kubectl create deployment '<DEPLOYMENT_NAME>' --image='<IMAGE>'
```

# Deletando Deployments com o kubectl delete

```bash
kubectl delete deployment '<DEPLOYMENT_NAME>'
```

Ou, caso o deployment tenha sido criado na forma declarativa (arquivo), também é possível deletar apontando para o arquivo

```bash
kubectl delete -f=deployment.yaml -f=service.yaml -f=...
```

# Criando um Service para expor uma aplicação com o kubectl expose

```bash
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

```bash
kubectl scale deployment/<DEPLOYMENT> --replicas=<DERIRED_NUMBER_OF_PODS>
```

Antes do escalonamento:

![image](https://user-images.githubusercontent.com/80921933/210464647-32c9dc80-c107-4276-816c-79230ac0d41c.png)

Comando rodado:

```bash
kubectl scale deployment/spring-app --replicas=5
```

Depois do escalonamento:

![image](https://user-images.githubusercontent.com/80921933/210464692-93118384-fb87-46a2-b5a2-13b8bc3cbb7c.png)

# Atualizando a docker-image de um deployment

```bash
kubectl set image deployment/<DEPLOYMENT> <CONTAINER>=<NEW_IMAGE>
```

**\<CONTAINER>** é a **base da imagem** utilizada, sem a tag e **sem o nome do usuário de repositório**, ex: Na imagem **azold6/imagemteste:6**, a base da imagem seria **imagemteste**

Para verificar o progresso das mudanças, executar 

```bash
kubectl rollout status deployment/<DEPLOYMENT>
```

Caso a seguinte saída aconteça, os Pods foram atualizados

![image](https://user-images.githubusercontent.com/80921933/210465852-881647b1-1dc7-4bea-bcad-29add2ed49ab.png)

# Realizando o rollout de uma alteração em um Deployment

Com o seguinte comando, podemos verificar o histórico de alterações em um deployment

```bash
kubectl rollout history deployment/<DEPLOYMENT>
```

Teremos uma saída nos informando sobre todas as alterações

![image](https://user-images.githubusercontent.com/80921933/210469570-512f78e3-12fb-4598-a775-934f0df94bde.png)

Podemos detalhar uma revision adicionando a flag **--revision=<REVISION_NUMBER>**

```bash
kubectl rollout history deployment/<DEPLOYMENT> --revision=2
```

Após escolher uma revision de destino para realizar o rollback, executamos

```bash
kubectl rollout undo deployment/<DEPLOYMENT> --to-revision=<REVISION_NUMBER>
```

Pronto! Para verificar o progresso das mudanças, executar 

```bash
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

**emptyDir** - Nasce "junto" com um pod, vazio. Fica vinculado a uma Pod, ou seja, não morre enquanto a Pod existir, mas se a Pod for substituida, o volume "reinicia" (fica fazio novamente). Usado somente quando precisamos de dados temporários <br>
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

Para esse exemplo, o ConfigMap acima foi referenciado no Deployment

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

# Exemplo 21 e puxando todas as envs de um ConfigMap com o envFrom

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: busybox
      image: busybox
      envFrom: ###################### Podemos puxar todas as envs
        - configMapRef:             # contidas em um ConfigMap 
            name: my-configmap ###### de uma só vez com o envFrom
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

```bash
kubectl create namespace <insert-namespace-name-here>
```

# Listando namespaces disponíveis

```bash
kubectl get namespaces
```

# Inserindo recursos em namespaces

Caso tenhamos um recurso em um arquivo yaml, e queiramos inserí-lo em um namespace, usamos o seguinte comando

```bash
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

```bash
kubectl get pods -n=<NAMESPACE>
```

Para buscar recursos de todos os namespaces, podemos executar

```bash
kubectl get pods --all-namespaces
```

# Habilianto o auto completion para comandos do Kubernetes

**Update:** Tive problemas com os passos abaixo. Alternativamente, é possível usar o seguinte tutorial: 

https://spacelift.io/blog/kubectl-auto-completion

-------------Tutorial antigo-------------

Trocar para o root user

```
sudo su -
```

Executar

```bash
kubectl completion bash > /etc/bash_completion.d/kubectl
```

Para não precisar reiniciar o terminal, executar

```bash
source <(kubectl completion bash)
```

-------------Tutorial antigo-------------

# Detalhando recursos com o kubectl describe

Podemos usar o comando **kubectl describe** para detalhar recursos (Pods, Deployments, Services, etc...) do Kubernetes

Cada recurso oferece diferentes informações

# Personalizando o formato de saída de recursos

Podemos especificar o formato de saída (yaml, json, wide, etc...) ao usar o comando **kubectl get ...** com a flag **-o \<FORMATO>**

```bash
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

O campo **limits** define o máximo que o Kubernetes irá fornecer de recursos <br>
O campo **requests** define a quantidade de recursos que o Kubernetes **garantirá** aos Pods

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

# Entrando em Pods com o kubectl exec

O **kubectl exec** funciona de forma exatamente igual ao **docker exec**, com a seguinte sintaxe

```bash
kubectl exec -it <POD> <COMANDO>
```

# Exemplo 13 e a criação de um LimitRange para aplicar limites em recursos em um namespace

limitacao.yaml

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-alex
  namespace: alex
spec:
  limits:
  - default: # O máximo de recursos que o Kubernetes irá liberar para Pods
      cpu: '1'
      memory: 500Mi 
    defaultRequest: # O mínimo de recursos que o Kubernetes irá liberar para Pods
      cpu: '0.5'
      memory: 250Mi 
    type: Container
```

Devemos aplicar esse LimitRange em algum namespace. Nesse caso, aplicarei no namespace **alex**

```bash
kubectl apply -f=limitacao.yaml -n=alex
```

Agora, executando o **kubectl describe**, podemos confirmar que a limitação funcionou

![image](https://user-images.githubusercontent.com/80921933/211369645-333c6fc9-e1f5-404d-9097-bc18afd25270.png)

# Adicionando e removendo Taints de nodes

NoSchedule e NoExecute
Consultar aula de Taints do Day 2

# Referência do Deployment de um app Spring usando volumes, configmaps e limitranges

A aplicação completa pode ser encontrada no seguinte repositório:

https://github.com/azl6/spring-fileupload-simple

Todos os yaml estão dentro da pasta **kubernetes**. Abaixo, segue o exemplo de um Deployment completo da aplicação. Para mais detalhes, favor direcionar-se para o link acima

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fileupload-dep
  namespace: alex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fileupload
  template:
    metadata:
      labels:
        app: fileupload
    spec:
      containers:
        - name: fileupload
          image: azold6/fileupload-pv-pvc:latest
          livenessProbe:
            httpGet: 
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 30
            initialDelaySeconds: 10
          #resources: ###############################
          #  limits:                                #
          #    cpu: '1'                             # Descomentar caso haja LimitRange
          #    memory: 500Mi                        # No namespace definido aqui...
          #  requests:                              #
          #    cpu: '2'                             #
          #    memory: 800Mi ########################
          env:
            - name: UPLOADED_FILES_DIR
              valueFrom:
                configMapKeyRef:
                  name: my-configmap
                  key: FILES_UPLOAD_CHOSEN_DIR
          volumeMounts:
            - mountPath: /home/my-custom-dir
              name: uploaded-files-storage
      volumes:
        - name: uploaded-files-storage
          persistentVolumeClaim:
            claimName: my-pvc
```

# Aplicando labels via CLI

Podemos usar o comando abaixo, substituindo **pod** pelo recurso desejado

```bash
kubectl label pod spring-app-6985dcbc89-8hbnx projeto=site_markket
```

Para **atualizar labels**, basta passar a flag --overwrite

```bash
kubectl label pod spring-app-6985dcbc89-8hbnx projeto=projeto_governo --overwrite
```

# Listando recursos com uma label desejada

Basta passar a flag **-l** nos comandos do Kubernetes

```bash
kubectl get pods -l app=spring-app
```

# Listando as labels de um recurso

Substituir pod pelo recurso adequado. 

```
kubectl label pod my-pod --list
```

# Exemplo 14 e a utilização do nodeSelector para subir Pods em nodes com certas labels

Podemos usar a chave **nodeSelector** para selecionar os nodes onde subiremos nossas Pods. Essa seleção é feita através de labels.

Exemplo de yaml com nodeSelector:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-dep
  namespace: default
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
      nodeSelector:
        disk: SSD # Pods só serão criadas em nodes que contém o label disk: SSD
```

# Utilização do ReplicaSet

O ReplicaSet é o controller das Pods. Sempre que um Deployment é criado, automaticamente é criado um ReplicaSet.

São importantes pelos seguintes motivos:

- **Verificação de logs caso alguma pod não seja criada de forma bem-sucedida**
    
    Passei por uma situação onde havia um LimitRange aplicado ao namespace. Ao tentar criar Pods com requests de recursos acima do limite estabelecido no LimitRange, **não recebia nenhum log na CLI** (como acontece de costume quando os limits e requests são definidos em um só Deployment/manifesto). Só consegui consultar tais logs através do ReplicaSet, com o **kubectl describe replicaset my-replicaset**
    
- **Entendimento do funcionamento de rollouts e do Kubernetes no geral**

    Por debaixo dos panos, o **kubectl rollout undo ...** se utiliza de ReplicaSets antigos para realizar o rollback. Da mesma forma, a cada versão nova do Deployment, temos um novo ReplicaSet criado.
    
# Utilização do DaemonSet

É utilizado quando precisamos **garantir** que uma aplicação está rodando em todos os nós do cluster.

Algumas implicações importantes:

- O nó master **não deve ter taint** quando rodarmos um DaemonSet, para que a aplicação consiga subir nele. Depois que a aplicação subir no master, **devemos re-aplicar um taint NoSchedule** lá, já que nenhum outro Pod deve ser agendado para lá. 
- Caso tenhamos subido uma Pod no master, e precisemos atualizar a imagem com o **kubectl set image..**, podemos: Atualizar o DaemonSet, deletar o Pod rodando atualmente no nó master (já que ele não será automaticamente deletado por causa do taint NoSchedule), e automaticamente uma nova Pod será criada no master com a versão mais atual da imagem.

# Exemplo 15 e a criação de um DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      system: my-system
  template:
    metadata:
      labels:
        system: my-system
    spec:
      containers:
        - name: spring-application
          image: azold6/jenkins-with-spring:jenkins-spring-pipeline-49
```

# Exemplo 16 e a utilização do campo strategy para definir comportamento em update de imagem em Pods

O campo **strategy** é usado para definir a estratégia de atualização de Pods. Sem esse campo, as Pods não são imediatamente substituidas, e devemos usar o comando **kubectl rollout status \<RESOURCE> <RESOURCE_NAME>**.

Esse campo pode ser usado tanto em **Deployments** quanto em **DaemonSets**

**Obs:** **maxSurge** e **maxUnavailable** podem ser informados por porcentagem (5%, 10%...)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-dep
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
  strategy: ######## Configuração de atualização de Pods
    type: RollingUpdate 
    rollingUpdate:
      maxSurge: 2 ## De quanto em quanto novas Pods serão adicionadas em um update
      maxUnavailable: 3 ## Máximo de contêineres "unavailable" em um update
```

# Informações detalhadas do cluster com o kubectl cluster-info

Podemos executar o comando abaixo para encontrar diversas informações do cluster, como CIDR, nodes, etc.

```bash
kubectl cluster-info dump > myclusterinfo.txt
```

O redirect será feito para o arquivo **myclusterinfo.txt** e poderemos analisar o output para encontrar as informações desejadas.

# Como encontrar arquivos armazenados nos volumes criados

Por enquanto, isso funciona para o volume emptyDir.

Executar, como root:

```bash
cd /var/lib/kubelet/pods
```

Depois, buscar pelo nome do volume criado

```
find . -iname "<VOLUME_NAME>"
```

# Exemplo 17 e a utilização de um NFS em um PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  storageClassName: standard
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs: # Definição do NFS
    path: /tmp/nfs-server
    readOnly: false
    server: host.minikube.internal
```

# Exemplo 18 e a criação de CronJobs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: giropops-cron
spec:
  schedule: "*/1 * * * *" # Definição do cron
  jobTemplate: # Template com o que será rodado
    spec:
      template:
        spec:
          containers:
          - name: giropops-cron
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Bem Vindo ao Descomplicando Kubernetes - LinuxTips VAIIII ;sleep 30
          restartPolicy: OnFailure
```

# Criando um secret com a CLI

Podemos criar um secret a partir de:

- Files (--from-file=)
- Env files (--from-env-file=)
- Literal (--from-literal=)

```bash
kubectl create secret generic <SECRET_NAME> <FLAG_FROM_ACIMA>
```

**Exemplos:**

- Criando secret a partir de um arquivo
  
  ```bash
  kubectl create secret generic my-file-secret --from-file=myfile.txt
  ```
  
- Criando secret a partir de **literals**
  
  ```bash
  kubectl create secret generic my-literal-secret --from-literal usuario=alex --from-literal senha=123
  ```
  
  ![image](https://user-images.githubusercontent.com/80921933/213944675-34b322d0-bb27-4d65-93cf-01144ee42350.png)


# Exemplo 19 e montando secrets como volumes

É possível "montar" secrets como se eles fossem volumes, da mesma forma que montamos emptyDir, hostPath, nfs, etc...

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3600"
      volumeMounts:
        - name: secret-volume 
          mountPath: /tmp/secretVolume
  volumes:
    - name: secret-volume
      secret: # Um secret pode ser montado como se fosse um emptyDir, hostPath, nfs, etc.
        secretName: my-secret
```

# Exemplo 20 e referenciando secrets com o valueFrom secretKeyRef

É uma forma de referenciar secrets literals, que são key-value.

Nesse exemplo, uma secret de um literal foi referenciada. Para mais informações de uma literal secret, consultar o tópico [Criando um secret com a CLI](#criando-uma-secret-com-a-cli)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: busybox
      image: busybox
      command:
        - sleep
        - "3000"
      env:
      - name: USERNAME
        valueFrom: # Definindo que a env vem de uma secret
          secretKeyRef:
            name: my-literal-secret # Nome da secret
            key: usuario # Chave da literal secret
      - name: PASSWORD
        valueFrom: # Mesma coisa da outra env...
          secretKeyRef:
            name: my-literal-secret
            key: senha
```

# Criando Service-Accounts

Um service-account é uma conta no cluster, que possui permissões (roles).

```
kubectl create serviceaccount <SERVICEACCOUNT_NAME>
```

# Listando e detalhando Cluster-Roles

**Cluster Roles** são permissões do cluster, agrupadas. Por exemplo, a role **cluster-admin** pode fazer tudo (criar recursos, deletar recursos, etc...)

```
kubectl get clusterroles
```

![image](https://user-images.githubusercontent.com/80921933/213954193-31700600-8bec-401a-a54b-ac2efba3d7bd.png)


Também podemos descrever uma Cluster-Role com o comando:

```
kubectl describe clusterrole <CLUSTERROLE_NAME>
```

![image](https://user-images.githubusercontent.com/80921933/213954216-c699ce91-e8b0-43a7-8171-69e82047edb9.png)

# Listando Cluster-Role-Bindings

São vínculos de roles a service-accounts.

```
kubectl get clusterrolebindings
```

Também é possível descrevê-los com o **describe**

# Vinculando um service-account a uma cluster-role com um cluster-role-binding

Estrutura do comando:
```bash
kubectl create clusterrolebinding <NOME_CLUSTERROLEBINDING> --serviceaccount=<SA_NAMESPACE>:<SA_NAME> --clusterrole=<ROLE_NAME>
```

Exemplo:
```bash
k create clusterrolebinding alex_virou_admin --serviceaccount=default:alex --clusterrole=cluster-admin
```

# Exemplo 22 e Criando um ServiceAccount via yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user ########## Nome e namespace
  namespace: kube-system #### do service-account
```

# Exemplo 23 e Criando um ClusterRoleBinding para vincular contas e roles via yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole ################# Informando o cluster-role
  name: cluster-admin ############### a ser bindado
subjects:
- kind: ServiceAccount ############## Informando o service-account
  name: admin-user ################## a ser bindado
  namespace: kube-system ############ e seu namespace
```

# Adicionando um repositório para os helm-charts

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

# Buscando repositórios disponíveis

Neste caso, bitnami foi o nome que dei ao repositório no passo acima

```bash
helm search repo bitnami
```

# Atualizando o repositório adicionado

```bash
helm repo update
```

# Instalando um chart

```bash
helm install bitnami/mysql --generate-name
```
# Desinstalando um chart

Para isso, primeiramente, devemos encontrar o nome do chat que desejamos desinstalar, com o **helm ls**

![image](https://user-images.githubusercontent.com/80921933/214358903-33c8177b-cc14-4a2f-921c-082e76ab3152.png)

Nesse caso, o nome é **mysql-1674579050**. Com essa informação, basta rodar:

```bash
helm unninstall mysql-1674579050
```
# Printando a tela inicial ao iniciar um chart

```bash
helm status <CHART_NAME>
```

# Printando detalhes de um chart

```bash
helm show chart <REPO_NAME>/<CHART_NAME>
```

Exemplo:

```bash
helm show chart bitnami/mysql
```

# Criando um helm chart

Executar

```bash
helm create <CHART_NAME>
```

Nesse exemplo, usei o nome **exemplo1** para o chart. Será criada uma estrutura de pastas, da seguinte forma:

![image](https://user-images.githubusercontent.com/80921933/214372349-1ce43289-529e-4a75-b494-7e5e61f8f6d5.png)

<br>

**Explicações dos arquivos:**

- `Chart.yaml`:
  
  É onde os metadados do seu chart ficarão, como versão do chart, versão da aplicação ,descrição, mantenedores, etc...
  
- `templates/deployment.yaml`

  Funciona exatamente como um deployment padrão usado no Kubernetes, porém, com substituição de valores utilizando chaves duplas 
  
  Um exemplo é o seguinte: Dentro do **deployment.yaml**, temos a seção **spec.template.spec.containers.name**. Em seu preenchimento, consta o seguinte valor:
  
  ![image](https://user-images.githubusercontent.com/80921933/214375273-4bc91455-9eb2-4b8b-8547-1434da84ce5c.png)

  Tal valor será substituído pelo que se encontra no arquivo **Chart**, no campo **name**
  
  ![image](https://user-images.githubusercontent.com/80921933/214375464-7d6e3175-72de-4414-8046-ad70a36fabba.png)
  
- `values.yaml`

  Trata-se de onde a maioria das variáveis "substituíveis" do `deployment.yaml` estarão
  
<br>
  
Finalmente, para criarmos o helm-chart com os valores customizados, executamos:

```bash
helm install <NOME_CHART> <CAMINHO_HELMCREATE> --values <CAMINHO_DO_VALUESPONTOYAML>
```

Em nosso exemplo, como nomeamos nosso chart de **exemplo1** no **helm create...**, executaremos (uma pasta acima do diretório gerado pelo **helm create**):

```bash
helm install exemplo1 exemplo1/ --values exemplo1/values.yaml
```

Pronto!

# Atualizando helm-charts

Ao realizar mudanças em um **helm-chart**, devemos subir o campo **version** do **Chart.yaml**.

Depois, basta executar:

```bash
helm upgrade <NOME_CHART> <CAMINHO_HELMCREATE> --values <CAMINHO_DO_VALUESPONTOYAML>
```

# Realizando um rollback de um helm-chart para outra revisão

Podemos listar todas as revisões com o comando

```bash
helm history <NOME_CHART>
```

Após escolher a revisão, basta executar:

```bash
helm rollback <NOME_CHART> <REVISION_NUMBER>
```

# Diferenças ClusterIP, NodePort e LoadBalancer

Ao criar um Service, podemos declarar esses três tipos de "exposição":

**ClusterIP:** Gera um IP acessível dentro do cluster <br>
**NodePort:** Gera uma porta (range 30000-32767) em todos os worker-nodes para expor Pods para o mundo externo <br>
**LoadBalancer:** Gera um IP publicamente acessível em cloud-providers

# Criando um Ingress

**Importante**: Para mais detalhes dessa implementação, consultar o diretório `./praticas/refazendo_ingress/`. Ele contém os arquivos usados nessa implementação. Para mais detalhes de como implementar o Ingress "na unha" (criando o deployment, service, configmap, CR, CRB, ServiceAccount, etc...) consultar o arquivo `InformaçõesIngress.txt` 

O **Ingress** é uma forma de expormos nossos services para a internet pública através de um **Ingress Controller**.

**Ingress Controllers** são aplicações que fazem o proxy-reverso do tráfego para que o usuário seja direcionado para o **Service** (e consequentemente **Pods**) correto. Os mais famosos são o **nginx-ingress-controller** e o **haproxy**

![image](https://user-images.githubusercontent.com/80921933/219681298-fca26031-8f84-4072-9e96-0c1f1e17fb9a.png)

No exemplo que será explicado aqui, fizemos uso da instalação do **nginx-ingress-controller** através do **Helm**. É uma instalação tranquila e basta buscar um tutorial na internet. Nesse caso, o **nginx-ingress-controller** é instalado como um **DaemonSet** (roda em todos os nós do Cluster, exceto o master que geralmente tem o taint NoSchedule)

Depois de executar o `helm install` para instalar o **nginx-ingress-controller**, precisamos criar as regras do nosso **Ingress**. O **Ingress** é uma "extensão" gerada pelo **nginx-ingress-controller** que nos permite descrever as regras de roteamento dele. Com essa fala, quero dizer que a instalação do **nginx-ingress-controller** por si sí não basta. Ainda é necessário definir as regras de roteamento, e essa é a responsabilidade do manifesto do tipo **Ingress**.

O Ingress tem a seguinte cara:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-01-ingress
  namespace: app-01
  labels:
    name: app-01-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app-01.alex.stefanomartins.net ####### DNS Record criado no Route 53
    http:
      paths:
      - path: /app-01 ########################## Para que essa regra seja satisfeita, o usuário deve acessar http://app-01.alex.stefanomartins.net/app-01
        pathType: Prefix      
        backend:
          service:
            name: app-01-service ############### Nome do service para qual o acesso ao endereço acima direcionará
            port: 
              number: 5678 ##################### Porta do serviço para qual o acesso ao endereço acima direcionará
```

Algumas anotações importantes:

- A primeira é que como nós estamos utilizando uma URI, nós temos que utilizar aquela annotation para que ele faça o rewrite.
- A segunda pegadinha é que agora é obrigatória a especificação de um host único.
- A terceira pegadinha é que agora também é obrigatória a especificação da classe do Ingress. Caso contrário, ele retorna um 404.

Depois, considerando que temos um **Deployment** (nesse caso, o **Deployment** utilizava a imagem `hashicorp/http-echo`, que escuta na porta 5678, como mostrado no Ingress) e um **Service** o expondo (mesmo que seja ClusterIP) no namespace `app-1`, basta deployarmos o **Ingress** nesse mesmo namespace.

Considerando que o nosso **DNS Record** no **Route 53** já está apontando para os IP's dos nosso worker-nodes, ao acessar app-01.alex.stefanomartins.net/app-1, devemos ver o conteúdo do nosso **Deployment** exposto.

# Criando um cluster com o eksctl

Fonte: https://eksctl.io/

Depois de subir o cluster, criamos uma IAM Policy com permissões no R53 (Day 6, Aula 5, Minuto 5:40)

```bash
eksctl create iamserviceaccount --name external-dns --namespace default --cluster CLUSTERNAME --attach-policy-arn POLICYARN --apṕrove
```
