# Mini-projet 6 — Sécurisation du cycle de vie des secrets dans Kubernetes

Ce dépôt montre, de bout en bout et de façon reproductible, comment sécuriser le cycle de vie
d'un secret dans Kubernetes : l'identifier, le stocker chiffré, le distribuer à la bonne
identité, limiter qui peut le joindre, le faire tourner, puis le révoquer et le supprimer.

Le coffre de secrets est **HashiCorp Vault**, la distribution vers le cluster passe par
**External Secrets Operator (ESO)**, et l'hygiène du dépôt est vérifiée par **Gitleaks**.

Fil conducteur : *nous n'avons pas rendu le vol impossible, nous l'avons rendu coûteux, visible
et de courte durée.*

## Architecture

```text
        JWT de ServiceAccount                  écrit des Secrets
Vault  ────────────────────►  ESO (external-secrets)  ─────────────────┐
  │                                                                     v
  │ KV v2 : secret/…                                        etcd (chiffrement aescbc)
  │ moteur database : creds à la demande                                  │
  v                                                                     v
PostgreSQL  ◄──── 5432 ────  API backend  ◄──── volume monté 0400 ──────┘
                                   ^
                                   │ 8080
                              frontend (aucun secret)
```

Vault ne parle jamais directement aux applications : c'est ESO qui porte la relation, et le Pod
ne lit qu'un fichier monté, réécrit à chaque rotation.

## Structure du dépôt

```text
k8s-secrets-project/
├── namespaces/        # namespaces + profils Pod Security Admission
├── vault/             # déploiement Vault, politiques (HCL), rôles, configuration
├── external-secrets/  # ClusterSecretStore et ExternalSecrets (le pont Vault -> Kubernetes)
├── rbac/              # Roles et RoleBindings nominatifs (get sur des noms, jamais list)
├── workloads/         # frontend, API backend, PostgreSQL
└── demos/             # scripts et documentation des démonstrations
```

## Les six couches de contrôle

| Couche | Question à laquelle elle répond | Emplacement |
|---|---|---|
| Matrice des secrets | Que dois-je protéger ? | `demos/` (documentation) |
| Chiffrement | Où le secret est-il lisible ? | `vault/` + chiffrement etcd du cluster |
| RBAC nominatif | Qui peut lire quoi via l'API ? | `rbac/` |
| Admission | À quoi un objet a-t-il droit de ressembler ? | `namespaces/` (labels PSS) |
| NetworkPolicy | Qui peut parler à qui ? | `namespaces/` et politiques réseau |
| Rotation et fin de vie | Combien de temps un secret volé reste-t-il utile ? | `external-secrets/` |

Aucune couche ne remplace les autres : chacune ferme un chemin que la précédente laisse ouvert.

## Prérequis

- Un cluster Kubernetes local (kind ou Minikube) et `kubectl`
- `helm` pour installer External Secrets Operator
- Vault déployé et initialisé (mode dev accepté pour ce laboratoire)
- Facultatif : `gitleaks` pour le contrôle d'absence de secret

## Déploiement

```bash
# 1. Namespaces et frontières de sécurité
kubectl apply -f namespaces/

# 2. Vault : politiques et rôles d'authentification
kubectl apply -f vault/

# 3. External Secrets Operator, puis le pont vers Vault
kubectl apply -f external-secrets/

# 4. Droits nominatifs des applications
kubectl apply -f rbac/

# 5. Charges de travail
kubectl apply -f workloads/
```

L'ordre compte : les namespaces d'abord, les droits ensuite, les applications en dernier.

## Vérifier que la chaîne fonctionne

```bash
kubectl get clustersecretstore          # doit être Valid / Ready: True
kubectl get externalsecret -A           # doit être SecretSynced / True
kubectl -n backend get secret           # les Secrets sont créés par ESO, jamais à la main
```

Si les ExternalSecrets restent en erreur, l'ordre de diagnostic est toujours le même :
réseau (DNS, Services, NetworkPolicy), puis authentification, puis droits.

## Démonstrations

Les scripts de `demos/` répondent chacun à une question précise et renvoient un code d'erreur si
le résultat attendu n'est pas obtenu.

| Démonstration | Question posée |
|---|---|
| Accès refusé | Un Pod sans droit peut-il lire un secret ? |
| Accès autorisé | Le backend reçoit-il seulement ce dont il a besoin ? |
| Chiffrement | etcd livre-t-il les secrets en clair ? |
| Rotation | Une rotation exige-t-elle un redémarrage ? |
| Réseau | Le backend peut-il joindre Vault directement ? |
| Gitleaks | Un secret peut-il fuiter par Git ? |
| Utilisateur | Un humain non autorisé peut-il lire un namespace ? |

## Hygiène du dépôt

- **Aucun secret n'est versionné.** Les valeurs sensibles vivent dans Vault et dans un Secret
  Kubernetes créé hors Git.
- `gitleaks_*.tar.gz` et les fichiers d'identifiants locaux sont exclus par `.gitignore`.
- Avant chaque commit :

```bash
gitleaks detect --source . -v      # ou via Docker si le binaire n'est pas disponible
```

## Limites assumées

Vault en mode dev (stockage mémoire), token root trivial, PostgreSQL sans stockage persistant,
certificat utilisateur d'un jour, application des NetworkPolicy dépendante du CNI.
Les mécanismes démontrés sont ceux de la production ; seule leur exploitation est simplifiée.

## Contexte

Mini-projet 6 du semestre 5 (DevSecOps et conteneurisation) : sécurisation du cycle de vie des
secrets dans Kubernetes. Réalisé en français, cible locale kind.
