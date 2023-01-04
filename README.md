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

# Criando deployments com o kubectl create no modo imperativo

```
kubectl create deployment '<DEPLOYMENT_NAME>' --image='<IMAGE>'
```

# Deletando deployments com o kubectl delete

```
kubectl delete deployment '<DEPLOYMENT_NAME>'
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

**\<CONTAINER>** é a base da imagem utilizada, sem a tag.

Para "confirmar" as mudanças, executar 

```
kubectl rollout status deployment/<DEPLOYMENT>
```

Caso a seguinte saída aconteça, os Pods foram atualizados

![image](https://user-images.githubusercontent.com/80921933/210465852-881647b1-1dc7-4bea-bcad-29add2ed49ab.png)



