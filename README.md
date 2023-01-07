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
