# 13 — Agile / Scrum / Jira / Confluence

## 1. Scrum — Cadre de Travail

### Rôles

| Rôle | Responsabilités |
|------|----------------|
| **Product Owner** | Vision produit, backlog priorisé, valeur business |
| **Scrum Master** | Facilitation, suppression des obstacles, coaching Agile |
| **Dev Team** | Auto-organisée, multi-compétences, livraison de l'incrément |

**Référent Technique dans Scrum :**
- Participe activement au **Refinement** (estimation, découpage technique)
- Garantit la **Definition of Done** technique
- Identifie et communique la **dette technique**
- Guide l'équipe sur les choix d'architecture

---

### Cérémonies

```
Sprint (2 semaines)
├── Sprint Planning (2-4h)
│   ├── "Quoi ?" : sélectionner les items du backlog
│   └── "Comment ?" : décomposer en tâches techniques
│
├── Daily Scrum (15 min max, debout)
│   ├── "Qu'ai-je fait hier ?"
│   ├── "Que vais-je faire aujourd'hui ?"
│   └── "Y a-t-il des obstacles ?"
│
├── Sprint Review (2-4h)
│   ├── Démonstration de l'incrément au PO/stakeholders
│   └── Feedback et adaptation du backlog
│
└── Sprint Retrospective (2-3h)
    ├── "Qu'est-ce qui s'est bien passé ?"
    ├── "Qu'est-ce qui pourrait être amélioré ?"
    └── "Actions concrètes pour le prochain sprint"
```

---

### Definition of Done (DoD)

```
✅ Code développé et testé (unit + intégration)
✅ Couverture de tests ≥ 80% (nouveau code)
✅ Quality Gate SonarQube : A (pas de bugs, pas de vulnérabilités)
✅ Code review approuvé (au moins 1 reviewer)
✅ Merge sur develop via Pull Request
✅ Pipeline CI vert (build + tests + Sonar)
✅ Documentation mise à jour (Swagger, Confluence si nécessaire)
✅ Déployé en environnement de recette
✅ Validé par le PO (acceptance criteria remplis)
```

---

## 2. User Stories & Estimation

### Format Standard
```
En tant que <persona>
Je veux <action>
Afin de <bénéfice>

Critères d'acceptation :
- CA1 : Étant donné... Quand... Alors...
- CA2 : ...

Story : "En tant que client, je veux pouvoir filtrer mes commandes par statut
         afin de retrouver rapidement les commandes en attente."

CA1 : Étant donné que je suis connecté,
      Quand je sélectionne le filtre "En attente",
      Alors seules les commandes avec statut PENDING s'affichent.

CA2 : Étant donné qu'il n'y a aucune commande PENDING,
      Quand j'applique le filtre,
      Alors un message "Aucune commande en attente" s'affiche.
```

### Estimation (Planning Poker)
```
Suite de Fibonacci : 1, 2, 3, 5, 8, 13, 21, ?, ∞

1  : trivial (1h)
2  : simple  (demi-journée)
3  : standard (1 jour)
5  : complexe (2-3 jours)
8  : très complexe (à découper si possible)
13+: trop gros → diviser en plusieurs stories
?  : manque d'information → Refinement nécessaire
```

---

## 3. Jira

### Types d'Items

```
Epic     : grande fonctionnalité (ex: "Gestion des commandes")
  └── Story  : user story (ex: "Filtrer les commandes")
        └── Sub-task : tâche technique (ex: "Créer endpoint REST")

Bug     : défaut logiciel
Task    : tâche technique sans valeur utilisateur directe
Spike   : investigation/recherche (timeboxé)
```

### Workflow Typique

```
To Do → In Progress → Code Review → QA/Testing → Done
                   ↓
              Blocked (avec commentaire sur le blocage)
```

### Requêtes JQL Utiles

```
# Mon backlog dans le sprint courant
assignee = currentUser() AND sprint in openSprints() AND status != Done

# Stories bloquées dans l'équipe
project = MYAPP AND status = Blocked AND sprint in openSprints()

# Bugs critiques non résolus
project = MYAPP AND issuetype = Bug AND priority in (High, Critical) AND status != Done

# Recherche par épic
"Epic Link" = MYAPP-100 AND status != Done
```

---

## 4. Confluence

### Documentation Technique Utile

```
Espace équipe/
├── Architecture/
│   ├── Architecture Decision Records (ADR)
│   ├── Diagrammes de composants
│   └── Modèle de données
├── Runbooks/
│   ├── Procédure de déploiement
│   ├── Gestion des incidents
│   └── Rollback procedure
├── Onboarding/
│   └── Guide de démarrage dev
└── Sprints/
    ├── Sprint 42 - Review notes
    └── Sprint 43 - Planning
```

### Architecture Decision Record (ADR)

```markdown
# ADR-007 : Utilisation de Kafka pour les événements inter-services

## Statut : Accepté

## Contexte
Les microservices communiquent actuellement par REST synchrone.
Problèmes : couplage fort, cascade de pannes, latence cumulée.

## Décision
Utiliser Apache Kafka pour la communication asynchrone inter-services
pour les événements métier (OrderCreated, PaymentProcessed, etc.)

## Conséquences
+ : Découplage des services, meilleure résilience
+ : Auditabilité des événements
- : Complexité opérationnelle (broker à maintenir)
- : Eventual consistency (plus de transactions distribuées ACID)

## Alternatives considérées
- RabbitMQ : écarté car moins adapté aux grandes volumétries
- REST asynchrone avec polling : trop complexe côté client
```

---

## Questions d'Interview — Agile/Scrum

---

### Q1 : Comment gérez-vous la dette technique dans un contexte Scrum ?

**Réponse :**

En tant que Référent Technique, plusieurs approches :

1. **Rendre la dette visible** : créer des tickets Jira de type "Technical Debt" avec estimation et impact
2. **Budget sprint** : réserver 20% de la capacité du sprint pour la dette technique (règle du 20%)
3. **Boy Scout Rule** : toujours laisser le code dans un meilleur état qu'on ne l'a trouvé
4. **ADR** : documenter les compromis pris délibérément pour des raisons de délai
5. **SonarQube** : utiliser la "Technical Debt" metric comme outil de communication avec le PO

---

### Q2 : Quelle est la différence entre velocity et capacité en Scrum ?

**Réponse :**

- **Capacité** : nombre de jours/homme disponibles dans le sprint (congés, formations déduits)
- **Velocity** : nombre moyen de points de story livrés sur les derniers sprints (3-5 sprints)

```
Capacité sprint : 4 devs × 10 jours = 40 jours
Velocity moyenne : 42 points
Ratio : ~1 point = 1 jour (pour calibrer les estimations)

Sprint Planning : ne pas dépasser la velocity pour éviter le sur-engagement
```

---

### Q3 : Comment gérez-vous les dépendances techniques entre stories dans un sprint ?

**Réponse :**

1. **Refinement** : identifier les dépendances en amont (avant le sprint planning)
2. **Sous-tâches** : décomposer en sous-tâches ordonnées (API d'abord, front ensuite)
3. **Feature flags** : développer en parallèle sans bloquer (front derrière un flag désactivé)
4. **Contrat d'interface** : définir le contrat API (Swagger) avant l'implémentation → front et back travaillent en parallèle avec des mocks
5. **Blocker** : si vraie dépendance externe → la marquer dans Jira et en parler au daily

---

## Pièges Classiques

- Confondre Scrum Master et chef de projet → le SM facilite, ne manage pas
- Ignorer la rétro → l'équipe ne s'améliore pas
- Sprint trop chargé → stories non terminées → velocity instable → démotivation
- Définition of Done non partagée → "terminé" signifie différentes choses pour chacun
- Stories trop grandes (> 8 points) → risque de ne pas finir dans le sprint → découper
