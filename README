# Déployer un blog sur GKE

Cet atelier est basé sur l'exemple de Kelsey Hightower lors de la KubeCon Europe 2016
[Kubecon Talk](https://github.com/kelseyhightower/talks/tree/master/kubecon-eu-2016/demo)

## Prérequis

Le présent exercice nécessite l’utilisation des clients gcloud et kubectl.

Il y a deux possibilités :
- Utiliser [CloudShell](https://cloud.google.com/shell/docs/quickstart) qui est accessible depuis la console Google Cloud (recommandé)
- ou Installer les clients
  - [gcloud](https://cloud.google.com/sdk/docs/quickstarts)
  - [kubect](https://cloud.google.com/container-engine/docs/quickstart#install_the_gcloud_command-line_interface)

Il est possible d'ajouter la [complétion bash/zsh](http://kubernetes.io/docs/user-guide/kubectl/kubectl_completion) pour kubectl.

## Préparation de l’environnement

Avant de commencer les travaux pratiques, télécharger (clonez) les fichiers dont vous aurez besoin.

```
$ git clone https://github.com/jcsirot/cncp-workshop 
$ cd cncp-workshop
```

`kubectl` est un outil en ligne de commande qui simplifie l’utilisation de l’API de Kubernetes.

Vérifier la configuration actuelle de `kubectl`.

```
$ kubectl config view
apiVersion: v1
clusters: []
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

## Création d’un cluster GKE

Nous allons créer un cluster en utilisant l’outil en ligne de command gcloud.
Vous pouvez consulter les options disponibles avec la commande suivante :

```
$ gcloud container clusters create --help
```

**Note:** ne pas oublier de changer le nom du cluster, ici *cncp*.

```
$ gcloud container clusters create cncp --zone europe-west1-b --additional-zones=europe-west1-c --machine-type n1-standard-2 --num-nodes=1 --no-enable-cloud-monitoring --no-enable-cloud-logging
```

La commande ci-dessus crée un cluster constitué de 2 noeuds (suffisant pour l’exercice). Les noeuds seront distribués sur 2 zones pour assurer une disponibilité optimale. Enfin les options de supervision et de logs ne sont pas utilisées.

Consulter la liste des clusters gke :

```
$ gcloud container clusters list
```

Lors de la création du cluster, gcloud s’est occupé de configurer `kubectl`. Cela peut se vérifier en utilisant la commande suivante qui affiche le contexte courant :

```
$ kubectl config current-context
```

Nous pouvons désormais afficher la liste des noeuds:

```
$ kubectl get nodes -o wide
NAME                                  STATUS    AGE       EXTERNAL-IP
gke-cncp-default-pool-0db1a45d-5tcb   Ready     31m       35.189.236.109
gke-cncp-default-pool-6b9e9d42-8m5k   Ready     31m       104.199.51.70
```

Et aussi vérifier la distribution des noeuds sur les différentes zones:

```
$ gcloud compute instances list --filter=name~gke-cncp.*
NAME                                 ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-cncp-default-pool-6b9e9d42-8m5k  europe-west1-b  n1-standard-2               10.132.0.2   104.199.51.70   RUNNING
gke-cncp-default-pool-0db1a45d-5tcb  europe-west1-c  n1-standard-2               10.132.0.3   35.189.236.109  RUNNING
```

Dans la cas où nous aurions à gérer plusieurs clusters nous pouvons lister les contexts avec la commande suivante

```
$ kubectl config get-contexts
```

Il est aussi possible de changer de contexte avec la commande suivante

```
$ kubectl config use-context <context-name>
```

**Note:** Une UI permettant de gérer le cluster kubernetes est mise à disposition mais nous ne l’utiliserons pas dans la première partie de l’exercice.

## Namespace personnel

Chaque participant devra créer son propre *namespace* qui sera utilisé tout au long de ce Hands-on. Les namespaces permettent de gérer plusieurs environnements au sein d’un même cluster kubernetes.

Exemple avec un namespace “foo” le namespace avec la commande suivante:

```
$ kubectl create namespace foo
$ export NAMESPACE=foo
```

**Attention:** ne pas oublier d’initialiser la variable d'environnement NAMESPACE 

Lister les namespaces

```
$ kubectl get ns
NAME        STATUS AGE
default     Active 1h
foo         Active 4s
kube-system Active 1h
```

Configurer le namespace pour le reste du Hands-on. Cela évitera de devoir ajouter systématiquement `--namespace=foo`

```
$ kubectl config set-context $(kubectl config current-context) --namespace=${NAMESPACE}
```

## Base de données

### Persistent disk

Les données mysql seront écrites sur un disque persistant ([persistent disk](https://cloud.google.com/compute/docs/disks/)) de GCP. Nous allons donc créer un *Persistent Volume Claim* (pvc).

```
$ kubectl create -f database/mysql-pvc.yaml
persistentvolumeclaim "mysql-persistent-disk" created

$ kubectl get pvc
NAME                    STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
mysql-persistent-disk   Bound     pvc-1c84b1ee-56b5-11e7-8a8a-42010a8400e1   10Gi       RWO           2m
```

### Passwords

Le mot de passe sera stocké dans un *secret*. Les secrets permettent de stocker les informations sensibles.

```
$ kubectl create secret generic mysql-passwords --from-literal=root=myRootS3cr3t --from-literal=ghost=myS3cr3t
secret "mysql-passwords" created

$ kubectl describe secret mysql-passwords
Name:         mysql-passwords
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:	Opaque

Data
====
ghost:	8 bytes
root:	12 bytes
```

Ces mots de passe seront utilisés ci-après

### Deployment

Consulter le contenu du fichier database/mysql-deployment.yaml et analyser la façon dont est utilisé le pvc. Créer le *deployment* avec la commande suivante :

```
$ kubectl apply -f database/mysql-deployment.yaml
```

Visualiser l’état du pod:

```
$ kubectl get pods --all-namespaces -o wide -l name=mysql
NAMESPACE   NAME                     READY     STATUS    RESTARTS   AGE       IP         NODE
default     mysql-2242316436-lwqf5   1/1       Running   0          6m        10.4.0.7   gke-cncp-default-pool-0db1a45d-5tcb
```

Cette commande permet de:
- lister les pods présents dans tous les namespaces `--all-namespaces`
- visualiser sur quels noeuds sont démarrés les pods `-o wide`
- filtrer sur les pods ayant un label *name* avec la valeur *mysql* `-l name=mysql`

### Service

Créer le service permettant d’accéder au serveur de base de données

```
$ kubectl create -f database/mysql-service.yaml
```

Lorsqu’un service est créé kubernetes attribue une adresse ip à ce service afin de pouvoir y accéder :

```
$ kubectl get svc mysql
NAME      CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
mysql     10.7.247.214   <none>        3306/TCP   2m
```

On peut alors se connecter au service pour vérifier l’existence de la base de données `ghost`. 

Les paramètres de connexion ont été configurés dans le fichier `database/mysql-deployment.yaml`.

Créer un *deployment* en mode interactif qui permettra d’utiliser le client mysql à partir d’un pod.

```
$ kubectl run -ti mysqlcli --image mysql --command /bin/bash
Waiting for pod default/foo-1498585132-lfpv3 to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.
root@foo-1498585132-lfpv3:/# 
```

On peut alors exécuter le client mysql en indiquant le nom de domaine de service mysql.

**Note:** le *service discovery* se fait principalement en utilisant un dns, il s’agit d’un composant installé par défaut sur les clusters gke. Un service est accessible en utilisant la nomenclature suivante: <service_name>.<namespace>.svc.<cluster_domain_name>

```
$ root@foo-1498585132-lfpv3:/# mysql -h mysql.foo.svc.cluster.local -u ghost -p
…
mysql> show databases;
| ghost              |
```

On va enfin supprimer ce deployment provisoire

```
$ kubectl delete deploy mysqlcli
deployment "mysqlcli" deleted
```

De la même manière, il est possible de se connecter à un container (ou simplement d’exécuter une commande) avec l’argument exec. ex:

```
$ kubectl exec -ti <pod-name> [--namespace <namespace>] [-c <container-name>] -- /bin/bash
```

## Ghost

### Service

Créer le service ghost.

```
$ kubectl create -f ghost/ghost-service.yaml
```

Au bout de quelques secondes un loadbalancer Google exposera le service ghost. 

**Note:** il faut bien attendre que le loadbalancer soit créé avant de passer à l’étape suivante.

Exécuter la commande suivante qui permet de voir les modifications du service en temps réel.

```
$ kubectl get svc ghost -w
```

Il est alors possible de récupérer l’adresse ip de la façon suivante:

```
$ export GHOST_IP=$(kubectl get svc ghost --template="{{range .status.loadBalancer.ingress}}{{.ip}}{{end}}")
```

### Secret

Étant donné que le fichier de configuration de ghost contient des informations sensibles (paramètres de connexions à la base données), nous allons créer un secret.

**Il faut au préalable éditer le fichier `configs/config.js`** et y indiquer l’adresse ip du service ghost (loadbalancer Google) et le namespace.

```
$ sed -i "s/GHOST_LOADBALANCER_IP/${GHOST_IP}/g" configs/config.js
$ sed -i "s/NAMESPACE/${NAMESPACE}/g" configs/config.js
```

```
$ kubectl create secret generic ghost --from-file=configs/config.js
secret "ghost" created
```

### ConfigMap

Le fichier de configuration nginx pour ghost est très simple et peut donc être stocké dans un *ConfigMap*.

```
$ kubectl create configmap nginx-ghost --from-file=configs/ghost.conf 
configmap "nginx-ghost" created
```

### Deployment

Créer le deployment ghost

```
$ kubectl create -f ghost/ghost-deployment.yaml
```

On peut y accéder via un navigateur web : `http://${GHOST_IP}`


### Scaling

Afin  de modifier le nombre d’instances (replicas) de ghost vous pouvez:

- Utiliser la commande suivante:

```
$ kubectl scale deploy ghost --replicas=3
```

- Editer le fichier ghost/ghost-deployment.yaml et changer la valeur de replicas à 3

```
$ kubectl apply -f ghost/ghost-deployment.yaml
```

- Editer directement la définition du deployment sur kubernetes

```
$ kubectl edit deploy ghost
```

- Configurer l’autoscaling qui permet de modifier le nombre de replicas en fonction de la charge cpu par exemple

```
$ kubectl autoscale deployment ghost --min=2 --max=5 --cpu-percent=80
```

On peut visualiser la progression de la création des ressources avec la commande suivante :

```
$ kubectl get events -w
```

### Rolling update

Modifier la version de l’image docker afin de provoquer un [rolling-update](http://kubernetes.io/docs/user-guide/rolling-updates/)

```
$ kubectl set image deployment/ghost ghost=kelseyhightower/ghost:0.7.8
$ kubectl get rs -w
```

Cela provoquera la création d’un nouveau *replicaSet*, les replicas de l’ancien *replicaSet* seront arrêtés progressivement pendant que les nouveaux seront démarrés.

```
$ kubectl get rs -l app=ghost -w
NAME              DESIRED   CURRENT   READY     AGE
ghost-559661548   0         0         0         43m
ghost-651739629   3         3         3         27m
```

### Rolling back

Afin d’utiliser la version précédente du deployment ghost, utiliser la commande suivante :

```
$ kubectl rollout undo deployment/ghost
```

Pour information il est aussi possible de choisir une version spécifique du deployment en indiquant le numéro de la “revision”. se référer à la documentation.


## Troubleshooting

Lorsque le déploiement d’une nouvelle application se passe mal, il y a plusieurs éléments qui permettent d’identifier la cause.

L’erreur la plus fréquente est *CrashLoopBackOff* qui indique que le pod n’a pas réussi à démarrer correctement et redémarre systématiquement.

Il faut donc être très attentif au nombre de RESTARTS

```
$ kubectl get pods -w
NAME                    READY     STATUS    RESTARTS   AGE
ghost-651739629-00h4j   2/2       Running   0          34m
mysql-266385602-1zdqu   1/1       Running   0          1h
```

Plus généralement ci-dessous on trouvera les éléments à consulter :

### Logs

Voici un exemple de commande permettant de visualiser les logs du container nginx lancé par le pod `ghost-651739629-00h4j`.

```
$ kubectl logs ghost-651739629-00h4j -c nginx --tail=5 -f
```

Les logs ne sont parfois pas suffisants pour identifier le problème (Ils sont en effet générés par l’application au lancement du container). D’autres infos permettent de constater un problème de scheduling ou un point de montage inaccessible par exemple.

### Describe

```
$ kubectl describe deploy ghost
Name:                   ghost
Namespace:              foo
CreationTimestamp:      Wed, 23 Nov 2016 09:39:24 +0100
Labels:                 app=ghost
                        track=stable
Selector:               app=ghost,track=stable
Replicas:               1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:         <none>
NewReplicaSet:          ghost-651739629 (1/1 replicas created)
Events:
  FirstSeen     LastSeen        Count   From                            SubobjectPath   Type            Reason                  Message
  ---------     --------        -----   ----                            -------------   --------        ------                  -------
  47m           47m             1       {deployment-controller }                        Normal          ScalingReplicaSet       Scaled up replica set ghost-651739629 to 1
  47m           47m             1       {deployment-controller }                        Normal          ScalingReplicaSet       Scaled down replica set ghost-559661548 to 0
```

### Events

Observer tous les événements qui se déroulent dans le namespace courant

```
$ kubectl get events -w
```

### Exec

Il peut s’avérer utile d’exécuter une commande à partir d’un container.

```
$ kubectl exec -ti ghost-651739629-00h4j -- /bin/bash
root@ghost-651739629-00h4j:/# apt-get update && apt-get install -y telnet
root@ghost-651739629-00h4j:/# telnet mysql.foo.svc.cluster.local 3306
Trying 10.71.245.46...
Connected to mysql.foo.svc.cluster.local.
Escape character is '^]'.
...
```

### Ressources utilisées

Pour consulter les ressources utilisées par les pods/nodes.

```
$ kubectl top pod -l app=ghost
```

## Nettoyage

La suppression du cluster se fait simplement avec la commande suivante

```
$ gcloud container clusters delete cncp --zone europe-west1-b
```
