# Startando e stoppando o minikube

O minikube é uma "VMzinha" que simula um cluster Kubernetes, sem inúmeros worker nodes para economizar recursos.

- **minikube start** – Starta a VM do minikube. Com a flag **–driver=[DRIVER_NAME]** podemos especificar o driver (docker, virtualbox, etc)
- **minikube stop** - Stoppa a VM do minikube.
- **minikube status** – Verifica o status da VM criada. Útil para verificar se está rodando.
- **minikube delete** – Deleta a VM criada.
- **minikube dashboard** – Abre uma aba no browser com um dashboard do Kubernetes.

# Criando deployments com o kubectl create no modo imperativo

```
kubectl create deployment '<DEPLOYMENT_NAME>' --image='<IMAGE>'
```

# Exemplo de criação de deployment no modo imperativo

```
kubectl create deployment 'my-deployment' --image='azold6/jenkins-with-spring:jenkins-spring-pipeline-53'
```

# Deletando deployments com o kubectl delete

```
kubectl delete deployment '<DEPLOYMENT_NAME>'
```

