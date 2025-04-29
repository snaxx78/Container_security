# Session 3 : Déploiement et Sécurisation d'un Cluster Kubernetes


## Introduction

Ce rapport documente la mise en œuvre et la sécurisation d'un cluster Kubernetes à l'aide de différents outils. L'objectif principal était de se familiariser avec Kind pour déployer un cluster local, d'implémenter un système RBAC (Role-Based Access Control), d'évaluer la sécurité du cluster avec Kube-Bench et de mettre en place un système de détection d'intrusion avec Falco.

## 1. Déploiement d'un cluster Kubernetes avec Kind

### 1.1 Installation des outils requis

La première étape consistait à installer les outils nécessaires :
- Kind (Kubernetes in Docker) pour créer un cluster local
- kubectl pour interagir avec le cluster

### 1.2 Configuration et création du cluster

Un fichier de configuration YAML a été utilisé pour spécifier l'architecture du cluster avec 2 nœuds master et 2 nœuds worker :

```yaml
# cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: worker
- role: worker
```

Commande utilisée pour créer le cluster :
```bash
kind create cluster --config cluster-config.yaml
```

![image](https://github.com/user-attachments/assets/208cc95a-dbba-42af-8bba-093f8b4a86ae)


### 1.3 Vérification de l'état du cluster

Pour vérifier l'état du cluster:
```bash
kubectl get nodes
```

![image](https://github.com/user-attachments/assets/7f7de32b-54fa-4af1-9ce5-f870c134d004)



### 1.4 Informations sur le cluster

Pour afficher la liste des namespaces :
```bash
kubectl get namespaces
```

![image](https://hackmd.io/_uploads/rkXsvJpyll.png)


La version de Kubernetes déployée :
```bash
kubectl version
```

![image](https://hackmd.io/_uploads/HyC2wJa1eg.png)


## 2. Expérimentation des RBAC (Role-Based Access Control)

### 2.1 Création d'un namespace de test

```bash
kubectl create ns test-rbac
```

![image](https://hackmd.io/_uploads/SyPuO161xe.png)


### 2.2 Déploiement d'un pod nginx

Déploiement d'un pod nginx dans le namespace test-rbac à l'aide du fichier YAML fourni :

```bash
kubectl apply -f mon-pod.yaml
```

![image](https://hackmd.io/_uploads/Bkss_Ja1el.png)


Pour afficher les logs du pod :
```bash
kubectl logs nginx -n test-rbac
```

![image](https://hackmd.io/_uploads/rJrlt1TJxx.png)


### 2.3 Création et implémentation d'un rôle

Création d'un rôle permettant la lecture des pods dans le namespace test-rbac :

```bash
kubectl apply -f role-pod-reader.yaml
```

Pour afficher ce rôle :
```bash
kubectl get role pod-reader -n test-rbac -o yaml
```

![image](https://hackmd.io/_uploads/SJgKF1pkex.png)


### 2.4 Liaison du rôle à un utilisateur

J'ai associé le rôle créé à un utilisateur fictif nommé "titi" :

```bash
kubectl apply -f rolebinding-pod-reader.yaml
```

![image](https://hackmd.io/_uploads/Bkbx9y61el.png)


### 2.5 Création et configuration de l'utilisateur

Pour créer l'utilisateur "titi" ensuite :

1. Extraction des certificats CA du cluster :
```bash
docker cp kind-control-plane:/etc/kubernetes/pki/ca.crt .
docker cp kind-control-plane:/etc/kubernetes/pki/ca.key .
```

2. Génération des clés et certificats pour l'utilisateur :
```bash
openssl genrsa -out titi.key 2048
openssl req -new -key titi.key -out titi.csr -subj "/CN=titi"
openssl x509 -req -in titi.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out titi.crt -days 365
```

3. Ajout de l'utilisateur au contexte kubectl :
```bash
kubectl config set-credentials titi --client-certificate=titi.crt --client-key=titi.key
```

4. Création d'un contexte pour cet utilisateur :
```bash
kubectl config set-context titi-context --cluster=kind-kind --namespace=test-rbac --user=titi
```

5. Basculement vers ce contexte :
```bash
kubectl config use-context titi-context
```

![image](https://hackmd.io/_uploads/Hy9CjQ6Jgl.png)


### 2.6 Tests des permissions RBAC

En utilisant le contexte titi-context, test des permissions accordées :

1. Listage des pods dans le namespace test-rbac :
```bash
kubectl get pods -n test-rbac
```

![image](https://hackmd.io/_uploads/B1876ma1ex.png)


2. Tentative de création d'un pod dans le namespace test-rbac :
```bash
kubectl apply -f test-pod.yaml
```

Résultat : Cette opération échoue avec un message d'erreur indiquant que l'utilisateur "titi" n'a pas les permissions nécessaires pour créer des pods. L'erreur spécifique était :
```
Error from server (Forbidden): pods is forbidden: User "titi" cannot create resource "pods" in API group "" in the namespace "test-rbac"
```

![image](https://hackmd.io/_uploads/ByJ5CmaJxl.png)


Retour au contexte par défaut :
```bash
kubectl config use-context kind-kind
```

## 3. Analyse de sécurité avec Kube-Bench

### 3.1 Déploiement de Kube-Bench

Déploiement de Kube-Bench en utilisant un Job Kubernetes :

```bash
kubectl apply -f job.yml
```


![image](https://hackmd.io/_uploads/rJIB98Tkle.png)


### 3.2 Résultats de l'analyse

Après avoir consulté les résultats du scan :

```bash
kubectl logs -l job-name=kube-bench
```

![image](https://hackmd.io/_uploads/rJnSOd01lg.png)


### 3.3 Résumé des résultats Kube-Bench

Kube-Bench a évalué la configuration du cluster selon les recommandations du CIS (Center for Internet Security) Kubernetes Benchmark. Voici un résumé des résultats :

Le scan kube-bench a détecté 17 échecs critiques et 41 avertissements.
Cela reflète l'absence de politiques de sécurité avancées sur un cluster local Kind.

C'est normal car notre cluster est minimal et les sécurité avancées ne doive pas être activée par défaut.

## 4. Détection d'intrusions avec Falco

### 4.1 Installation de Falco

Installation de Falco en utilisant Helm :

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
kubectl create ns falco
helm -n falco install falco falcosecurity/falco --set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true
```

Vérification des pods :
```bash
kubectl get pods -n falco
```

![image](https://hackmd.io/_uploads/BkziR_Ryle.png)


### 4.2 Configuration du port-forward pour l'UI

```bash
kubectl port-forward svc/falco-falcosidekick-ui 2802:2802 --namespace falco
```

[SCREENSHOT 16: Interface web de Falco]

### 4.3 Tests de détection avec Falco

#### 4.3.1 Déploiement d'un pod de test

```bash
kubectl apply -f mon-pod.yml
```

#### 4.3.2 Exécution d'un shell dans le pod

```bash
kubectl exec -it front -- sh
```

![image](https://hackmd.io/_uploads/S1i5wYA1lg.png)

![image](https://hackmd.io/_uploads/H1yAPKAklx.png)


**Résultat observé :** Une alerte a été détectée dans l'interface Falco avec la priorité "Notice". La règle déclenchée était "A shell was spawned in a container with an attached terminal" qui détecte l'utilisation d'un terminal interactif dans un conteneur.

#### 4.3.3 Requête vers l'API Kubernetes

Dans le shell du pod :
```bash
apk add curl
curl -k http://10.96.0.1:80
```

![image](https://hackmd.io/_uploads/Byq-OKA1gl.png)


**Résultat observé :** Une alerte de priorité "Notice" a été générée. La règle déclenchée était "Unexpected connection to K8s API Server from container" qui détecte les tentatives d'accès à l'API Kubernetes depuis un conteneur.

