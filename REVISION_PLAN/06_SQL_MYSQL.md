# 06 — SQL / MySQL

## 1. Requêtes Avancées

### Window Functions
```sql
-- Rang des ventes par région
SELECT
    region,
    salesperson,
    sales_amount,
    RANK() OVER (PARTITION BY region ORDER BY sales_amount DESC) AS rank_in_region,
    SUM(sales_amount) OVER (PARTITION BY region) AS region_total,
    sales_amount / SUM(sales_amount) OVER (PARTITION BY region) * 100 AS pct_of_region
FROM sales;

-- Calcul de running total
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;

-- LAG / LEAD : comparer avec ligne précédente/suivante
SELECT
    order_date,
    amount,
    LAG(amount, 1) OVER (ORDER BY order_date) AS previous_amount,
    amount - LAG(amount, 1) OVER (ORDER BY order_date) AS difference
FROM orders;
```

### CTEs (Common Table Expressions)
```sql
-- CTE simple
WITH active_customers AS (
    SELECT id, name, email
    FROM customers
    WHERE status = 'ACTIVE' AND created_at > '2024-01-01'
),
customer_orders AS (
    SELECT c.id, c.name, COUNT(o.id) AS order_count, SUM(o.amount) AS total_spent
    FROM active_customers c
    LEFT JOIN orders o ON o.customer_id = c.id
    GROUP BY c.id, c.name
)
SELECT *
FROM customer_orders
WHERE total_spent > 1000
ORDER BY total_spent DESC;

-- CTE Récursive : hiérarchie (org chart, arborescence de catégories)
WITH RECURSIVE category_tree AS (
    -- Ancre : racine
    SELECT id, name, parent_id, 0 AS depth, CAST(name AS CHAR(500)) AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Récursion
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, CONCAT(ct.path, ' > ', c.name)
    FROM categories c
    JOIN category_tree ct ON ct.id = c.parent_id
)
SELECT * FROM category_tree ORDER BY path;
```

---

## 2. Index & Optimisation

### Types d'Index MySQL
```sql
-- B-Tree (défaut) : pour =, >, <, BETWEEN, LIKE 'prefix%'
CREATE INDEX idx_orders_customer_date ON orders (customer_id, order_date);

-- Index composite : ordre des colonnes est crucial
-- Utiliser pour : customer_id seul, ou customer_id + order_date
-- PAS pour : order_date seul (left-most prefix rule)

-- Index UNIQUE : contrainte d'unicité + performance
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- Index FULLTEXT : recherche textuelle
CREATE FULLTEXT INDEX idx_products_search ON products (name, description);
SELECT * FROM products WHERE MATCH(name, description) AGAINST ('spring boot' IN BOOLEAN MODE);

-- Index partiel (MySQL 8+)
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'PENDING';
```

### EXPLAIN / EXPLAIN ANALYZE
```sql
EXPLAIN SELECT o.*, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'PENDING' AND o.created_at > '2024-01-01';

-- Colonnes importantes dans EXPLAIN :
-- type: ALL (scan complet) > range > ref > eq_ref > const (optimal)
-- key: index utilisé
-- rows: estimation du nombre de lignes examinées
-- Extra: Using filesort / Using temporary (signaux d'alerte)

-- Forcer l'utilisation d'un index
SELECT * FROM orders USE INDEX (idx_orders_customer_date)
WHERE customer_id = 123;
```

### Optimisations Courantes
```sql
-- 1. Éviter SELECT * (charge inutile, empêche index-only scan)
SELECT id, reference, amount FROM orders WHERE customer_id = ?;

-- 2. Éviter les fonctions sur les colonnes indexées
-- BAD : l'index n'est pas utilisé
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- GOOD
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- 3. Pagination efficace (éviter OFFSET élevé)
-- BAD : O(n) pour sauter les N premières lignes
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;
-- GOOD : Keyset Pagination
SELECT * FROM orders WHERE id > :lastId ORDER BY id LIMIT 20;
```

---

## 3. Transactions & Isolation

### Niveaux d'Isolation
```sql
-- READ UNCOMMITTED : lit données non commitées (dirty read)
-- READ COMMITTED : lit seulement les données commitées (défaut PostgreSQL)
-- REPEATABLE READ : même lecture → même résultat dans la transaction (défaut MySQL)
-- SERIALIZABLE : isolation totale (performances réduites)

SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

START TRANSACTION;
SELECT amount FROM accounts WHERE id = 1 FOR UPDATE; -- verrou exclusif
UPDATE accounts SET amount = amount - 100 WHERE id = 1;
UPDATE accounts SET amount = amount + 100 WHERE id = 2;
COMMIT;
```

### Deadlocks
```sql
-- Transaction A                    -- Transaction B
START TRANSACTION;                  START TRANSACTION;
UPDATE orders SET status='A' WHERE id=1;  -- verrouille order 1
                                    UPDATE orders SET status='B' WHERE id=2;  -- verrouille order 2
UPDATE orders SET status='A' WHERE id=2;  -- attend order 2
                                    UPDATE orders SET status='B' WHERE id=1;  -- attend order 1
-- DEADLOCK ! MySQL annule la transaction avec le moins de travail
```

**Prévention :** Toujours verrouiller les ressources dans le **même ordre** dans toutes les transactions.

---

## 4. Modélisation & Normalisation

```sql
-- 1NF : valeurs atomiques, pas de colonnes répétées
-- 2NF : 1NF + chaque attribut dépend de TOUTE la clé primaire
-- 3NF : 2NF + pas de dépendances transitives

-- Exemple de normalisation
-- Non normalisé
CREATE TABLE orders_bad (
    order_id INT,
    customer_name VARCHAR(100),  -- dépend de customer, pas de order
    customer_email VARCHAR(100),
    product_name VARCHAR(100),   -- dépend de product, pas de order
    product_price DECIMAL
);

-- Normalisé (3NF)
CREATE TABLE customers (id INT PK, name VARCHAR(100), email VARCHAR(100));
CREATE TABLE products (id INT PK, name VARCHAR(100), price DECIMAL);
CREATE TABLE orders (
    id INT PK,
    customer_id INT FK → customers,
    created_at TIMESTAMP,
    status ENUM('PENDING','CONFIRMED','CANCELLED')
);
CREATE TABLE order_lines (
    id INT PK,
    order_id INT FK → orders,
    product_id INT FK → products,
    quantity INT,
    unit_price DECIMAL  -- snapshot du prix au moment de la commande
);
```

---

## Questions d'Interview — SQL/MySQL

---

### Q1 : Quelle est la différence entre `INNER JOIN`, `LEFT JOIN` et `FULL OUTER JOIN` ?

**Réponse :**

```sql
-- INNER JOIN : lignes qui ont une correspondance dans les DEUX tables
SELECT o.reference, c.name
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id;
-- Résultat : uniquement les orders avec un customer existant

-- LEFT JOIN : toutes les lignes de gauche + correspondances droite (NULL si absent)
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id;
-- Résultat : tous les clients, même ceux sans commande (order_count = 0)

-- FULL OUTER JOIN (MySQL n'a pas FULL OUTER JOIN → UNION de LEFT et RIGHT)
SELECT c.name, o.reference
FROM customers c LEFT JOIN orders o ON o.customer_id = c.id
UNION
SELECT c.name, o.reference
FROM customers c RIGHT JOIN orders o ON o.customer_id = c.id;
```

---

### Q2 : Comment optimiser une requête lente ?

**Réponse :**

Démarche en 5 étapes :

1. **EXPLAIN / EXPLAIN ANALYZE** : identifier le type d'accès (ALL = scan complet → problème)
2. **Index manquant** : sur les colonnes dans WHERE, JOIN, ORDER BY
3. **Requête mal écrite** : fonctions sur colonnes indexées, SELECT *, sous-requêtes N+1
4. **Statistiques obsolètes** : `ANALYZE TABLE orders;`
5. **Architecture** : cache (Redis), partition de table, read replica

```sql
-- Avant
EXPLAIN SELECT * FROM orders WHERE LOWER(status) = 'pending';  -- type: ALL

-- Après
CREATE INDEX idx_orders_status ON orders (status);
EXPLAIN SELECT id, reference FROM orders WHERE status = 'PENDING';  -- type: ref
```

---

### Q3 : Expliquez le problème N+1 en SQL et comment le résoudre.

**Réponse :**

```sql
-- Problème : 1 requête pour les orders + N requêtes pour les customers
SELECT * FROM orders;  -- retourne 100 orders
-- Puis pour chaque order :
SELECT * FROM customers WHERE id = ?;  -- 100 requêtes supplémentaires !

-- Solution 1 : JOIN
SELECT o.*, c.name, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id;
-- 1 seule requête

-- Solution 2 : IN clause
SELECT * FROM customers WHERE id IN (SELECT DISTINCT customer_id FROM orders);

-- Solution 3 (en Java/JPA) : JOIN FETCH ou @BatchSize
```

---

## Pièges Classiques

- `NULL` dans les conditions : `NULL != NULL` → utiliser `IS NULL` / `IS NOT NULL`
- `COUNT(*)` vs `COUNT(colonne)` : `COUNT(*)` compte toutes les lignes, `COUNT(col)` ignore les NULL
- Index non utilisé avec `LIKE '%prefix'` (wildcard au début) → index inutile
- `GROUP BY` sans `ORDER BY` → ordre de résultat non garanti
- Modifier le schéma sur une grande table sans transaction ni `pt-online-schema-change`
