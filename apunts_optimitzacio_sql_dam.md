# Optimització de consultes SQL (Oracle i MySQL)

Apunts per a 2n de DAM. La idea és entendre **per què una consulta pot anar lenta**, **com ho podem veure** i **què podem fer**.

---

## 1. Idea principal

Quan escrivim SQL diem **què** volem, però no **com** obtenir-ho. Del *com* se n'encarrega l'**optimitzador** de la base de dades.

> **Analogia:** l'optimitzador és com el **GPS**. Tu dius «vull anar a Girona» i ell tria la ruta. Normalment encerta, però de vegades tria una ruta pitjor perquè té informació desactualitzada.

Dues consultes que donen **el mateix resultat** poden triar camins molt diferents i trigar **1 mil·lisegon o 30 segons**.

```
  La teva consulta SQL
          │
          ▼
   ┌──────────────┐
   │ Optimitzador │ ◄── estadístiques + índexs + hints
   └──────┬───────┘
          ▼
    Pla d'execució  ──►  Resultat
```

---

## 2. Conceptes clau

| Concepte | Què vol dir | Analogia |
|---|---|---|
| **Pla d'execució** | Els passos que farà la BD per resoldre la consulta. | La ruta del GPS. |
| **Estadístiques** | Dades que la BD guarda sobre les taules (nombre de files, valors diferents...). | El mapa i l'estat del trànsit. |
| **Cost** | Nota interna que dona l'optimitzador a cada pla. Tria el més baix. | Temps estimat del trajecte. |
| **Full scan** | Llegir **tota** la taula. | Llegir tot el llibre per trobar una paraula. |
| **Índex** | Estructura auxiliar que permet trobar files sense llegir tota la taula. | L'índex alfabètic del final d'un llibre. |
| **Selectivitat** | Quin percentatge de files passa un filtre. | Buscar «un DNI» és molt selectiu; buscar «homes» no. |
| **Hint** | Una **pista** que li dones a l'optimitzador per triar un altre pla. | Dir al GPS «passa per la carretera de la costa». |

### Quan ajuda un índex?

- ✅ Busques **poques files** (uns pocs %): l'índex guanya.
- ❌ Necessites **moltes files** (desenes de %): un full scan pot ser millor.

> **Idea a recordar:** un índex no sempre és més ràpid. Si has de llegir gairebé tot el llibre, no serveix de res mirar l'índex.

---

## 3. Com veure el pla d'execució

### Oracle

```sql
EXPLAIN PLAN FOR
SELECT * FROM comandes WHERE client_id = 25;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### MySQL

```sql
EXPLAIN SELECT * FROM comandes WHERE client_id = 25;

-- Versió que executa la consulta i mostra els temps reals (MySQL 8.0.18+)
EXPLAIN ANALYZE SELECT * FROM comandes WHERE client_id = 25;
```

### Què hem de mirar en un pla?

1. **Com llegeix cada taula**:
   - Oracle: `TABLE ACCESS FULL` (full scan) o `INDEX RANGE SCAN` (índex).
   - MySQL: `type = ALL` (full scan) o `ref` / `range` (índex).
2. **Quantes files** estima que processarà.
3. **Quin tipus de join** fa servir (secció següent).

---

## 4. Els tres tipus de join

| Tipus | Com funciona | Analogia |
|---|---|---|
| **Nested Loops** | Per cada fila de la taula A, busca a la taula B. | Per cada alumne, anar a cercar el seu expedient a l'arxiu. |
| **Hash Join** | Carrega la taula petita en memòria i hi consulta la gran. | Fer una llista ordenada de tots els alumnes i anar-hi comparant. |
| **Sort Merge** | Ordena les dues taules i les recorre alhora. | Dues piles de fitxes ordenades que es recorren en paral·lel. |

- **Nested loops** és bo si hi ha **pocs resultats i un índex** a la segona taula.
- **Hash join** és bo per a **volums grans** sense índex.
- MySQL: fa nested loop i, des de la versió 8.0.18, també hash join. No fa sort merge de manera habitual.

> **Exemple de la demo:** amb 200 clients i 500.000 comandes, un nested loop **sense índex** llegeix la taula de comandes **200 vegades**. Un hash join la llegeix **una sola vegada**.

---

## 5. Hints: què són i com funcionen

Un **hint** és un comentari especial dins del SQL que suggereix a l'optimitzador com ha d'executar la consulta.

```sql
SELECT /*+ hint */ columnes ...
```

### Regles bàsiques

1. Va **just després** del `SELECT` (o `INSERT`, `UPDATE`, `DELETE`), amb la forma `/*+ ... */`.
2. Si la taula té **àlies**, el hint ha d'usar l'**àlies**.
3. Un hint és una **pista, no una ordre**: si no es pot aplicar, s'ignora.
   - **Oracle** l'ignora **sense avisar** (no dona error!).
   - **MySQL** l'ignora i mostra un avís (`SHOW WARNINGS`).
4. Sempre cal **comprovar el pla** per veure si el hint ha fet efecte.

### Hints més importants d'Oracle

| Hint | Què fa |
|---|---|
| `FULL(o)` | Força un full scan de la taula `o`. |
| `INDEX(o nom_index)` | Força l'ús d'aquest índex. |
| `NO_INDEX(o nom_index)` | Prohibeix aquest índex. |
| `USE_NL(o)` | Join amb nested loops. |
| `USE_HASH(o)` | Join amb hash join. |
| `LEADING(c)` | La taula `c` és la primera del join. |
| `FIRST_ROWS(n)` | Optimitza per obtenir ràpid les primeres `n` files. |
| `PARALLEL(o 4)` | Llegeix amb 4 processos en paral·lel. |

### Hints més importants de MySQL

| Hint | Què fa |
|---|---|
| `INDEX(o nom_index)` | Força l'ús d'aquest índex. |
| `NO_INDEX(o nom_index)` | Ignora aquest índex. |
| `JOIN_ORDER(c, o)` | Defineix l'ordre dels joins. |
| `MAX_EXECUTION_TIME(2000)` | Cancel·la el `SELECT` si triga més de 2 segons. |

MySQL també té la sintaxi antiga, que va dins del `FROM`:

```sql
SELECT * FROM comandes FORCE INDEX (idx_comandes_data) WHERE ...;   -- força l'índex
SELECT * FROM comandes IGNORE INDEX (idx_comandes_data) WHERE ...;  -- ignora l'índex
```

### Exemple 1: full scan vs índex (mateixa consulta, un hint diferent)

**Oracle**

```sql
-- Lenta: llegeix tota la taula
SELECT /*+ FULL(o) */ COUNT(*)
  FROM comandes o
 WHERE o.data_comanda >= DATE '2025-03-15'
   AND o.data_comanda <  DATE '2025-03-16';

-- Ràpida: usa l'índex
SELECT /*+ INDEX(o idx_comandes_data) */ COUNT(*)
  FROM comandes o
 WHERE o.data_comanda >= DATE '2025-03-15'
   AND o.data_comanda <  DATE '2025-03-16';
```

**MySQL**

```sql
-- Lenta
SELECT /*+ NO_INDEX(o idx_comandes_data) */ COUNT(*)
  FROM comandes o
 WHERE o.data_comanda >= '2025-03-15' AND o.data_comanda < '2025-03-16';

-- Ràpida
SELECT /*+ INDEX(o idx_comandes_data) */ COUNT(*)
  FROM comandes o
 WHERE o.data_comanda >= '2025-03-15' AND o.data_comanda < '2025-03-16';
```

### Exemple 2: mètode de join (Oracle)

```sql
-- Lent: nested loops llegint tota la taula per cada client
SELECT /*+ LEADING(c) USE_NL(o) FULL(o) */
       c.client_id, COUNT(*), SUM(o.import)
  FROM clients c JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;

-- Ràpid: hash join, la taula es llegeix una sola vegada
SELECT /*+ LEADING(c) USE_HASH(o) FULL(o) */
       c.client_id, COUNT(*), SUM(o.import)
  FROM clients c JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;
```

### Quan usar hints?

- ✅ Per **provar i entendre** per què una consulta va lenta.
- ✅ Com a **solució temporal** en un cas concret.
- ❌ **No** com a primera solució: si les dades canvien, el hint pot deixar de ser una bona idea.

> **Regla d'or:** primer estadístiques i índexs, després reescriure la consulta, i **només al final** un hint.

---

## 6. Cinc errors típics que fan lenta una consulta

**1. Funció sobre una columna amb índex**

```sql
-- Malament: no pot usar l'índex de la data
WHERE TO_CHAR(data_comanda, 'YYYY-MM') = '2025-03'      -- Oracle
WHERE DATE_FORMAT(data_comanda, '%Y-%m') = '2025-03'    -- MySQL

-- Bé: comparar amb un rang
WHERE data_comanda >= DATE '2025-03-01' AND data_comanda < DATE '2025-04-01'   -- Oracle
WHERE data_comanda >= '2025-03-01' AND data_comanda < '2025-04-01'             -- MySQL
```

**2. `LIKE` amb comodí al principi**

```sql
WHERE nom LIKE '%garcia'   -- malament: no pot usar l'índex
WHERE nom LIKE 'garcia%'   -- bé
```

**3. Subconsulta que s'executa per cada fila**

```sql
-- Malament: es repeteix una vegada per cada client
SELECT c.nom,
       (SELECT COUNT(*) FROM comandes o WHERE o.client_id = c.client_id)
  FROM clients c;

-- Bé: un sol join amb GROUP BY
SELECT c.nom, COUNT(o.comanda_id)
  FROM clients c LEFT JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id, c.nom;
```

**4. `SELECT *` quan només necessites unes quantes columnes.** Es llegeix i es transporta més dades de les necessàries.

**5. Massa índexs o cap índex.** Sense índexs les consultes van lentes; amb massa índexs van lentes les insercions i modificacions. Cal trobar l'equilibri.

---

## 7. Mètode per optimitzar una consulta (5 passos)

1. **Mesurar**: quina consulta va lenta i quant triga?
2. **Veure el pla d'execució**: hi ha un full scan on esperàvem un índex?
3. **Revisar les estadístiques i els índexs**: estan actualitzats? Falta un índex?
4. **Revisar la consulta**: hi ha una funció sobre una columna? Una subconsulta que es repeteix?
5. **Provar un hint** per confirmar la teoria, i decidir la solució definitiva.

Per actualitzar les estadístiques:

```sql
-- Oracle
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'COMANDES', cascade => TRUE);

-- MySQL
ANALYZE TABLE comandes;
```

---

## 8. Oracle vs MySQL: resum

| | Oracle | MySQL |
|---|---|---|
| Veure el pla | `EXPLAIN PLAN` + `DBMS_XPLAN` | `EXPLAIN` / `EXPLAIN ANALYZE` |
| Actualitzar estadístiques | `DBMS_STATS` | `ANALYZE TABLE` |
| Sintaxi dels hints | `/*+ ... */` | `/*+ ... */` i `FORCE`/`IGNORE INDEX` |
| Hint que no s'aplica | S'ignora sense avisar | S'ignora amb un avís |
| Tipus de joins | Nested loops, hash, sort merge | Nested loop i hash join (8.0.18+) |

---

## 9. Activitat pràctica

Amb l'script `demo_optimitzacio_hints.sql` (Oracle):

1. Executa el **Test 1A** i el **Test 1B** amb `SET TIMING ON`. Apunta els temps.
2. Mira el pla de cada un amb `EXPLAIN PLAN`. Quin tipus de join fa cada un?
3. Executa el bucle del **Test 2** i compara el `FULL SCAN` amb l'`INDEX`.
4. Escriu un hint amb un nom d'índex inventat. Què passa? Per què no dona error?
5. **Repte:** repeteix el Test 2 però busca **un any sencer** en lloc d'un sol dia. Guanya encara l'índex? Per què?

## 10. Preguntes de repàs

1. Per què dues consultes amb el mateix resultat poden tardar temps molt diferents?
2. Què és un pla d'execució i com el podem veure?
3. Quina és la diferència entre un full scan i un index scan? Quan és millor cada un?
4. Què fan les estadístiques i què passa si estan desactualitzades?
5. Quan és preferible un hash join a un nested loops?
6. Què és un hint? Per què no és una ordre?
7. Per què `WHERE YEAR(data) = 2025` no aprofita un índex sobre `data`?
8. Quin és l'ordre recomanat abans d'arribar a usar un hint?
