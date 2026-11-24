# Réponses — TP Migrer StayBook vers Kubernetes avec Argo CD

**Nom(s) :** Martin Mixte  
**Dépôt GitHub :** https://github.com/zebaxter/staybook-gitops  


## Étape 1 — Docker Compose : le point de départ

### Question 1

_Dans ce `docker-compose.yml`, repérez tout ce que Compose fait **pour vous** et que vous n'aurez plus « gratuitement » dans Kubernetes : l'ordre de démarrage, le réseau, les volumes, les ports, les variables d'environnement, la construction des images. Pour chacun, notez le mot-clé Compose correspondant._

Réponse :

Ce que Compose fait pour nous dans `docker-compose.yml`, et que Kubernetes ne fera pas tout seul :

| Ce que fait Compose | Mot-clé Compose |
|---|---|
| Ordre de démarrage | `depends_on` (forme courte : liste de services) |
| Attendre qu'un service soit **prêt**, pas seulement démarré | `depends_on` + `condition: service_healthy`, avec un `healthcheck` (`pg_isready`) |
| Réseau privé et DNS entre conteneurs (on s'appelle par le nom du service : `postgres-hotels`, `hotels-service:3001`…) | `networks: [tp]` + le bloc `networks:` (`driver: bridge`) |
| Persistance des données (bases, Qdrant, modèles Ollama) | `volumes:` (volumes nommés `hotels-data`, `ollama-data`…) et bind-mount des scripts SQL d'init |
| Exposer le site sur la machine | `ports: - "8080:80"` (seul le `front` est exposé) |
| Variables d'environnement | `environment:`, plus `${EMBED_MODEL}` / `${LLM_MODEL}` lus dans le fichier `.env` |
| Construction des images depuis notre code | `build: ./hotels-service` (etc.) ; `image:` pour les images toutes faites |

Vérification : `docker compose up -d --build` lance les 9 conteneurs. http://localhost:8080 affiche les 6 hôtels, et le concierge répond (« Je cherche un hotel a Paris pour 2 personnes ») une fois les modèles `nomic-embed-text` et `llama3.2:1b` téléchargés et les 6 hôtels indexés.


### Question 2

_Pourquoi faut-il impérativement faire `docker compose down` avant de continuer ? (Deux raisons : l'une concerne un port, l'autre la mémoire.)_

Réponse :

1. **Le port** : le front Compose publie `8080:80` sur la machine. Or le cluster kind utilise aussi le port **8080** de la machine (redirigé vers le NodePort 30080 d'Argo CD). Deux programmes ne peuvent pas écouter sur le même port : la création du cluster échouerait (`address already in use`), ou Argo CD serait inaccessible. Pour le TP, j'ai d'ailleurs dû arrêter le nœud kind (`docker stop argocd-tp-control-plane`) pour pouvoir relancer Compose.
2. **La mémoire** : les 9 conteneurs, surtout **Ollama** avec son modèle Llama chargé en RAM (plus Qdrant et 2 PostgreSQL), consomment plusieurs Go. Le cluster kind va relancer **les mêmes services** (dont un deuxième Ollama), plus Argo CD et Kubernetes lui-même. Avec les ~8 Go alloués à Docker, les deux ensemble saturent la mémoire : pods `OOMKilled` ou `Pending`.

`docker compose down` supprime les conteneurs et le réseau mais **garde les images** (`docker images | grep staybook` les liste toujours) et les volumes nommés (sans `-v`).


## Étape 2 — Kubernetes + Argo CD : la migration

### Question 3

_En mode push, qui détient les droits d'administration sur le cluster ? Citez deux risques concrets que le modèle pull réduit._

Réponse :

En mode **push**, c'est l'outil de déploiement (le pipeline CI : GitLab CI, GitHub Actions, Jenkins), et donc toute personne ou tout script qui contrôle ce pipeline, qui détient un **kubeconfig administrateur** du cluster, stocké comme secret dans la CI.

Deux risques concrets que le modèle **pull** réduit :
1. **Fuite des identifiants** : si la CI est compromise (dépendance malveillante, secret affiché dans un log, runner partagé), l'attaquant obtient un accès admin au cluster de production. En pull, la CI n'a aucun accès au cluster : c'est l'agent (Argo CD), **dans** le cluster, qui va lire Git. Il n'y a aucun port à ouvrir vers le cluster ni de credentials cluster à l'extérieur.
2. **Dérive non détectée** : en push, on applique une fois puis plus personne ne compare. Une modification manuelle (`kubectl edit`) ou un déploiement raté reste invisible. En pull, l'agent compare **en continu** l'état réel à Git et corrige ou signale l'écart. On a aussi une seule porte d'entrée, Git, donc tout changement est tracé (qui, quoi, quand) et réversible (`git revert`).


### Question 4

_Un collègue modifie « à la main » le nombre de réplicas d'un déploiement avec `kubectl scale` sur un cluster géré par Argo CD. D'après le principe 4, que va-t-il se passer ? Est-ce souhaitable ?_

Réponse :

D'après le principe 4 (**réconcilié en continu**), Argo CD compare en permanence le cluster à Git. Le `kubectl scale` crée une **dérive** : l'Application passe `OutOfSync`. Si le **self-heal** est activé, Argo CD remet presque immédiatement le nombre de réplicas écrit dans Git, et la modification du collègue est annulée. Sans self-heal, l'application reste `OutOfSync` jusqu'au prochain sync, qui l'écrasera.

C'est **souhaitable** : Git reste la seule source de vérité. Une modification non tracée, non relue et non reproductible ne doit pas survivre. Si le collègue a vraiment besoin de 3 réplicas, il doit faire un commit (et une revue). En cas d'urgence, on peut suspendre l'auto-sync, mais c'est une décision explicite, pas un contournement silencieux. (Constaté en pratique à la question 29.)


### Question 5

_Quel composant est le seul à parler à GitHub ? Lequel est le seul à créer des ressources dans le cluster ?_

Réponse :

| Composant | Rôle (une phrase) | Deployment ou StatefulSet ? |
|---|---|---|
| argocd-server | API (gRPC/REST) et interface web ; c'est à lui que parlent le navigateur et la CLI `argocd`. | Deployment |
| argocd-repo-server | Clone les dépôts Git et génère les manifests (YAML brut, Kustomize, Helm) ; garde un cache. | Deployment |
| argocd-application-controller | Compare en continu l'état voulu (manifests générés) à l'état réel du cluster et synchronise : applique, prune, self-heal. | **StatefulSet** |
| argocd-redis | Cache partagé (manifests générés, état de l'application) entre les composants. | Deployment |
| argocd-dex-server | Fournisseur d'identité (OIDC) pour le SSO : connexion via GitHub, LDAP, SAML… | Deployment |
| argocd-applicationset-controller | Génère automatiquement des Applications à partir de modèles (ApplicationSet : liste, dossiers Git, clusters…). | Deployment |
| argocd-notifications-controller | Envoie des notifications (Slack, e-mail, webhook…) sur les événements des Applications. | Deployment |

- Le seul composant qui **parle à GitHub** est **`argocd-repo-server`** : il clone et lit les dépôts.
- Le seul qui **crée des ressources dans le cluster** est **`argocd-application-controller`** : il applique les manifests que lui fournit le repo-server. L'`argocd-server` ne fait que transmettre les demandes, par exemple un sync manuel.


### Question 6

_Le cluster est « vierge ». Quels namespaces existent déjà, et à quoi servent-ils ? Combien de pods tournent, et pourquoi n'y en a-t-il aucun dans `default` ?_

Réponse :

Le contexte est `kind-argocd-tp` et le nœud `argocd-tp-control-plane` est `Ready`. Namespaces présents dans le cluster vierge :

| Namespace | Rôle |
|---|---|
| `default` | namespace utilisé quand on n'en précise aucun ; vide par convention |
| `kube-system` | les composants internes de Kubernetes : plan de contrôle, DNS, réseau |
| `kube-public` | lisible par tous, même sans authentification (ConfigMap `cluster-info`, utilisée pour rejoindre le cluster) |
| `kube-node-lease` | un objet `Lease` par nœud : le « battement de cœur » qui indique que le nœud est vivant |
| `local-path-storage` | spécifique à kind : le provisionneur de volumes `local-path`, qui crée les PersistentVolumes sur le disque du nœud |

**9 pods** tournent :
- 8 dans `kube-system` : 2 × `coredns`, `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `kindnet` (réseau des pods) ;
- 1 dans `local-path-storage` : `local-path-provisioner`.

Il n'y en a **aucun dans `default`** : les composants système sont volontairement rangés dans leurs propres namespaces, pour les isoler et les protéger. `default` est laissé aux utilisateurs, et nous n'avons encore rien déployé.

(Dans mon cluster, le namespace `argocd` apparaît aussi : il a été créé plus tard, à l'installation d'Argo CD en partie 4.)


### Question 7

_Pourquoi le script tague les images `v1` plutôt que `latest` ? Cherchez la règle de Kubernetes sur `imagePullPolicy` quand le tag est `latest` : que se passerait-il au démarrage d'un pod ?_

Réponse :

Le script tague `v1` parce qu'un tag explicite est **immuable et traçable** : on sait exactement quelle version tourne, et changer de version veut dire changer une ligne dans Git (`v1` → `v2`). Argo CD voit alors la différence et redéploie. Avec `latest`, le manifest ne change jamais, même quand l'image change : impossible de savoir ce qui tourne, impossible de revenir en arrière proprement.

Règle de Kubernetes : si le tag est `latest` (ou absent), l'`imagePullPolicy` par défaut est **`Always`**. Pour un autre tag, c'est `IfNotPresent`. Avec `latest`, au démarrage d'un pod, le kubelet essaierait donc de **tirer l'image depuis Docker Hub**, où `staybook-front:latest` n'existe pas. On obtiendrait `ErrImagePull` puis `ImagePullBackOff`, alors que l'image est pourtant présente dans le nœud grâce à `kind load`. Avec `v1`, la politique est `IfNotPresent` : le kubelet trouve l'image en local et ne contacte aucun registre.


### Question 8

_Où sont physiquement les images maintenant ? Vérifiez avec `docker exec argocd-tp-control-plane crictl images | grep staybook`. Qu'est-ce que `crictl`, et pourquoi n'est-ce pas `docker images` ?_

Réponse :

Les images sont **dans le conteneur Docker qui joue le rôle de nœud kind** (`argocd-tp-control-plane`), plus précisément dans le stockage d'images de **containerd**, le runtime de conteneurs du nœud. `docker exec argocd-tp-control-plane crictl images | grep staybook` liste bien les 5 images `docker.io/library/staybook-*:v1`.

`crictl` est la CLI de débogage de la **CRI** (Container Runtime Interface), l'interface standard entre le kubelet et le runtime de conteneurs. Ce n'est pas `docker images` parce que **Kubernetes n'utilise pas Docker** dans le nœud (le support de dockershim a été retiré en 1.24) : le kubelet parle à containerd via la CRI. `docker images` sur ma machine montre le stockage du démon Docker de l'hôte, qui est un autre stockage. C'est pour ça qu'il faut `kind load docker-image` pour copier les images de l'un vers l'autre.


### Question 9

_Combien d'objets vont être créés ? Parmi les types listés, trois ne sont pas des objets Kubernetes « standard ». Lesquels, et à quoi servent-ils ? (Indice : `CustomResourceDefinition`.)_

Réponse :

`kubectl kustomize argocd-install | grep -c '^kind:'` donne **60 objets** : 7 ConfigMap, 6 Deployment, 1 StatefulSet, 8 Service, 7 ServiceAccount, 6 Role, 6 RoleBinding, 3 ClusterRole, 3 ClusterRoleBinding, 7 NetworkPolicy, 2 Secret, 1 Namespace et **3 CustomResourceDefinition**.

Les trois objets « non standard » sont les types que définissent ces **3 CRD**. Une CRD étend l'API Kubernetes avec un nouveau type d'objet, que le contrôleur d'Argo CD sait interpréter :
- **`Application`** (`applications.argoproj.io`) : relie une source Git (dépôt, chemin, révision) à une destination (cluster, namespace), avec une politique de sync ;
- **`AppProject`** (`appprojects.argoproj.io`) : regroupe des Applications et limite ce qu'elles ont le droit de faire (dépôts sources, destinations, types de ressources autorisés) ;
- **`ApplicationSet`** (`applicationsets.argoproj.io`) : un modèle qui génère automatiquement plusieurs Applications (par liste, par dossier Git, par cluster…).


### Question 10

_Pourquoi est-il préférable de passer par Kustomize plutôt que de faire `kubectl apply -f install.yaml` puis `kubectl patch` sur le Service ? Pensez au jour où vous mettrez Argo CD à jour._

Réponse :

Avec Kustomize, le fichier officiel `install.yaml` est utilisé **tel quel**, et nos modifications sont décrites **à côté**, sous forme de patchs **versionnés dans Git**. Le jour d'une mise à jour, on change seulement l'URL ou la version de la ressource dans `kustomization.yaml` : Kustomize réapplique automatiquement nos patchs sur la nouvelle version. Tout est déclaratif, reproductible et relisible.

Avec `kubectl apply -f install.yaml` puis `kubectl patch`, le patch est une **action manuelle**, qui n'existe nulle part ailleurs que dans le cluster et dans la mémoire de celui qui l'a tapé. À la mise à jour, le nouvel `install.yaml` **écrase** le Service : on repasse en `ClusterIP` et en HTTPS, et l'interface devient inaccessible tant que quelqu'un ne se souvient pas de refaire les patchs. Un collègue qui monte un cluster identique n'a aucun moyen de savoir quels patchs ont été faits.


### Question 11

_Vérifiez votre tableau de la partie 1 : quel composant est un StatefulSet ? Avez-vous les sept composants ?_

Réponse :

`kubectl get deploy,sts -n argocd` affiche 6 Deployments (`argocd-server`, `argocd-repo-server`, `argocd-redis`, `argocd-dex-server`, `argocd-applicationset-controller`, `argocd-notifications-controller`) et **1 StatefulSet : `argocd-application-controller`**. On a bien les **sept composants** du tableau de la question 5. `kubectl get crd | grep argoproj` liste les 3 CRD.

C'est un StatefulSet parce que le contrôleur peut être réparti en *shards*, chacun gérant une partie des clusters : chaque réplica a besoin d'une **identité stable** (`argocd-application-controller-0`, `-1`…) pour savoir de quels clusters il est responsable.


### Question 12

_Essayez `kubectl apply -k argocd-install` **sans** `--server-side` (c'est sans danger, tout existe déjà). Quelle erreur obtenez-vous, sur quel objet ? Cherchez la cause (`last-applied-configuration`)._

Réponse :

Sans `--server-side`, on obtient, pour chaque objet, un avertissement `missing the kubectl.kubernetes.io/last-applied-configuration annotation…`, puis cette **erreur** :

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```

L'objet en cause est la **CRD `applicationsets.argoproj.io`**.

Cause : un `kubectl apply` classique (*client-side*) enregistre une copie **complète** de l'objet appliqué dans l'annotation `kubectl.kubernetes.io/last-applied-configuration`, pour calculer le prochain diff. Or la CRD ApplicationSet contient un schéma OpenAPI énorme, et sa copie dépasse la **limite de 256 Ko** imposée au total des annotations d'un objet. Avec `--server-side`, c'est l'API server qui calcule la fusion et suit qui possède quel champ (`managedFields`), sans annotation : la limite ne s'applique plus.


### Question 13

_Pourquoi la commande se termine-t-elle par `| base64 -d` ? Que verriez-vous sans ? Un Secret Kubernetes est-il chiffré ?_

Réponse :

Les données d'un Secret (`.data`) sont stockées **encodées en base64**. `jsonpath="{.data.password}"` renvoie donc la valeur encodée, et `base64 -d` la décode pour obtenir le mot de passe lisible. Sans `| base64 -d`, on verrait une chaîne base64 de 24 caractères, inutilisable telle quelle pour se connecter.

**Un Secret n'est pas chiffré** : base64 est un *encodage* réversible, pas un chiffrement. N'importe qui ayant le droit de lire le Secret (`get secret`) lit le mot de passe. Par défaut, il est aussi stocké en clair dans etcd. Il faut activer le chiffrement au repos d'etcd (`EncryptionConfiguration`) et restreindre l'accès aux Secrets par RBAC pour qu'il soit réellement protégé.


### Question 14

_Que signifient `--plaintext` et `--grpc-web` ? Que se passe-t-il sans `--plaintext` (essayez) ? Ces options seraient-elles acceptables en production ?_

Réponse :

- **`--plaintext`** : la CLI parle au serveur en **HTTP, sans TLS**. C'est nécessaire ici parce que nous avons patché `server.insecure: true` : l'argocd-server n'écoute qu'en HTTP sur le NodePort 30080.
- **`--grpc-web`** : la CLI utilise **gRPC-Web**, c'est-à-dire du gRPC encapsulé dans de simples requêtes HTTP/1.1, au lieu du gRPC natif en HTTP/2. C'est utile quand un intermédiaire (proxy, ingress, ici la redirection de port de kind) ne transmet pas correctement le HTTP/2.

Sans `--plaintext`, la CLI tente une connexion TLS. Elle affiche `WARNING: server is not configured with TLS. Proceed (y/n)?`, puis la connexion échoue (`fatal … EOF`) : le serveur ne parle pas TLS.

En production, **non** pour `--plaintext` : le mot de passe et le jeton de session circuleraient en clair sur le réseau. On garde TLS, avec un vrai certificat sur l'Ingress, par exemple via cert-manager. `--grpc-web` est acceptable : c'est seulement un mode de transport, compatible avec TLS.


### Question 15

_Regardez `apps/staybook/00-secrets.yaml` : les mots de passe PostgreSQL sont en clair dans Git. Est-ce acceptable ici ? Et en entreprise ? Citez une solution._

Réponse :

Ici, en TP, c'est **acceptable** : les mots de passe (`hotels`/`hotels`…) sont factices, la base est locale et jetable, et le dépôt était prévu privé.

En entreprise, **non** : tout ce qui est commité reste à jamais dans l'historique Git. Tous ceux qui ont accès au dépôt (développeurs, prestataires, CI, outils tiers, forks, sauvegardes) liraient les mots de passe de production. Et changer un mot de passe ne l'efface pas des anciens commits.

Solutions : **Sealed Secrets** (le Secret est chiffré avec une clé publique, et seul le contrôleur du cluster peut le déchiffrer), **SOPS** (avec age ou KMS), ou un coffre externe (**HashiCorp Vault**, AWS Secrets Manager…) synchronisé dans le cluster par **External Secrets Operator**. Git ne contient alors qu'une référence ou une version chiffrée.


### Question 16

_Notez l'erreur. Pourquoi GitHub répond-il « Repository not found » et non « accès refusé » ?_

Réponse :

Ce test devait se faire sur le dépôt privé, sans identifiants. Mon dépôt `staybook-gitops` étant **public**, je l'ai reproduit sur un dépôt qui n'est pas lisible sans identifiants :

```
error testing repository connectivity: unable to ls-remote HEAD on repository: failed to list refs: authentication required: Repository not found.
```

GitHub répond « **Repository not found** » (404) et non « accès refusé » (403) pour **ne pas révéler l'existence** d'un dépôt privé. S'il répondait « accès refusé », un attaquant pourrait deviner les noms des dépôts privés d'une organisation (énumération). Pour un utilisateur non autorisé, un dépôt privé et un dépôt inexistant doivent donc être indiscernables.


### Question 17

_Sous quelle forme Argo CD a-t-il stocké ces identifiants dans le cluster ? Trouvez l'objet avec `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository` et listez ses clés (`-o jsonpath='{.data}'` puis décodez `url` et `username`, pas `password`)._

Réponse :

Argo CD a stocké les identifiants dans un **Secret Kubernetes** du namespace `argocd`, nommé `repo-312481070`, de type `Opaque`. Il porte le label `argocd.argoproj.io/secret-type=repository`, et c'est grâce à ce label qu'Argo CD le reconnaît comme une déclaration de dépôt.

Ses clés (`-o jsonpath='{.data}'`) sont : `password`, `type`, `url`, `username`. Une fois décodées :
- `url` = `https://github.com/zebaxter/staybook-gitops.git`
- `username` = `zebaxter`
- `type` = `git`
- `password` (non décodé) = le token GitHub, seulement encodé en base64, donc **lisible par quiconque peut lire les Secrets de `argocd`** (voir question 13).

Il n'y a pas de base de données propre à Argo CD : sa configuration est faite d'objets Kubernetes (Secrets, ConfigMaps, Applications). On pourrait donc aussi déclarer ce dépôt en YAML.


### Question 18

_Comparez trois façons de s'authentifier (token personnel, deploy key SSH, GitHub App) : portée, rotation, ce qui se passe quand la personne quitte l'entreprise. Laquelle recommanderiez-vous pour 200 dépôts ?_

Réponse :

| | Token personnel (PAT) | Deploy key SSH | GitHub App |
|---|---|---|---|
| **Portée** | les droits **de la personne**, sur les dépôts choisis (fine-grained) ou sur tous ses dépôts (classic) | **un seul dépôt**, en lecture seule (ou écriture si cochée) | les dépôts et les permissions fines choisis à l'installation sur l'organisation |
| **Rotation** | manuelle, avec une date d'expiration à surveiller | manuelle (nouvelle paire de clés), pas d'expiration | automatique : jetons d'installation de courte durée (1 h) générés à partir de la clé privée de l'App |
| **Départ de la personne** | **tout casse** : le token est lié à son compte, qui est désactivé ; et pendant ce temps, l'accès reste celui d'une personne | rien ne casse : la clé est liée au dépôt, pas à une personne | rien ne casse : l'App appartient à l'**organisation** |

Pour **200 dépôts**, je recommande une **GitHub App** : une seule installation pour toute l'organisation (au lieu de 200 deploy keys à gérer une par une), des permissions fines (`Contents: read`), des jetons éphémères renouvelés automatiquement, l'indépendance vis-à-vis des personnes, et une trace d'audit au nom de l'App. Argo CD la supporte nativement (`githubAppID`, `githubAppInstallationID`, `githubAppPrivateKey`), y compris en modèle d'identifiants partagé par préfixe d'URL (*credential template*).


### Question 19

_Combien de ressources Argo CD compte-t-il déployer ? Retrouvez chacune dans les fichiers de `apps/staybook/`._

Réponse :

Juste après la création, Argo CD affiche **16 ressources**, toutes en `OutOfSync` / `Missing` :

| Fichier de `apps/staybook/` | Ressources |
|---|---|
| `00-secrets.yaml` | Secret `postgres-hotels`, Secret `postgres-reservations` |
| `10-postgres-hotels.yaml` | ConfigMap `postgres-hotels-init`, Service `postgres-hotels`, StatefulSet `postgres-hotels` |
| `11-postgres-reservations.yaml` | ConfigMap `postgres-reservations-init`, Service `postgres-reservations`, StatefulSet `postgres-reservations` |
| `20-hotels-service.yaml` | Service + Deployment `hotels-service` |
| `21-reservations-service.yaml` | Service + Deployment `reservations-service` |
| `30-gateway.yaml` | Service + Deployment `gateway` |
| `40-front.yaml` | Service (NodePort 30081) + Deployment `front` |

Au total : 2 Secrets, 2 ConfigMaps, 6 Services, 2 StatefulSets et 4 Deployments. Le Namespace `staybook` n'est pas dans le décompte : il est créé à part par l'option `CreateNamespace=true`.


### Question 20

_Les ressources ne sont pas déployées toutes en même temps. Ouvrez les manifests et trouvez l'annotation qui pilote cet ordre. Quel ordre a été choisi, et pourquoi ? À quoi cela correspondait-il dans `docker-compose.yml` ?_

Réponse :

L'ordre est piloté par l'annotation **`argocd.argoproj.io/sync-wave`**. Argo CD applique les vagues par ordre croissant et **attend que chaque vague soit `Healthy`** avant de passer à la suivante :

| Vague | Ressources | Pourquoi |
|---|---|---|
| `0` | Secrets, ConfigMaps d'init, Services et StatefulSets PostgreSQL | les bases doivent exister et être prêtes en premier |
| `1` | `hotels-service`, `reservations-service` | ils se connectent aux bases |
| `2` | `gateway` | elle appelle les deux services |
| `3` | `front` | il appelle la gateway, et il est la porte d'entrée du site |

Observé pendant le sync : vague 0 à 16:50:30, puis rien pendant **30 s** (le temps que les StatefulSets passent `Healthy`), puis vague 1 à 16:51:03, puis gateway et front quelques secondes après.

C'est l'équivalent des **`depends_on`** de `docker-compose.yml`, avec `condition: service_healthy` sur les bases (Argo CD attend l'état `Healthy`, et Compose attend le `healthcheck`). L'ordre de dépendance est le même : bases → services métier → gateway → front.


### Question 21

_« OutOfSync » et « Healthy » sont deux statuts indépendants. Donnez un exemple d'application `Synced` mais `Degraded`, et un exemple `OutOfSync` mais `Healthy`._

Réponse :

Les deux statuts répondent à deux questions différentes. **Sync** : est-ce que le cluster correspond à Git ? **Health** : est-ce que ça fonctionne ?

- **`Synced` mais `Degraded`** : Git dit `image: staybook-front:v3` et c'est bien ce qui est appliqué dans le cluster (donc `Synced`), mais l'image n'existe pas : le pod est en `ImagePullBackOff`, et le Deployment est `Degraded` (vécu à la question 33). Même cas avec un mauvais mot de passe de base commité : le service démarre en `CrashLoopBackOff`.
- **`OutOfSync` mais `Healthy`** : un collègue fait `kubectl scale deploy front --replicas=3` (et le self-heal est désactivé). Le cluster ne correspond plus à Git (`OutOfSync`), mais les 3 pods tournent parfaitement (`Healthy`). Autre cas : on vient de pousser un commit qui n'est pas encore synchronisé, et l'ancienne version tourne toujours très bien.


### Question 22

_Vous venez de faire un `kubectl apply`… n'est-ce pas contraire au GitOps ? Pourquoi ce dernier `apply` est-il inévitable, et comment réduire cette « amorce » à un seul objet pour tout un cluster ? (Cherchez « App of Apps ».)_

Réponse :

Oui, en apparence c'est contraire au GitOps : on agit directement sur le cluster. Mais cet `apply` est **inévitable**, c'est l'**amorce** (*bootstrap*). Argo CD ne surveille que les dépôts et les chemins que lui indiquent des objets `Application`. L'objet `Application` lui-même doit donc exister avant qu'Argo CD puisse lire quoi que ce soit : il ne peut pas se créer lui-même depuis Git. C'est le problème de l'œuf et de la poule. On constate d'ailleurs en partie 8.2 qu'une modification de `argocd/staybook-application.yaml` poussée dans Git n'a **aucun effet** tant qu'on ne la réapplique pas.

Pour réduire l'amorce à **un seul objet pour tout le cluster** : le motif **« App of Apps »**. On crée une Application racine (`root`) dont le dossier source (par exemple `argocd/apps/`) contient… les autres Applications (`staybook`, `concierge`, …). On ne fait qu'un `kubectl apply -f root.yaml`. Ensuite, ajouter, modifier ou supprimer une Application se fait par commit, puisque `root` les gère comme n'importe quelle ressource. On peut aussi utiliser un `ApplicationSet`. (C'est ce que met en place le TP 2.)


### Question 23

_Reprenez votre réponse à la question 1 et complétez la colonne de droite en vous appuyant sur les manifests fournis._

Réponse :

| Ce que faisait Compose | Mot-clé Compose | Dans les manifests Kubernetes |
|---|---|---|
| Ordre de démarrage | `depends_on` | annotation Argo CD `argocd.argoproj.io/sync-wave` (0 → 3), et `initContainers` qui attendent leurs dépendances (rag-service) |
| Attendre qu'un service soit prêt | `condition: service_healthy` + `healthcheck` | `readinessProbe` dans les conteneurs ; Argo CD attend l'état `Healthy` d'une vague avant la suivante |
| Réseau et DNS entre services | `networks: [tp]` | un objet **`Service`** (ClusterIP) par composant ; le DNS du cluster (CoreDNS) résout `hotels-service`, `postgres-hotels`… dans le namespace `staybook` |
| Persistance des données | `volumes:` (volumes nommés) | `volumeClaimTemplates` des **StatefulSets** PostgreSQL et Qdrant, **PersistentVolumeClaim** `ollama-models` ; scripts SQL d'init montés depuis une **ConfigMap** |
| Exposer le site | `ports: "8080:80"` | Service `front` de type **NodePort** (30081), redirigé vers `localhost:8081` par `extraPortMappings` de kind |
| Variables d'environnement | `environment:` / `.env` | `env:` dans le conteneur, valeurs en dur ou via `configMapKeyRef` (ConfigMap `concierge-config`) |
| Mots de passe | `environment:` (en clair) | **Secret** (`postgres-hotels`, `postgres-reservations`), lu par PostgreSQL via `envFrom: secretRef` ; mais toujours en clair dans Git (question 15), et répété en clair dans `DB_PASSWORD` des Deployments `hotels-service`/`reservations-service` |
| Image à utiliser | `build:` / `image:` | `image: staybook-front:v1` uniquement : la construction se fait **hors** de Kubernetes (`docker build` + `kind load`, plus tard une CI) |


### Question 24

_Pourquoi les deux Applications pointent-elles vers le **même namespace** `staybook` ? Que se passerait-il si le concierge était dans un namespace `concierge` ? (Indice : regardez `RAG_URL` dans le manifest de la gateway.)_

Réponse :

Parce que les composants s'appellent par leur **nom court**. Dans `apps/staybook/30-gateway.yaml`, `RAG_URL` vaut `http://rag-service:3003`, et le rag-service appelle `http://hotels-service:3001` et `http://reservations-service:3002`. Un nom court comme `rag-service` n'est résolu par le DNS du cluster **que dans le namespace du pod qui fait la requête** : le resolver ajoute `staybook.svc.cluster.local`, d'après le `search` de `/etc/resolv.conf`. Les deux Applications doivent donc déployer dans le **même namespace** `staybook` pour que ces noms fonctionnent, exactement comme le réseau `tp` partagé de Compose.

Si le concierge était dans un namespace `concierge`, la gateway chercherait `rag-service.staybook.svc.cluster.local`, qui n'existe pas. Le DNS renverrait `NXDOMAIN` et le chat tomberait en erreur, alors que le reste du site marcherait. Dans l'autre sens, le rag-service ne trouverait plus `hotels-service` et ne pourrait plus indexer les hôtels. Il faudrait utiliser les noms complets : `http://rag-service.concierge.svc.cluster.local:3003` dans la gateway, et `http://hotels-service.staybook.svc.cluster.local:3001` dans le rag-service. (C'est exactement ce que fait le TP 2.)


### Question 25

_Ouvrez `apps/concierge/20-rag-service.yaml`. Que font les deux `initContainers` ? En quoi est-ce différent des sync waves de la question 20 ? Pourquoi avoir les deux ?_

Réponse :

Les deux `initContainers` (`busybox`) s'exécutent **l'un après l'autre, avant** le conteneur `rag-service`. Chacun boucle jusqu'à ce que sa dépendance réponde :
- `wait-for-ollama` : `until wget -qO- http://ollama:11434/api/tags … ; sleep 3` ;
- `wait-for-qdrant` : `until wget -qO- http://qdrant:6333/readyz … ; sleep 3`.

C'est pourquoi on voit le pod en `Init:0/2`, puis `Init:1/2`, avant `Running`. C'est l'équivalent du `depends_on` de Compose, dont Kubernetes est dépourvu.

La différence avec les **sync waves** :
- les sync waves sont un mécanisme **d'Argo CD**, qui n'agit **qu'au moment d'un sync** : elles décident dans quel ordre les objets sont **envoyés à l'API** Kubernetes ;
- les initContainers sont un mécanisme **de Kubernetes**, qui agit **à chaque démarrage du pod**, quelle qu'en soit la cause : sync, mais aussi redémarrage, pod supprimé, nœud qui redémarre, réplica ajouté. Ils vérifient que la dépendance **répond réellement**.

On a besoin des deux. Les waves donnent un déploiement initial propre et lisible, mais une fois le sync terminé elles ne protègent plus rien : si Ollama redémarre demain, aucune vague n'est rejouée. Les initContainers garantissent qu'à **tout** démarrage, le service attend ses dépendances. À l'inverse, ils ne règlent pas l'ordre de création des objets : Services, ConfigMap, PVC.


### Question 26

_Ouvrez `apps/concierge/11-ollama.yaml`. Où sont stockés les modèles téléchargés ? Supprimez le pod Ollama (`kubectl -n staybook delete pod -l app=ollama`) et vérifiez avec `kubectl -n staybook exec deploy/ollama -- ollama list` que les modèles sont toujours là. Qu'est-ce qui les a conservés ? Que signifie `strategy: Recreate` ?_

Réponse :

Les modèles sont stockés dans **`/root/.ollama`**, monté depuis le volume `models`. Ce volume est le **PersistentVolumeClaim `ollama-models`** (5 Gi, `ReadWriteOnce`, StorageClass `standard` de kind, donc `local-path` sur le disque du nœud).

Test : `ollama list` affichait `llama3.2:1b` (1,3 GB) et `nomic-embed-text` (274 MB). Après `kubectl -n staybook delete pod -l app=ollama`, le Deployment a recréé un pod, et `ollama list` affiche **les mêmes modèles**, avec la même date de modification. Rien n'a été retéléchargé.

C'est le **PVC** qui les a conservés : son cycle de vie est indépendant de celui du pod. Le nouveau pod remonte le même volume. Sans lui (`emptyDir` ou système de fichiers du conteneur), il faudrait retélécharger ~1,6 Go à chaque redémarrage.

*Vécu pendant le TP : j'avais supprimé le pod Ollama pendant que le rag-service téléchargeait encore `llama3.2:1b`. Le téléchargement a été interrompu, et le chat renvoyait `Ollama generate error: 404` (modèle absent). Supprimer le pod rag-service a suffi : à son redémarrage, il a relancé le téléchargement puis l'indexation (`Indexed 6 hotels`).*

**`strategy: Recreate`** : lors d'une mise à jour, Kubernetes **supprime d'abord l'ancien pod, puis crée le nouveau**, au lieu du `RollingUpdate` par défaut, qui démarre le nouveau avant d'arrêter l'ancien. C'est nécessaire parce que le volume est en **`ReadWriteOnce`** : un seul pod peut le monter. En rolling update, le nouveau pod resterait bloqué en attendant un volume tenu par l'ancien. Le prix à payer est une courte coupure du concierge pendant la mise à jour.


### Question 27

_Toujours dans ce fichier, lisez le bloc `resources`. Que se passe-t-il si Ollama dépasse la limite mémoire ? Et si le nœud n'a pas les 2 Gi demandés ?_

Réponse :

Le bloc `resources` d'Ollama : `requests: cpu: "1", memory: 2Gi` et `limits: memory: 6Gi`.

- **Si Ollama dépasse la limite mémoire (6 Gi)** : le noyau tue le processus, `OOMKilled` (Out Of Memory, code de sortie 137). Le kubelet redémarre le conteneur selon la `restartPolicy` (`Always`). Si cela se répète, le pod part en `CrashLoopBackOff`, avec un délai croissant entre les redémarrages. Il n'y a pas de limite CPU : au-delà du CPU demandé, Ollama est seulement ralenti (*throttling*), jamais tué.
- **Si le nœud n'a pas les 2 Gi demandés** : le **scheduler** ne place pas le pod. Il reste en **`Pending`**, avec un événement `FailedScheduling … Insufficient memory`, tant qu'aucun nœud n'a 2 Gi **réservables**. Le calcul se fait sur la somme des *requests* déjà attribuées, pas sur la consommation réelle. Le rag-service, lui, reste bloqué en `Init:0/2`.

Dans mon cluster : le nœud a ~7,6 Gi allouables, et `kubectl describe node` montre **2722 Mi de requests (34 %)** mais **8276 Mi de limits (105 %)**. Le nœud est donc *overcommitted* : si tous les conteneurs montaient à leur limite en même temps, le nœud manquerait de mémoire et des pods seraient tués ou évincés.


## Étape 3 — GitOps : Git pilote le cluster

### Question 28

_Sans `--refresh`, au bout de combien de temps Argo CD aurait-il vu le commit ? D'où vient cette valeur (ConfigMap `argocd-cm`, clé `timeout.reconciliation`) ? Quel mécanisme permettrait de réagir en quelques secondes sans interroger GitHub en boucle, et pourquoi ne peut-on pas l'utiliser ici ?_

Réponse :

Sans `--refresh`, Argo CD aurait vu le commit au plus tard à la **prochaine réconciliation périodique** : par défaut toutes les **120 secondes** (3 minutes au maximum, avec la gigue). Cette valeur vient de la clé **`timeout.reconciliation`** de la ConfigMap **`argocd-cm`**. Dans mon cluster, la clé n'est pas définie (`kubectl -n argocd get cm argocd-cm -o jsonpath='{.data.timeout\.reconciliation}'` ne renvoie rien), donc c'est la valeur par défaut, `120s`, qui s'applique, avec `timeout.reconciliation.jitter` qui étale les polls jusqu'à +60 s. On peut la changer, par exemple `timeout.reconciliation: 30s`, ou `0` pour désactiver le polling.

Pour réagir **en quelques secondes** sans interroger GitHub en boucle : un **webhook**. Dans les paramètres du dépôt GitHub, on déclare un webhook vers `https://<argocd>/api/webhook` (avec un secret partagé). À chaque push, GitHub **appelle Argo CD**, qui rafraîchit immédiatement les Applications concernées.

On ne peut pas l'utiliser ici parce que GitHub doit pouvoir **joindre Argo CD depuis Internet**. Or notre Argo CD tourne sur `localhost:8080`, dans un cluster kind sur notre portable, derrière une box ou un NAT, sans adresse publique. GitHub ne peut pas l'atteindre. Il faudrait l'exposer, avec un tunnel comme ngrok ou cloudflared, ou un cluster hébergé avec un Ingress public.


### Question 29

_Que s'est-il passé, et en combien de temps ? Quel composant a agi ? Retrouvez sa trace : `kubectl -n argocd logs sts/argocd-application-controller | grep staybook | tail`. Essayez plus fort : `kubectl -n staybook delete deploy gateway`. Le site est-il resté accessible ?_

Réponse :

**Le `scale` à 3 réplicas** : à 16:07:56.6, `kubectl -n staybook scale deploy front --replicas=3`. Dans la même seconde, le contrôleur voit la dérive et la corrige. Le Deployment est revenu à `1/1` **en moins d'une seconde** : le pod supplémentaire n'a même pas eu le temps de démarrer. Trace dans `kubectl -n argocd logs sts/argocd-application-controller` :

```
16:07:56Z Updated sync status: Synced -> OutOfSync
16:07:56Z Initiated automated sync to '408328ca…'
16:07:57Z Partial sync operation to 408328ca… succeeded
16:07:57Z Updated sync status: OutOfSync -> Synced
```

Le composant qui a agi est l'**`argocd-application-controller`**. Il ne fait pas de polling : il surveille le cluster en continu (*watch* sur les ressources gérées). Dès que le Deployment change, il compare au manifest de Git, et comme `selfHeal: true`, il relance un sync partiel limité aux ressources en écart.

**Plus fort, `kubectl -n staybook delete deploy gateway`** à 17:08:19 : Argo CD recrée le Deployment dans la même seconde (`Initiated automated sync` → `Partial sync … succeeded` à 16:08:19Z). Sa santé passe `Progressing` puis `Healthy` à 16:08:25. En sondant le site toutes les 0,5 s :
- la **page d'accueil** (`/`, servie par le front) a répondu **200 en continu** ;
- l'**API** (`/api/hotels`, qui passe par la gateway) a répondu **502 pendant ~6 s** (17:08:19 → 17:08:25), le temps que le nouveau pod gateway démarre et passe sa readinessProbe, puis 200 à nouveau.

Le site est donc resté accessible. Seules les données ont été indisponibles quelques secondes, et sans aucune intervention humaine.


### Question 30

_Décrivez précisément la chaîne d'événements entre votre `git push` et le nouveau titre dans le navigateur. Qui a fait quoi ? Où est passé l'ancien pod ? (`kubectl -n staybook get rs`)_

Réponse :

Chaîne observée (push à 17:08:48, nouveau titre servi à 17:09:04, soit ~16 s) :

1. **Moi, hors du cluster** : modification du `<title>` dans `staybook-src/front/public/index.html`, `docker build -t staybook-front:v2`, puis `kind load docker-image` : l'image est copiée dans le containerd du nœud.
2. **Moi, dans Git** : `image: staybook-front:v1` → `v2` dans `apps/staybook/40-front.yaml`, commit `95abfab`, `git push` vers GitHub.
3. **argocd-repo-server** : sur le `--refresh` (sinon au prochain poll, ≤ 3 min), il fait un `git fetch` du dépôt et régénère les manifests de `apps/staybook`.
4. **argocd-application-controller** : il compare ces manifests au cluster, voit que l'image du Deployment `front` diffère (`OutOfSync`) et, comme l'auto-sync est actif, il **applique** le nouveau Deployment via l'API server.
5. **Kubernetes, contrôleur de Deployment** : le *pod template* a changé, donc il crée un **nouveau ReplicaSet** (`front-cc75d9b86`) et fait un **RollingUpdate** : il crée un pod v2, attend sa readinessProbe, puis réduit l'ancien ReplicaSet à 0.
6. **kubelet du nœud** : il démarre le conteneur avec `staybook-front:v2`, déjà présent localement (`IfNotPresent`, aucun registre contacté).
7. **Le navigateur** : http://localhost:8081 → kind → NodePort 30081 → Service `front` → nouveau pod : titre « StayBook v2 ».

**L'ancien pod** a été supprimé, mais son **ReplicaSet est conservé** avec 0 réplica (`kubectl -n staybook get rs` : `front-5c76dcf44c 0 0 0`, et `front-cc75d9b86 1 1 1`). Kubernetes garde ainsi l'historique des versions (`revisionHistoryLimit`, 10 par défaut), ce qui permet un `kubectl rollout undo`. Mais en GitOps, on revient en arrière par Git.


### Question 31

_Dans une entreprise, qui construirait et chargerait l'image à votre place, et où ? Quelle partie de ce que vous venez de faire relève de la **CI**, et quelle partie du **CD** ?_

Réponse :

En entreprise, ni le `docker build` ni le `kind load` ne seraient faits à la main. C'est une **chaîne d'intégration continue** (GitHub Actions, GitLab CI, Jenkins) qui, à chaque push sur le dépôt **des sources** : lance les tests, **construit l'image**, lui donne un tag unique (numéro de version ou SHA du commit), et la **pousse dans un registre** (ghcr.io, Docker Hub, Harbor, ECR…). Les nœuds la **tirent** ensuite eux-mêmes depuis ce registre : on ne charge jamais d'image directement dans un nœud.

Ce que j'ai fait, réparti :
- **CI** (Intégration Continue) : modifier le code, `docker build`, taguer `v2`, mettre l'image à disposition (`kind load`, qui remplace ici le push vers un registre). La CI peut aussi faire le dernier pas : un commit automatique du nouveau tag dans le dépôt GitOps.
- **CD** (Déploiement Continu) : changer le tag dans le dépôt GitOps, puis tout ce qui suit le `git push`. Argo CD détecte, compare et applique, et Kubernetes fait le rolling update. C'est le rôle d'Argo CD, en pull, **depuis l'intérieur** du cluster.

La frontière est le **dépôt GitOps** : la CI y écrit un tag, le CD le lit. La CI n'a jamais besoin d'accès au cluster. (C'est ce que met en place le TP 2.)


### Question 32

_Qu'est-il arrivé au ConfigMap ? Que se serait-il passé avec `prune: false` ? Pourquoi `prune` est-il désactivé par défaut dans Argo CD ?_

Réponse :

Après l'ajout de `apps/staybook/99-test.yaml` (commit `a2eea2c`), le ConfigMap `test-prune` est apparu dans le cluster. Après le `git rm` (commit `f5ee43b`), Argo CD l'a **supprimé du cluster** : `configmaps "test-prune" not found`, et le log du contrôleur indique `status: 'Pruned', phase: 'Succeeded', message: 'pruned'`. C'est l'effet de `prune: true` : une ressource gérée par l'Application qui n'existe plus dans Git est supprimée.

Avec `prune: false`, le ConfigMap serait **resté** dans le cluster. L'Application serait passée `OutOfSync`, et la ressource serait marquée « requires pruning » dans l'interface. Elle ne serait supprimée qu'avec un sync manuel en cochant *Prune*. Le cluster contiendrait alors des objets que Git ne décrit plus : de la dérive accumulée.

`prune` est **désactivé par défaut** par **sécurité** : une suppression est destructive et souvent irréversible (un PVC supprimé, ce sont des données perdues). Une erreur banale dans Git, comme un fichier déplacé ou renommé, une mauvaise branche, un `path` modifié ou un `kustomization.yaml` cassé qui ne génère plus rien, pourrait sinon effacer automatiquement des ressources de production. Argo CD préfère donc qu'on active ce comportement **explicitement**, en connaissance de cause. On peut aussi protéger une ressource précise avec l'annotation `argocd.argoproj.io/sync-options: Prune=false`.


### Question 33

_Après le rollback (a), l'auto-sync est-il toujours actif ? Pourquoi Argo CD fait-il cela ? Laquelle des deux approches est « GitOps-compatible » ? Le site est-il resté disponible pendant le mauvais déploiement, et grâce à quoi ?_

Réponse :

Le mauvais déploiement (commit `efa1f02`, `image: staybook-front:v3`) : Argo CD est `Synced`, puisqu'il a bien appliqué ce que dit Git, mais `Progressing`, puis `Degraded` au-delà du `progressDeadlineSeconds`. Le nouveau pod est en **`ImagePullBackOff`** : `staybook-front:v3` n'existe ni dans le nœud ni sur Docker Hub.

**(a) `argocd app rollback staybook 4`** : Argo CD **refuse** tant que l'auto-sync est actif :

```
rpc error: code = FailedPrecondition desc = rollback cannot be initiated when auto-sync is enabled
```

J'ai dû le désactiver (`argocd app set staybook --sync-policy none`), puis le rollback a réussi : l'image est redevenue `v2`. **Après le rollback, l'auto-sync n'est plus actif** (`Sync Policy: Manual`), et l'application est `OutOfSync from HEAD (efa1f02)`.

Pourquoi : un rollback remet le cluster dans un état **différent de Git**. Si l'auto-sync restait actif, Argo CD verrait immédiatement l'écart et redéploierait la v3 depuis Git, ce qui annulerait le rollback. Argo CD impose donc de sortir du mode automatique : le rollback est une mesure **d'urgence**, temporaire, qui laisse le cluster en écart avec Git.

**(b) `git revert HEAD && git push`** : crée le commit `911ff7b`, qui annule `efa1f02`. Git redit `v2`. Après avoir réappliqué `argocd/staybook-application.yaml` pour réactiver l'auto-sync, l'application est `Synced to HEAD (911ff7b)` / `Healthy`.

*Observé : en réactivant l'auto-sync, Argo CD avait encore l'ancien HEAD `efa1f02` en cache. Il a donc redéployé la v3 pendant ~8 s (entrée 7 de l'historique), puis il a lu le revert et il est revenu à la v2. Leçon : il faut corriger Git **et** s'assurer qu'Argo CD l'a lu (`--refresh`) avant de réactiver l'auto-sync.*

L'approche **GitOps-compatible** est **(b), `git revert`** : Git reste la source de vérité, le retour en arrière est un commit tracé (qui, quand, pourquoi), relisible et lui-même réversible, et l'auto-sync reste actif. (a) ne sert qu'en urgence, et il faut ensuite aligner Git.

**Le site est resté disponible** pendant le mauvais déploiement (`curl` → 200, titre « StayBook v2 »), grâce au **RollingUpdate** du Deployment : Kubernetes crée d'abord le nouveau pod, et ne supprime l'ancien que lorsque le nouveau est **prêt** (readinessProbe). Le pod v3 n'est jamais devenu prêt, donc l'ancien pod v2 (`front-cc75d9b86-t7c5z`) n'a jamais été arrêté et a continué de recevoir le trafic du Service.


### Question 34

_Reliez chaque entrée de `argocd app history staybook` à un commit de `git log --oneline`. Face à un auditeur qui demande « qui a déployé quoi, quand », que répondez-vous, en combien de commandes ?_

Réponse :

`argocd app history staybook` comparé à `git log --oneline` :

| ID | Date du sync | Révision | Commit Git (heure) | Ce qui a été déployé |
|---|---|---|---|---|
| 0 | 16:51:54 | `7b54f2d` | `feat(argocd): Application staybook en YAML` (16:51:32) | 1er déploiement (sync manuel, partie 6.2) |
| 1 | 17:03:11 | `2155f92` | `scale(hotels-service): 2 réplicas` (17:02:53) | hotels-service à 2 réplicas (sync manuel, 8.1) |
| 2 | 17:08:57 | `95abfab` | `release(front): v2` (17:08:46) | front v2 (auto-sync, 8.4) |
| 3 | 17:09:20 | `a2eea2c` | `test: ConfigMap pour tester le prune` (17:09:09) | ajout du ConfigMap (8.5) |
| 4 | 17:09:27 | `f5ee43b` | `test: suppression du ConfigMap (prune)` (17:09:17) | ConfigMap supprimé (prune) |
| 5 | 17:09:38 | `efa1f02` | `release(front): v3 (image inexistante)` (17:09:28) | mauvais déploiement (8.6) |
| 6 | 17:10:16 | `f5ee43b` | (aucun nouveau commit) | **`argocd app rollback staybook 4`** : on redéploie la révision de l'ID 4 |
| 7 | 17:10:25 | `efa1f02` | (aucun nouveau commit) | réactivation de l'auto-sync avec l'ancien HEAD en cache (voir Q33) |

Les commits `87bac28` (Application concierge) et `408328c` (auto-sync) n'apparaissent pas : ils modifient `argocd/`, qui n'est pas le `path` de l'Application `staybook`. Le revert `911ff7b`, lui, n'a pas d'entrée parce qu'Argo CD l'a appliqué par un *partial sync*, comme les self-heal de la question 29, qui n'apparaissent pas non plus dans l'historique. L'historique d'Argo CD est donc **incomplet**, et il est **limité** aux 10 derniers syncs par défaut (`revisionHistoryLimit`). Il disparaît aussi avec le cluster.

Face à un auditeur qui demande « qui a déployé quoi, quand », je réponds avec **une seule commande, `git log`**, sur le dépôt GitOps : par exemple `git log --format='%h %ad %an %s' --date=iso -- apps/staybook` (et `git show <sha>` pour le détail d'un changement). Puisque tout passe par Git et que l'auto-sync applique ce qui est dans Git, l'historique des commits **est** l'historique des déploiements : auteur, date, contenu exact (diff), message, et via les PR, qui a relu et approuvé. Il est complet, permanent et indépendant du cluster. `argocd app history` sert seulement à confirmer quand le cluster a convergé.


## Preuves

- Sortie de `argocd app history staybook` :

```
SOURCE  https://github.com/zebaxter/staybook-gitops.git
ID      DATE                           REVISION
0       2026-11-24 16:51:54 +0100 CET  HEAD (7b54f2d)
1       2026-11-24 17:03:11 +0100 CET  HEAD (2155f92)
2       2026-11-24 17:08:57 +0100 CET  HEAD (95abfab)
3       2026-11-24 17:09:20 +0100 CET  HEAD (a2eea2c)
4       2026-11-24 17:09:27 +0100 CET  HEAD (f5ee43b)
5       2026-11-24 17:09:38 +0100 CET  HEAD (efa1f02)
6       2026-11-24 17:10:16 +0100 CET  HEAD (f5ee43b)
7       2026-11-24 17:10:25 +0100 CET  HEAD (efa1f02)
```

- Sortie de `argocd app list` :

```
NAME              CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                             PATH            TARGET
argocd/concierge  https://kubernetes.default.svc  staybook   default  Synced  Healthy  Auto-Prune  <none>      https://github.com/zebaxter/staybook-gitops.git  apps/concierge  HEAD
argocd/staybook   https://kubernetes.default.svc  staybook   default  Synced  Healthy  Auto-Prune  <none>      https://github.com/zebaxter/staybook-gitops.git  apps/staybook   HEAD
```
