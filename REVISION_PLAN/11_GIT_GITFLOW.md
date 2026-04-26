# 11 — Git & GitFlow

## 1. Git — Commandes Essentielles

```bash
# Workflow quotidien
git status                       # état du working tree
git diff --staged                # diff des fichiers stagés
git log --oneline --graph        # historique graphique

# Rebase vs Merge
git rebase main                  # rejouer ses commits sur main (historique linéaire)
git merge --no-ff feature/xxx    # merge avec merge commit (préserve l'historique)

# Stash
git stash push -m "WIP: feature X"
git stash list
git stash pop                    # restaurer + supprimer
git stash apply stash@{1}        # restaurer sans supprimer

# Cherry-pick : appliquer un commit spécifique
git cherry-pick abc1234

# Reset
git reset --soft HEAD~1          # annule le commit, garde les changements stagés
git reset --mixed HEAD~1         # annule le commit, garde les changements non stagés
git reset --hard HEAD~1          # annule le commit ET les changements (DESTRUCTIF)

# Reflog : récupérer un commit "perdu"
git reflog                       # historique de toutes les actions HEAD
git reset --hard HEAD@{3}        # revenir à l'état d'il y a 3 actions

# Bisect : trouver le commit qui a introduit un bug
git bisect start
git bisect bad                   # commit actuel = bugué
git bisect good v1.2.3           # tag connu comme bon
# Git checkout automatiquement les commits intermédiaires
git bisect good / git bisect bad # répéter jusqu'à trouver le coupable
git bisect reset
```

---

## 2. GitFlow

### Branches

```
main (ou master)
  ├── Contient : code en production
  └── Tags : versions (v1.0.0, v1.1.0)

develop
  ├── Contient : intégration des features
  └── Base pour les releases

feature/xxx    (depuis develop, merge dans develop)
release/x.y.z  (depuis develop, merge dans main ET develop)
hotfix/xxx     (depuis main, merge dans main ET develop)
bugfix/xxx     (depuis develop, merge dans develop)
```

### Cycle de Vie GitFlow

```bash
# FEATURE
git flow feature start user-authentication
# ... développement ...
git flow feature finish user-authentication
# = merge dans develop + suppression de la branche

# RELEASE
git flow release start 1.2.0
# Corrections de bugs mineurs, bump de version, CHANGELOG
git flow release finish 1.2.0
# = merge dans main + tag v1.2.0 + merge dans develop

# HOTFIX (bug critique en production)
git flow hotfix start critical-security-fix
# Correction minimale du bug
git flow hotfix finish critical-security-fix
# = merge dans main + tag v1.2.1 + merge dans develop
```

### Naming Conventions

```
feature/JIRA-123-user-authentication
bugfix/JIRA-456-fix-null-pointer-login
release/1.2.0
hotfix/1.2.1-fix-sql-injection
```

---

## 3. Conventional Commits

```
<type>(<scope>): <description>

feat(auth): add JWT refresh token support
fix(order): resolve null pointer in calculateTotal
refactor(customer): extract email validation to service
test(order): add integration tests for order creation
docs(api): update swagger documentation for auth endpoints
chore(deps): upgrade spring-boot to 3.2.0
perf(query): add index on orders.customer_id
```

---

## 4. Merge Strategies

```bash
# Fast-forward (défaut si possible) : pas de merge commit
# Résultat : historique linéaire, pas de trace de la branche
git merge feature/xxx

# No fast-forward : toujours créer un merge commit
# Résultat : historique non linéaire mais traçable
git merge --no-ff feature/xxx

# Squash : condenser tous les commits en un seul
git merge --squash feature/xxx
git commit -m "feat(xxx): feature description"
```

---

## Questions d'Interview — Git & GitFlow

---

### Q1 : Quelle est la différence entre `git rebase` et `git merge` ?

**Réponse :**

```
# Situation initiale
main:    A-B-C
feature:     D-E-F

# merge : crée un merge commit (M)
main après merge: A-B-C-M
                      \   /
feature:          D-E-F

# rebase : rejoue les commits D,E,F sur C
main après rebase: A-B-C-D'-E'-F'
(historique linéaire, commits recréés avec nouveaux SHA)
```

**Règle d'or :** Ne jamais rebaser des branches publiques/partagées (on réécrit l'historique → problèmes pour les autres). Rebaser uniquement ses branches locales avant de les merger.

---

### Q2 : Comment gérer un conflit lors d'un rebase ?

**Réponse :**

```bash
git rebase main

# Si conflit :
# 1. Résoudre les conflits dans les fichiers
git add <fichier-résolu>
git rebase --continue   # passer au commit suivant

# Pour annuler complètement le rebase
git rebase --abort

# Outils de merge
git mergetool          # ouvre l'outil configuré (VSCode, IntelliJ...)
```

---

### Q3 : Quand utiliser GitFlow vs trunk-based development ?

**Réponse :**

| | GitFlow | Trunk-Based |
|-|---------|-------------|
| **Releases** | Versionnées, planifiées | Continue |
| **Équipe** | Plusieurs versions en parallèle | Déploiement fréquent |
| **Branches** | Longue durée de vie | Courte durée (< 1 jour) |
| **CI/CD** | Moins fréquent | Plusieurs fois par jour |
| **Feature flags** | Non requis | Recommandés |

**Recommandation :** GitFlow pour les applications avec cycles de release, trunk-based pour les équipes DevOps avec CD.

---

## Pièges Classiques

- Faire `git push --force` sur `main`/`develop` → réécriture de l'historique partagé
- Branches features qui vivent trop longtemps → conflits de merge catastrophiques
- Ne pas taguer les releases → impossibilité de rollback précis
- Commiter des secrets, fichiers compilés, `.env` → `.gitignore` obligatoire dès le début
- `git reset --hard` sans réfléchir → perte irréversible de travail non commité
