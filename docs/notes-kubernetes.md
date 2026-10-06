# Notes Kubernetes — TaskFlow

## Séance 4 : limites de l'approche par Pods

Déploiement : un Pod `db` (PostgreSQL) et un Pod `api` dans le namespace `taskflow`, sans contrôleur, sans Service, sans volume. L'API joint la base par l'adresse IP du Pod `db` (`DB_HOST`).

### Constat 1 — Pod `db` supprimé puis recréé

- Le Pod recréé obtient une **nouvelle adresse IP**.
- Les **tâches sont perdues** : sans volume, les données de PostgreSQL vivaient dans le conteneur supprimé.
- L'API continue de tourner et `/healthz` répond toujours (cette route ne consulte pas la base), mais `/readyz` renvoie 503 et `/api/tasks` renvoie 500 : elle vise encore l'ancienne adresse.
- Pour qu'elle retrouve la base, il faut reporter la nouvelle adresse dans `DB_HOST`, puis supprimer et recréer le Pod `api` (l'environnement d'un Pod n'est pas modifiable).

**Ce qu'un orchestrateur devrait apporter :** un point d'accès stable, avec un nom DNS et une adresse qui ne changent pas quand le Pod est remplacé (Service), et un stockage qui survit au Pod (PersistentVolumeClaim).

### Constat 2 — Pod `api` supprimé

- Le Pod **n'est pas recréé**.
- La boucle de réconciliation compare un état désiré à l'état réel. Un Pod créé seul n'est l'état désiré d'aucun contrôleur : une fois supprimé, rien n'indique qu'il devrait exister.

**Ce qu'un orchestrateur devrait apporter :** un contrôleur qui porte l'état désiré (« N Pods de l'API ») et recrée les Pods manquants : Deployment, qui gère un ReplicaSet.

### Constat 3 — `kill 1` dans le conteneur de l'API

- La colonne `RESTARTS` passe à 1 ; le **nom du Pod et son adresse IP ne changent pas**.
- Le kubelet redémarre le conteneur sur place, selon `restartPolicy: Always` (valeur par défaut).
- Avec Docker (`restart: unless-stopped`, séance 1), le comportement est proche : même conteneur, même machine. Swarm, à l'inverse, remplace la tâche par un nouveau conteneur, avec un nouveau nom et une nouvelle adresse.
- Ce redémarrage ne couvre que l'arrêt du processus : ni la suppression du Pod, ni la perte du nœud, ni une application bloquée mais toujours vivante.

**Ce qu'un orchestrateur devrait apporter :** des sondes de santé (liveness, readiness) pour détecter une application bloquée ou pas encore prête, et un contrôleur pour replanifier le Pod sur un autre nœud si le sien tombe.

### Synthèse

| Constat | Manque | Réponse de Kubernetes |
|---|---|---|
| Nouvelle IP, données perdues | Adresse stable, persistance | Service, PersistentVolumeClaim |
| Pod non recréé | Contrôleur | Deployment / ReplicaSet |
| Redémarrage sur place uniquement | Santé applicative, replanification | Sondes, contrôleur |

### Choix provisoires à lever

| Choix | Raison | Remplacé par |
|---|---|---|
| `DB_HOST` = adresse IP du Pod `db` | Le nom `db` n'est pas résolu : aucun Service | Service `db` (séance 5) |
| Mot de passe de test dans les manifestes | Pas encore de Secret | Secret (séance 6) |
| Pas de volume pour PostgreSQL | Stockage persistant non abordé | PersistentVolumeClaim (séance 7) |
