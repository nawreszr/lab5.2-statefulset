# Lab 5.2 : Base de données avec StatefulSet (K3s)

## 📖 Description du Lab
Ce laboratoire explore le déploiement d'applications avec état (Stateful) sur Kubernetes. Nous utilisons un **StatefulSet** pour déployer PostgreSQL, garantissant une identité réseau stable (via un Service Headless) et un stockage persistant dédié (via des `volumeClaimTemplates`). Nous démontrons également que les réplicas d'un StatefulSet ne partagent pas leurs données par défaut.

## 📂 Fichiers du projet
- `app-config.yaml` : ConfigMap contenant le nom de la base de données.
- `db.yaml` : Manifeste combiné du Service Headless et du StatefulSet PostgreSQL.
- *(Note : le Secret `db-credentials` est créé impérativement via la ligne de commande pour des raisons de sécurité).*

---

## 🧠 Questions de réflexion (Réponses)

**1. Pourquoi le Pod recréé s'appelle-t-il encore db-0 et retrouve-t-il ses données ?**
> Un StatefulSet donne à ses Pods une identité stable (nom et volume associé). Quand `db-0` est supprimé, le contrôleur du StatefulSet recrée un Pod portant exactement le même nom et le rattache au même PVC (`data-db-0`), ce qui lui permet de retrouver ses données.

**2. Quelle est la particularité d'un Service headless et pourquoi la préfère-t-on pour un StatefulSet ?**
> Un Service headless n'a pas d'adresse IP virtuelle (ClusterIP). Le DNS renvoie directement les adresses IP des Pods. Chaque Pod obtient un nom DNS stable (`db-0.db`, `db-1.db`). C'est indispensable pour les bases de données répliquées où l'on doit s'adresser à un membre spécifique du cluster.

**3. Pourquoi la table taches n'existait-elle pas dans db-1 ?**
> Un StatefulSet ne réplique pas les données. Chaque Pod a son propre volume (`data-db-1`) et sa propre instance de base de données indépendante. La synchronisation des données doit être configurée au niveau de l'application (ex: réplication PostgreSQL).

**4. Pourquoi le PVC data-db-1 a-t-il subsisté après la réduction à un réplica ?**
> Kubernetes ne supprime pas automatiquement les PVC d'un StatefulSet lors du scale down, pour éviter de détruire des données par accident. Il faut les supprimer explicitement.

**5. Pourquoi le changement du mot de passe dans le Secret n'a-t-il pas modifié le mot de passe de la base ?**
> La variable `POSTGRES_PASSWORD` n'est utilisée qu'à la toute première initialisation de la base (sur un volume vide). Une fois la base initialisée, le mot de passe est stocké dans les fichiers de données internes de PostgreSQL, et il faut utiliser une commande SQL (`ALTER USER`) pour le modifier.

---

## 🚀 Commandes clés et manipulations

### 1. Préparer la configuration
```bash
kubectl create namespace tp-k8s
kubectl config set-context --current --namespace=tp-k8s

kubectl apply -f app-config.yaml

kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=appuser \
  --from-literal=POSTGRES_PASSWORD='ChangeMe-123'
2. Déployer le StatefulSet
bash
kubectl apply -f db.yaml
kubectl get pods -w # Attendre que db-0 soit Running
kubectl exec db-0 -- pg_isready -U appuser -d appdb
3. Créer des données
bash
kubectl exec db-0 -- psql -U appuser -d appdb -c "CREATE TABLE taches (id serial PRIMARY KEY, titre text NOT NULL);"
kubectl exec db-0 -- psql -U appuser -d appdb -c "INSERT INTO taches (titre) VALUES ('Apprendre les StatefulSets'), ('Tester la persistance');"
4. Prouver la persistance
bash
kubectl delete pod db-0
kubectl get pods -w # Attendre la recréation de db-0
kubectl exec db-0 -- psql -U appuser -d appdb -c "SELECT * FROM taches;"
5. Tester le Service Headless
bash
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- nslookup db
# L'adresse IP renvoyée est celle du Pod, pas une IP virtuelle de Service
6. Tester les réplicas multiples
bash
kubectl scale statefulset db --replicas=2
kubectl exec db-1 -- psql -U appuser -d appdb -c "\dt"
# Résultat : Did not find any relations (le volume de db-1 est vide)
🧹 Nettoyage
bash
kubectl delete namespace tp-k8s
kubectl config set-context --current --namespace=default
rm -f app-config.yaml db.yaml
