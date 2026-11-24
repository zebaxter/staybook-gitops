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


### Question 7

_Pourquoi le script tague les images `v1` plutôt que `latest` ? Cherchez la règle de Kubernetes sur `imagePullPolicy` quand le tag est `latest` : que se passerait-il au démarrage d'un pod ?_

Réponse :


### Question 8

_Où sont physiquement les images maintenant ? Vérifiez avec `docker exec argocd-tp-control-plane crictl images | grep staybook`. Qu'est-ce que `crictl`, et pourquoi n'est-ce pas `docker images` ?_

Réponse :


### Question 9

_Combien d'objets vont être créés ? Parmi les types listés, trois ne sont pas des objets Kubernetes « standard ». Lesquels, et à quoi servent-ils ? (Indice : `CustomResourceDefinition`.)_

Réponse :


### Question 10

_Pourquoi est-il préférable de passer par Kustomize plutôt que de faire `kubectl apply -f install.yaml` puis `kubectl patch` sur le Service ? Pensez au jour où vous mettrez Argo CD à jour._

Réponse :


### Question 11

_Vérifiez votre tableau de la partie 1 : quel composant est un StatefulSet ? Avez-vous les sept composants ?_

Réponse :


### Question 12

_Essayez `kubectl apply -k argocd-install` **sans** `--server-side` (c'est sans danger, tout existe déjà). Quelle erreur obtenez-vous, sur quel objet ? Cherchez la cause (`last-applied-configuration`)._

Réponse :


### Question 13

_Pourquoi la commande se termine-t-elle par `| base64 -d` ? Que verriez-vous sans ? Un Secret Kubernetes est-il chiffré ?_

Réponse :


### Question 14

_Que signifient `--plaintext` et `--grpc-web` ? Que se passe-t-il sans `--plaintext` (essayez) ? Ces options seraient-elles acceptables en production ?_

Réponse :


### Question 15

_Regardez `apps/staybook/00-secrets.yaml` : les mots de passe PostgreSQL sont en clair dans Git. Est-ce acceptable ici ? Et en entreprise ? Citez une solution._

Réponse :


### Question 16

_Notez l'erreur. Pourquoi GitHub répond-il « Repository not found » et non « accès refusé » ?_

Réponse :


### Question 17

_Sous quelle forme Argo CD a-t-il stocké ces identifiants dans le cluster ? Trouvez l'objet avec `kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository` et listez ses clés (`-o jsonpath='{.data}'` puis décodez `url` et `username`, pas `password`)._

Réponse :


### Question 18

_Comparez trois façons de s'authentifier (token personnel, deploy key SSH, GitHub App) : portée, rotation, ce qui se passe quand la personne quitte l'entreprise. Laquelle recommanderiez-vous pour 200 dépôts ?_

Réponse :


### Question 19

_Combien de ressources Argo CD compte-t-il déployer ? Retrouvez chacune dans les fichiers de `apps/staybook/`._

Réponse :


### Question 20

_Les ressources ne sont pas déployées toutes en même temps. Ouvrez les manifests et trouvez l'annotation qui pilote cet ordre. Quel ordre a été choisi, et pourquoi ? À quoi cela correspondait-il dans `docker-compose.yml` ?_

Réponse :


### Question 21

_« OutOfSync » et « Healthy » sont deux statuts indépendants. Donnez un exemple d'application `Synced` mais `Degraded`, et un exemple `OutOfSync` mais `Healthy`._

Réponse :


### Question 22

_Vous venez de faire un `kubectl apply`… n'est-ce pas contraire au GitOps ? Pourquoi ce dernier `apply` est-il inévitable, et comment réduire cette « amorce » à un seul objet pour tout un cluster ? (Cherchez « App of Apps ».)_

Réponse :


### Question 23

_Reprenez votre réponse à la question 1 et complétez la colonne de droite en vous appuyant sur les manifests fournis._

Réponse :


### Question 24

_Pourquoi les deux Applications pointent-elles vers le **même namespace** `staybook` ? Que se passerait-il si le concierge était dans un namespace `concierge` ? (Indice : regardez `RAG_URL` dans le manifest de la gateway.)_

Réponse :


### Question 25

_Ouvrez `apps/concierge/20-rag-service.yaml`. Que font les deux `initContainers` ? En quoi est-ce différent des sync waves de la question 20 ? Pourquoi avoir les deux ?_

Réponse :


### Question 26

_Ouvrez `apps/concierge/11-ollama.yaml`. Où sont stockés les modèles téléchargés ? Supprimez le pod Ollama (`kubectl -n staybook delete pod -l app=ollama`) et vérifiez avec `kubectl -n staybook exec deploy/ollama -- ollama list` que les modèles sont toujours là. Qu'est-ce qui les a conservés ? Que signifie `strategy: Recreate` ?_

Réponse :


### Question 27

_Toujours dans ce fichier, lisez le bloc `resources`. Que se passe-t-il si Ollama dépasse la limite mémoire ? Et si le nœud n'a pas les 2 Gi demandés ?_

Réponse :


## Étape 3 — GitOps : Git pilote le cluster

### Question 28

_Sans `--refresh`, au bout de combien de temps Argo CD aurait-il vu le commit ? D'où vient cette valeur (ConfigMap `argocd-cm`, clé `timeout.reconciliation`) ? Quel mécanisme permettrait de réagir en quelques secondes sans interroger GitHub en boucle, et pourquoi ne peut-on pas l'utiliser ici ?_

Réponse :


### Question 29

_Que s'est-il passé, et en combien de temps ? Quel composant a agi ? Retrouvez sa trace : `kubectl -n argocd logs sts/argocd-application-controller | grep staybook | tail`. Essayez plus fort : `kubectl -n staybook delete deploy gateway`. Le site est-il resté accessible ?_

Réponse :


### Question 30

_Décrivez précisément la chaîne d'événements entre votre `git push` et le nouveau titre dans le navigateur. Qui a fait quoi ? Où est passé l'ancien pod ? (`kubectl -n staybook get rs`)_

Réponse :


### Question 31

_Dans une entreprise, qui construirait et chargerait l'image à votre place, et où ? Quelle partie de ce que vous venez de faire relève de la **CI**, et quelle partie du **CD** ?_

Réponse :


### Question 32

_Qu'est-il arrivé au ConfigMap ? Que se serait-il passé avec `prune: false` ? Pourquoi `prune` est-il désactivé par défaut dans Argo CD ?_

Réponse :


### Question 33

_Après le rollback (a), l'auto-sync est-il toujours actif ? Pourquoi Argo CD fait-il cela ? Laquelle des deux approches est « GitOps-compatible » ? Le site est-il resté disponible pendant le mauvais déploiement, et grâce à quoi ?_

Réponse :


### Question 34

_Reliez chaque entrée de `argocd app history staybook` à un commit de `git log --oneline`. Face à un auditeur qui demande « qui a déployé quoi, quand », que répondez-vous, en combien de commandes ?_

Réponse :


## Preuves

- Sortie de `argocd app history staybook` :

```

```

- Sortie de `argocd app list` :

```

```
