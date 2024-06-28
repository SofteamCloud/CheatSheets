# kubernetes

## minikube

### minikube reset
```
minikube delete
```
### minikube specify version
```
minikube start --kubernetes-version=v1.9.0
```
### minikube inside docker
```
minikube start —vm-driver=none
```
### update minikube
```
minikube update-check                  #check if update needed

sudo rm -rf /usr/local/bin/minikube    # unlink existing minikube
brew update                            # update brew itself
brew cask reinstall minikube           # reinstall latest minikube
```
### partage de sa registry docker local avec minikube
```
eval $(minikube docker-env)

start registery local :
docker container run -d -p 5000:5000 registry
```
## kubernetes-commandes
### get status of K8s component
```
kubectl get cs
```
### get custom information
```
kubectl get nodes \
 -o=custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu
```
### explain commande
```
k explain deploy --recursive
```

Commands to quikly create k8s resources.
```
kubectl run -h | grep '# ' -A2
tldr kubectl run
```

### check last 10 events on pod
```
k describe pod <pod-name> | grep -i events -A 10`
```

### get all k8s resources
```
kubectl api-resources 
```
### find api_group/version for a resource
```
k api-resources | grep -i "resource name"
k api-versions | grep -i "api_group name"
```
### get sample yaml 
```sh
kubectl create deploy test --image=alpine --dry-run -o yaml
# or
kubectl run my-cool-app —-image=me/my-cool-app:v1 \
  -o yaml --dry-run > my-cool-app.yaml
  
kubectl create secret generic my-secret --from-literal=foo=bar -o yaml --dry-run > my-secret.yaml.
```

### run busybox inside cluster
```
kubectl run -ti --image=odise/busybox-curl busybox --rm
```

### be admin on cluster GCP
```
kubectl create clusterrolebinding cluster-admin-binding-<<trigramme>> --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)
```

### delete list
```
k delete -n kanary $(k get po -o name -n kanary)
k delete $(k get po -o name -n logging | grep fluent)
```

### force pod delete

```
k delete po <name> --grace-period=0 --force
```

### view secret
```
 k get secret alertmanager-main
 ```
### replace secret
```
 kubectl create secret generic alertmanager-main --from-literal=alertmanager.yaml="$(< alertmanager.yaml)" --dry-run -oyaml | kubectl  replace secret --filename=-
```

## Dashboard
install : dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
### Open tunnel
```
kubectl proxy
```
### Get token to login
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep  clusterrole-aggregation-controller-token | awk '{print $1}')
```

### force to patch
```
k patch deploy haproxy-kanary -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"
```

### port-forward
```
  kubectl port-forward -n monitoring prometheus-k8s-0 9090

  kubectl port-forward $(kubectl get  pods --selector=id=kibana-logging -n logging --output=jsonpath="{.items..metadata.name}") -n logging  5601

  kubectl port-forward $(kubectl get  pods --selector=app=grafana -n monitoring --output=jsonpath="{.items..metadata.name}") -n monitoring 3000
```


### list docker images on cluster
```
kubectl get po --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n'| sort -u
```

## Finde service cidr
```
echo '{"apiVersion":"v1","kind":"Service","metadata":{"name":"dumy-test-to-get-error"},"spec":{"clusterIP":"0.0.0.0","ports":[{"port":443}]}}' | kubectl apply -f - 2>&1 | sed 's/.*The range of valid IPs is //'
```
## Find cluster info 
```
kubectl cluster-info dump | grep -I cidr
```

## customize bash

utilitaire shell [kube_ps1](https://github.com/jonmosco/kube-ps1) pour modifier notre prompt pour clairement visualiser le contexte actif :

```
dev $ kubeon
dev (☸ |kubernetes-admin@kubernetes:default) $
```
l’outil [kubectx](https://github.com/ahmetb/kubectx) permet la gestion de nouveau contexte pour travailler dans un namespace :

```
dev (☸ |kubernetes-admin@kubernetes:default) $ kubectl config set-context cluster-prod --cluster=kubernetes --user=kubernetes-admin --namespace=prod
Context "cluster-prod" created.
```

## debug 

## Start point 
to get a good idea of where to start debugging, e.g. broken nodes, infra issues
```
kubectl get nodes
kubectl cluster-info dump
```

more [here](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)
## Logs

### Kubelet logs
 provide pointers to underlying infra issues
```
journalctl -xeu kubelet.service
```

## Api server logs 
```
tail /var/log/kube-apiserver.log 
less /var/containers/kube-apiserver-...log
```

## Pods 
```
kubectl get pods -A
```

Get diiference between origin manifest 
```
kubectl diff -f ./my-manifest.yaml
```

## event 
```
kubectl get events --sort-by=.metadata.creationTimestamp

```
