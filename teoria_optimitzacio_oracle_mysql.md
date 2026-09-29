# Optimització de consultes en bases de dades: Oracle i MySQL

Guia teòrica i pràctica sobre com funciona l'optimitzador, com es llegeixen els plans d'execució i com s'utilitzen els **hints** a Oracle i MySQL. Els exemples reutilitzen les taules `clients` i `comandes` de l'script de demostració.

## Índex

1. [Com executa una consulta una base de dades](#1-com-executa-una-consulta-una-base-de-dades)
2. [Conceptes clau](#2-conceptes-clau)
3. [Oracle: l'optimitzador](#3-oracle-loptimitzador)
4. [Oracle: hints](#4-oracle-hints)
5. [MySQL: l'optimitzador](#5-mysql-loptimitzador)
6. [MySQL: hints](#6-mysql-hints)
7. [Comparativa Oracle vs MySQL](#7-comparativa-oracle-vs-mysql)
8. [Errors habituals que destrueixen el rendiment](#8-errors-habituals-que-destrueixen-el-rendiment)
9. [Metodologia d'optimització](#9-metodologia-doptimització)
10. [Resum ràpid](#10-resum-ràpid)

---

## 1. Com executa una consulta una base de dades

SQL és un llenguatge **declaratiu**: descrivim *què* volem, no *com* obtenir-ho. Del *com* se n'encarrega l'**optimitzador**.

```
  Text SQL
     │
     ▼
 ┌──────────┐   sintaxi, permisos, noms d'objectes
 │  Parser  │
 └────┬─────┘
      ▼
 ┌──────────────┐   genera molts plans possibles i n'estima el cost
 │ Optimitzador │◄── estadístiques, índexs, paràmetres, hints
 └────┬─────────┘
      ▼
 ┌────────────┐   pla d'execució escollit
 │ Executor   │──► resultat
 └────────────┘
```

L'optimitzador decideix, entre altres coses:

- **Com llegir cada taula**: full scan, per índex, etc.
- **En quin ordre unir les taules** (join order).
- **Amb quin algorisme unir-les**: nested loops, hash join, merge join.
- **Si transforma la consulta**: aplanar subconsultes, empènyer filtres cap a dins de vistes, etc.

Dues consultes que retornen el mateix resultat poden tenir plans radicalment diferents, i per tant temps molt diferents. Això és el que demostra l'script de la demo.

## 2. Conceptes clau

| Concepte | Explicació |
|---|---|
| **Optimitzador basat en cost (CBO)** | Estima el cost de cada pla candidat i tria el més barat. Tant Oracle com MySQL/InnoDB en fan servir un. |
| **Estadístiques** | Nombre de files, blocs, valors distints, valors mínim i màxim, histogrames. Són l'entrada principal de l'optimitzador: si són dolentes, el pla serà dolent. |
| **Cardinalitat** | Nombre de files que una operació estima que retornarà. Un error de cardinalitat és la causa més freqüent d'un mal pla. |
| **Selectivitat** | Fracció de files que passa un filtre (0 a 1). `WHERE sexe = 'H'` és poc selectiu; `WHERE dni = '...'` és molt selectiu. |
| **Cost** | Unitat interna que combina I/O i CPU estimats. Només serveix per comparar plans de la mateixa consulta. No és un temps. |
| **Full scan** | Es llegeix tota la taula. És ràpid per llegir molt (lectures multibloc/seqüencials) i lent per buscar poques files. |
| **Index scan** | Es busca a l'índex i després s'accedeix a la fila. Genial per pocs resultats; dolent si has de llegir un percentatge alt de la taula. |
| **Sargable** | Predicat que permet usar un índex (*Search ARGument ABLE*). `data >= X` ho és; `TO_CHAR(data) = ...` no. |
| **Índex cobridor (covering)** | L'índex conté totes les columnes que necessita la consulta i no cal llegir la taula. |
| **Hint** | Indicació que li dones a l'optimitzador per influir en el pla. Veure seccions 4 i 6. |

### Quan un índex ajuda i quan no

Un índex no és sempre millor que un full scan. Com a regla molt aproximada:

- Si la consulta llegeix **pocs percentatges** de la taula (uns pocs %), l'índex sol guanyar.
- Si en llegeix **molts** (desenes de %), el full scan sol guanyar, perquè llegir la taula seqüencialment és més barat que fer milers d'accessos aleatoris.
- Amb dades poc agrupades per la columna de l'índex (Oracle en diu *clustering factor* alt), cada fila sol viure en un bloc diferent i l'índex es torna encara menys atractiu.

Per això un hint que força un índex no sempre accelera res: pot empitjorar la consulta.

---

## 3. Oracle: l'optimitzador

### 3.1 Estadístiques

Oracle usa el paquet `DBMS_STATS`. Cal recollir-les després de canvis importants de dades.

```sql
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname    => USER,
        tabname    => 'COMANDES',
        cascade    => TRUE,                          -- també els índexs
        method_opt => 'FOR ALL COLUMNS SIZE AUTO'    -- histogrames automàtics
    );
END;
/
```

Oracle també recull estadístiques automàticament en una finestra de manteniment nocturna. Els **histogrames** ajuden quan les dades estan molt esbiaixades (per exemple, un valor que apareix al 90% de les files).

### 3.2 Veure el pla d'execució

**Opció 1: `EXPLAIN PLAN`** (pla estimat, la consulta no s'executa).

```sql
EXPLAIN PLAN FOR
SELECT COUNT(*) FROM comandes
 WHERE data_comanda >= DATE '2025-03-01'
   AND data_comanda <  DATE '2025-04-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

**Opció 2: pla real amb estadístiques d'execució** (la millor opció per diagnosticar).

```sql
SELECT /*+ GATHER_PLAN_STATISTICS */ COUNT(*)
  FROM comandes
 WHERE data_comanda >= DATE '2025-03-01'
   AND data_comanda <  DATE '2025-04-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT => 'ALLSTATS LAST'));
```

Amb `ALLSTATS LAST` es veuen, per a cada operació:

- **E-Rows**: files estimades.
- **A-Rows**: files reals.
- **Buffers**: lectures de blocs.
- **A-Time**: temps real.

Si **E-Rows i A-Rows difereixen molt**, l'optimitzador ha estimat malament: aquí és on cal investigar (estadístiques, histogrames, correlació entre columnes).

**Opció 3: `AUTOTRACE`** (SQL*Plus / SQLcl).

```sql
SET AUTOTRACE TRACEONLY EXPLAIN STATISTICS
```

Mostra el pla i estadístiques com `consistent gets` (lectures lògiques de blocs), `physical reads` i `sorts`.

### 3.3 Com llegir un pla d'Oracle

```
--------------------------------------------------------------
| Id | Operation                    | Name              | Rows |
--------------------------------------------------------------
|  0 | SELECT STATEMENT             |                   |    1 |
|  1 |  SORT AGGREGATE              |                   |    1 |
|* 2 |   INDEX RANGE SCAN           | IDX_COMANDES_DATA | 450  |
--------------------------------------------------------------
Predicate Information:
   2 - access("DATA_COMANDA">=... AND "DATA_COMANDA"<...)
```

- El pla és un **arbre**: s'executa de dins cap a fora (les operacions més indentades primer).
- **`access`** vol dir que el predicat s'usa per navegar (índex) i és eficient.
- **`filter`** vol dir que es comprova fila a fila després de llegir-la: menys eficient.

### 3.4 Mètodes d'accés

| Operació | Descripció | Quan surt |
|---|---|---|
| `TABLE ACCESS FULL` | Llegeix tots els blocs de la taula. | Sense índex útil o quan es llegeix molt. |
| `INDEX UNIQUE SCAN` | Una única entrada d'un índex únic. | `WHERE pk = :x`. |
| `INDEX RANGE SCAN` | Rang d'un índex. | Filtres per igualtat o rang. |
| `INDEX FULL SCAN` | Recorre tot l'índex en ordre. | Pot evitar un `ORDER BY`. |
| `INDEX FAST FULL SCAN` | Llegeix tot l'índex amb lectura multibloc. | Índex cobridor sense ordre. |
| `INDEX SKIP SCAN` | Usa un índex compost sense filtrar per la 1a columna. | Si la 1a columna té pocs valors. |
| `TABLE ACCESS BY INDEX ROWID` | Accedeix a la fila a partir del rowid de l'índex. | Després d'un index scan. |

### 3.5 Mètodes de join

| Mètode | Com funciona | Bo per a | Dolent per a |
|---|---|---|---|
| **Nested Loops** | Per cada fila de la taula externa, busca a la interna. | Pocs resultats a l'externa i índex a la interna. | Moltes files externes sense índex a la interna. |
| **Hash Join** | Construeix una taula hash de la taula petita i la sonda amb la gran. | Volums grans, cap índex útil, igualtats. | Joins que no són d'igualtat. |
| **Sort Merge** | Ordena tots dos costats i els fusiona. | Desigualtats (`<`, `>`), dades ja ordenades. | Volums grans desordenats. |

A la demo, el nested loops amb full scan (1A) fa **una passada completa a `comandes` per cada client**; el hash join (1B) en fa **una sola**.

### 3.6 Bind variables i *peeking*

```sql
SELECT COUNT(*) FROM comandes WHERE client_id = :id;
```

Amb variables *bind*, Oracle reutilitza el pla (menys parseig, menys memòria). Però el **bind peeking** fa que el pla es decideixi amb el primer valor vist. Si les dades estan esbiaixades (un client té el 50% de les comandes), un mateix pla pot ser bo per a un valor i pèssim per a un altre. Oracle mitiga el problema amb l'**Adaptive Cursor Sharing**, que pot generar diversos plans per a la mateixa sentència.

### 3.7 Alternatives als hints dins d'Oracle

Abans de posar hints en el codi, Oracle ofereix mecanismes per estabilitzar plans **sense tocar el SQL**:

- **SQL Plan Management (SPM)**: *baselines* que fixen plans acceptats.

  ```sql
  DECLARE n PLS_INTEGER;
  BEGIN
      n := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(sql_id => 'abcd1234efgh5');
  END;
  /
  ```

- **SQL Profiles** (amb el *SQL Tuning Advisor*): correccions d'estimacions.
- **SQL Patch**: injecta hints en una sentència sense modificar-ne el text (útil amb codi d'aplicacions que no controles).

  ```sql
  BEGIN
      DBMS_SQLDIAG.CREATE_SQL_PATCH(
          sql_id    => 'abcd1234efgh5',
          hint_text => 'FULL(o)',
          name      => 'patch_comandes');
  END;
  /
  ```

---

## 4. Oracle: hints

### 4.1 Què és un hint a Oracle

Un hint és una **directiva** que es posa en un comentari especial i que influeix en les decisions de l'optimitzador. Sintaxi:

```sql
SELECT /*+ hint1 hint2(args) */ columnes ...
```

Regles importants:

1. **Ha d'anar just després** de `SELECT`, `INSERT`, `UPDATE`, `DELETE` o `MERGE` (també a les subconsultes), en un comentari que comenci per `/*+`.
2. Es poden posar **diversos hints** separats per espais (no per comes).
3. **Si la taula té àlies, el hint ha d'usar l'àlies**, no el nom de la taula.
4. **Oracle ignora els hints invàlids sense donar cap error.** Un error de sintaxi, un nom d'índex inexistent o un hint impossible d'aplicar simplement no fan res.
5. Els hints són **indicacions forts, no ordres absolutes**: si són contradictoris o impossibles, l'optimitzador els descarta.
6. També existeix la sintaxi `--+ hint`, però és més fràgil i millor evitar-la.

### 4.2 Un hint invàlid es perd en silenci

```sql
-- MAL: la taula té àlies "o", el hint parla de "comandes" → s'ignora
SELECT /*+ INDEX(comandes idx_comandes_data) */ COUNT(*)
  FROM comandes o
 WHERE o.data_comanda >= DATE '2025-03-15';

-- BÉ
SELECT /*+ INDEX(o idx_comandes_data) */ COUNT(*)
  FROM comandes o
 WHERE o.data_comanda >= DATE '2025-03-15';
```

Des d'Oracle 19c es pot comprovar si els hints s'han aplicat amb l'informe de hints:

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT => 'TYPICAL +HINT_REPORT'));
```

L'informe llista cada hint com a usat (`Y`), no usat (`U`) o amb errors (`E`), i explica el motiu.

### 4.3 Categories de hints

#### Camí d'accés

| Hint | Efecte |
|---|---|
| `FULL(t)` | Força un full scan de la taula. |
| `INDEX(t idx)` | Força l'ús de l'índex indicat (si no s'indica, l'optimitzador tria entre els possibles). |
| `NO_INDEX(t idx)` | Prohibeix l'índex. |
| `INDEX_DESC(t idx)` | Recorre l'índex en ordre descendent. |
| `INDEX_FFS(t idx)` | Fast full scan de l'índex. |
| `INDEX_JOIN(t)` | Combina diversos índexs de la mateixa taula. |

```sql
-- Force full scan (com el test 2A de la demo)
SELECT /*+ FULL(o) */ COUNT(*), SUM(o.import)
  FROM comandes o
 WHERE o.data_comanda >= DATE '2025-03-15'
   AND o.data_comanda <  DATE '2025-03-16';

-- Force l'índex (test 2B)
SELECT /*+ INDEX(o idx_comandes_data) */ COUNT(*), SUM(o.import)
  FROM comandes o
 WHERE o.data_comanda >= DATE '2025-03-15'
   AND o.data_comanda <  DATE '2025-03-16';
```

#### Ordre dels joins

| Hint | Efecte |
|---|---|
| `LEADING(t1 t2 ...)` | Defineix l'ordre en què s'uneixen les taules. |
| `ORDERED` | Uneix les taules en l'ordre del `FROM` (antic; millor `LEADING`). |

#### Mètode de join

| Hint | Efecte |
|---|---|
| `USE_NL(t)` | Nested loops, amb `t` com a taula interna. |
| `USE_HASH(t)` | Hash join amb `t`. |
| `USE_MERGE(t)` | Sort merge amb `t`. |
| `NO_USE_NL(t)`, `NO_USE_HASH(t)`, `NO_USE_MERGE(t)` | Prohibeixen aquest mètode. |

```sql
-- Nested loops amb full scan a comandes (lent)
SELECT /*+ LEADING(c) USE_NL(o) FULL(o) */
       c.client_id, COUNT(*), SUM(o.import)
  FROM clients c
  JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;

-- Hash join (ràpid)
SELECT /*+ LEADING(c) USE_HASH(o) FULL(o) */
       c.client_id, COUNT(*), SUM(o.import)
  FROM clients c
  JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;
```

`LEADING(c)` diu que primer va `clients`; `USE_NL(o)` o `USE_HASH(o)` diu com s'hi uneix `comandes`.

#### Objectiu de l'optimitzador

| Hint | Efecte |
|---|---|
| `ALL_ROWS` | Optimitza el **temps total** per obtenir totes les files (processos batch). |
| `FIRST_ROWS(n)` | Optimitza el temps per obtenir les **primeres n files** (pantalles, paginació). |

```sql
SELECT /*+ FIRST_ROWS(10) */ *
  FROM comandes
 ORDER BY data_comanda DESC
 FETCH FIRST 10 ROWS ONLY;
```

#### Transformacions de consulta

| Hint | Efecte |
|---|---|
| `UNNEST` / `NO_UNNEST` | Aplanar o no una subconsulta convertint-la en join. |
| `MERGE` / `NO_MERGE` | Fusionar o no una vista en línia amb la consulta externa. |
| `PUSH_PRED` / `NO_PUSH_PRED` | Empènyer o no un predicat cap a dins d'una vista. |
| `NO_QUERY_TRANSFORMATION` | Desactiva les transformacions. |

```sql
-- Evitar que Oracle converteixi la subconsulta en join
SELECT c.nom
  FROM clients c
 WHERE EXISTS (SELECT /*+ NO_UNNEST */ 1
                 FROM comandes o
                WHERE o.client_id = c.client_id
                  AND o.import > 900);
```

#### Paral·lelisme, escriptura i altres

| Hint | Efecte |
|---|---|
| `PARALLEL(t n)` | Executa la lectura amb `n` processos paral·lels. |
| `NO_PARALLEL(t)` | Desactiva el paral·lelisme. |
| `APPEND` | *Direct-path insert*: escriu per damunt del *high-water mark*, sense passar pel buffer cache (càrregues massives). |
| `RESULT_CACHE` | Desa el resultat a la memòria de la instància i el reutilitza. |
| `DYNAMIC_SAMPLING(t n)` | Mostreig en temps de parseig quan no hi ha estadístiques fiables. |
| `GATHER_PLAN_STATISTICS` | Recull estadístiques reals per a `DBMS_XPLAN` (`ALLSTATS LAST`). |

```sql
SELECT /*+ PARALLEL(o 4) */ SUM(import) FROM comandes o;

INSERT /*+ APPEND */ INTO comandes_arxiu
SELECT * FROM comandes WHERE data_comanda < DATE '2024-01-01';
```

### 4.4 Quan usar hints (i quan no) a Oracle

**Bons usos:**

- **Diagnosticar**: provar què passaria amb un altre pla per entendre el problema.
- **Estabilitzar** un pla crític que es degrada per estadístiques canviants, sabent que és una solució temporària.
- **Tancar la porta** a plans reconeguts com a dolents (`NO_INDEX`, `NO_USE_NL`).
- **Tasques especials**: `APPEND`, `PARALLEL`, `RESULT_CACHE`.

**Inconvenients:**

- Són **fràgils**: el que és òptim avui pot no ser-ho quan les dades creixin o canviï la distribució.
- **Ocupen el codi**: cal mantenir-los i recordar per què hi són.
- **Impedeixen que l'optimitzador s'adapti**: si millores les estadístiques o afegeixes un índex, un hint rígid pot impedir aprofitar-ho.
- Després d'una migració o actualització de versió, un hint pot tornar-se perjudicial.

> **Regla d'or:** primer estadístiques i índexs; després reescriure la consulta; el hint, com a últim recurs.

---

## 5. MySQL: l'optimitzador

Es fa referència a **MySQL 8.0 / 8.4 amb InnoDB**. Algunes coses no existeixen a versions anteriors.

### 5.1 Estadístiques

InnoDB manté estadístiques a nivell de taula i d'índex (nombre de files aproximat, **cardinalitat** dels índexs) obtingudes per mostreig d'unes poques pàgines. Per actualitzar-les:

```sql
ANALYZE TABLE comandes;
```

Des de MySQL 8.0 també hi ha **histogrames** per a columnes sense índex:

```sql
ANALYZE TABLE comandes UPDATE HISTOGRAM ON import, descripcio WITH 100 BUCKETS;
```

Els histogrames ajuden l'optimitzador a estimar selectivitats de filtres sobre columnes sense índex.

El model de cost de MySQL es basa en constants configurables que es troben a les taules `mysql.server_cost` i `mysql.engine_cost`.

### 5.2 Veure el pla d'execució

```sql
-- Pla estimat en format taula
EXPLAIN SELECT COUNT(*) FROM comandes
 WHERE data_comanda >= '2025-03-15' AND data_comanda < '2025-03-16';

-- Pla en arbre (més llegible a 8.0)
EXPLAIN FORMAT=TREE SELECT ...;

-- Pla en JSON, amb costos
EXPLAIN FORMAT=JSON SELECT ...;

-- Pla REAL: executa la consulta i mostra temps i files reals (8.0.18+)
EXPLAIN ANALYZE SELECT ...;
```

`EXPLAIN ANALYZE` és l'equivalent MySQL d'`ALLSTATS LAST`: compara files estimades i reals i dona temps per operació.

### 5.3 Llegir la sortida d'`EXPLAIN`

Columnes principals:

| Columna | Significat |
|---|---|
| `id` | Número de `SELECT` dins la consulta. |
| `select_type` | `SIMPLE`, `PRIMARY`, `SUBQUERY`, `DERIVED`, `UNION`... |
| `table` | Taula de la fila. |
| `type` | **Tipus d'accés** (la columna més important). |
| `possible_keys` | Índexs que es podrien usar. |
| `key` | Índex realment triat. |
| `key_len` | Bytes de l'índex que s'usen (indica quantes columnes d'un índex compost). |
| `ref` | Què es compara amb l'índex. |
| `rows` | Files estimades que caldrà examinar. |
| `filtered` | Percentatge estimat de files que passen el filtre. |
| `Extra` | Informació addicional. |

**Valors de `type`, de millor a pitjor:**

| `type` | Descripció |
|---|---|
| `system` / `const` | Com a màxim una fila (clau primària o única amb valor constant). |
| `eq_ref` | Una fila per cada combinació anterior (join per PK/únic). |
| `ref` | Diverses files amb el mateix valor d'índex no únic. |
| `range` | Un rang d'un índex (`BETWEEN`, `<`, `>`, `IN`). |
| `index` | Recorre **tot l'índex** (millor que la taula, però continua sent complet). |
| `ALL` | **Full table scan.** |

**Valors freqüents d'`Extra`:**

| `Extra` | Significat |
|---|---|
| `Using index` | Índex cobridor: no cal llegir la taula (bo). |
| `Using where` | S'aplica un filtre després de llegir. |
| `Using index condition` | *Index Condition Pushdown*: filtra dins de l'índex. |
| `Using filesort` | Ordena en memòria o disc (no vol dir necessàriament un fitxer). Costós amb molts resultats. |
| `Using temporary` | Crea una taula temporal (per `GROUP BY`, `DISTINCT`, etc.). |
| `Using join buffer (hash join)` | Join amb hash join (8.0.18+). |

### 5.4 Algorismes de join a MySQL

- **Nested Loop Join**: l'algorisme clàssic; per cada fila de la taula externa, busca a la interna (millor amb índex).
- **Block Nested Loop (BNL)**: versió sense índex que agrupa files en un buffer.
- **Hash Join**: des de **MySQL 8.0.18**, substitueix el BNL per a joins d'igualtat sense índex útil.
- **Batched Key Access (BKA)**: llegeix les claus per lots i accedeix a l'índex de la taula interna de manera més eficient (desactivat per defecte).

MySQL **no té sort-merge join** de manera general.

### 5.5 Diferències importants respecte d'Oracle

- L'optimitzador de MySQL és **més senzill** i pren decisions amb menys informació estadística; per això algunes decisions dolentes són més freqüents en consultes complexes amb molts joins.
- No hi ha equivalent directe de **SQL Plan Management**: no es poden fixar plans. Les solucions són hints, reescriure la consulta o (a 8.0) el connector `Rewriter`, que reescriu sentències.
- El **query cache** va ser eliminat a MySQL 8.0.

---

## 6. MySQL: hints

MySQL té **dues famílies** de hints. Conviuen, i cal saber-ne la diferència.

### 6.1 Index hints (antics, dins del `FROM`)

Van a continuació del nom de la taula:

```sql
SELECT ...
  FROM comandes o USE INDEX (idx_comandes_data)
 WHERE ...;
```

| Hint | Efecte |
|---|---|
| `USE INDEX (idx)` | Suggereix que només es considerin aquests índexs. L'optimitzador pot preferir un full scan. |
| `FORCE INDEX (idx)` | Com `USE INDEX`, però un full scan es considera **molt car**: només el fa si l'índex no és aplicable. |
| `IGNORE INDEX (idx)` | Prohibeix aquest índex. |

Es poden restringir a una fase: `FOR JOIN`, `FOR ORDER BY`, `FOR GROUP BY`.

```sql
-- Obligar l'ús de l'índex per a la cerca
SELECT COUNT(*), SUM(o.import)
  FROM comandes o FORCE INDEX (idx_comandes_data)
 WHERE o.data_comanda >= '2025-03-15'
   AND o.data_comanda <  '2025-03-16';

-- Ignorar l'índex (equival a forçar un full scan en aquest cas)
SELECT COUNT(*), SUM(o.import)
  FROM comandes o IGNORE INDEX (idx_comandes_data)
 WHERE o.data_comanda >= '2025-03-15'
   AND o.data_comanda <  '2025-03-16';

-- Només per ordenar
SELECT * FROM comandes o USE INDEX FOR ORDER BY (idx_comandes_data)
 ORDER BY o.data_comanda LIMIT 10;
```

**`STRAIGHT_JOIN`**, que és un modificador del `SELECT`, força l'**ordre de join** com al `FROM` (taula de l'esquerra primer):

```sql
SELECT STRAIGHT_JOIN c.client_id, COUNT(*), SUM(o.import)
  FROM clients c
  JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;
```

### 6.2 Optimizer hints (estil Oracle, `/*+ ... */`)

Existeixen des de MySQL 5.7 i s'han ampliat a la 8.0. Es posen just després de `SELECT`, `INSERT`, `UPDATE`, `DELETE` o `REPLACE`:

```sql
SELECT /*+ INDEX(o idx_comandes_data) */ ...
```

Diferències amb els index hints:

- Poden actuar a **nivell de taula, de bloc de consulta o global**.
- Fan servir **àlies** o el nom de la taula.
- Si un hint no es reconeix o no s'aplica, MySQL **no dona error, sinó un avís** que es pot consultar amb `SHOW WARNINGS`.
- Es poden posar **noms de bloc** (`QB_NAME`) per dirigir el hint a una subconsulta: `@nom_bloc`.

#### Accés a taules i índexs

| Hint | Efecte |
|---|---|
| `INDEX(t idx)` / `NO_INDEX(t idx)` | Equivalent a `FORCE INDEX` / `IGNORE INDEX`. |
| `JOIN_INDEX(t idx)` / `NO_JOIN_INDEX` | Índex per a l'accés de join. |
| `GROUP_INDEX(t idx)` / `NO_GROUP_INDEX` | Índex per al `GROUP BY`. |
| `ORDER_INDEX(t idx)` / `NO_ORDER_INDEX` | Índex per a l'`ORDER BY`. |
| `INDEX_MERGE(t idx...)` / `NO_INDEX_MERGE` | Combinar índexs. |
| `SKIP_SCAN(t idx)` / `NO_SKIP_SCAN` | *Skip scan* d'índexs compostos (8.0.13+). |
| `NO_RANGE_OPTIMIZATION(t idx)` | Desactiva l'accés per rang. |

#### Ordre i mètode de join

| Hint | Efecte |
|---|---|
| `JOIN_ORDER(t1, t2)` | Ordre exacte dels joins. |
| `JOIN_PREFIX(t1, t2)` | Aquestes taules van primer. |
| `JOIN_SUFFIX(t1, t2)` | Aquestes taules van al final. |
| `JOIN_FIXED_ORDER` | Ordre del `FROM` (equival a `STRAIGHT_JOIN`). |
| `BNL(t)` / `NO_BNL(t)` | Permet o prohibeix el BNL i, des de 8.0.20, també el **hash join**. |
| `BKA(t)` / `NO_BKA(t)` | Permet o prohibeix el Batched Key Access. |

```sql
SELECT /*+ JOIN_ORDER(c, o) */ c.client_id, COUNT(*), SUM(o.import)
  FROM clients c
  JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;

-- Prohibir el hash join a comandes (torna al nested loop)
SELECT /*+ NO_BNL(o) */ c.client_id, COUNT(*), SUM(o.import)
  FROM clients c
  JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id;
```

#### Subconsultes i transformacions

| Hint | Efecte |
|---|---|
| `SEMIJOIN(...)` / `NO_SEMIJOIN(...)` | Converteix (o no) `IN`/`EXISTS` en semijoin, amb estratègies com `MATERIALIZATION`, `LOOSESCAN`, `FIRSTMATCH`, `DUPSWEEDOUT`. |
| `SUBQUERY(...)` | Estratègia per a subconsultes que no són semijoin: `INTOEXISTS` o `MATERIALIZATION`. |
| `MERGE(t)` / `NO_MERGE(t)` | Fusionar o materialitzar una vista o *derived table*. |
| `DERIVED_CONDITION_PUSHDOWN(t)` | Empènyer condicions a una *derived table* (8.0.22+). |

```sql
SELECT /*+ SEMIJOIN(@sub MATERIALIZATION) */ c.nom
  FROM clients c
 WHERE c.client_id IN (SELECT /*+ QB_NAME(sub) */ o.client_id
                         FROM comandes o WHERE o.import > 900);
```

#### Hints que no toquen el pla

| Hint | Efecte |
|---|---|
| `MAX_EXECUTION_TIME(ms)` | Cancel·la el `SELECT` si triga més de `ms` mil·lisegons. |
| `SET_VAR(var=valor)` | Canvia una variable de sessió **només per a aquesta sentència**. |
| `RESOURCE_GROUP(nom)` | Executa la sentència dins d'un grup de recursos. |

```sql
-- Protegir el servidor de consultes descontrolades
SELECT /*+ MAX_EXECUTION_TIME(2000) */ COUNT(*) FROM comandes;

-- Més memòria per ordenar només en aquesta consulta
SELECT /*+ SET_VAR(sort_buffer_size = 16777216) */ *
  FROM comandes
 ORDER BY import DESC
 LIMIT 100;
```

### 6.3 Controlar l'optimitzador amb `optimizer_switch`

A part dels hints, MySQL permet activar o desactivar estratègies a nivell de sessió:

```sql
SET SESSION optimizer_switch = 'index_merge=off,hash_join=on';
SHOW VARIABLES LIKE 'optimizer_switch';
```

També es pot fer per sentència amb `SET_VAR(optimizer_switch = '...')`.

### 6.4 Comprovar que un hint s'ha aplicat

```sql
EXPLAIN SELECT /*+ INDEX(o idx_inexistent) */ COUNT(*)
  FROM comandes o WHERE o.data_comanda >= '2025-03-15';

SHOW WARNINGS;
```

Si el hint no és vàlid, `SHOW WARNINGS` explicarà el problema. Per veure com ha reescrit i optimitzat la consulta MySQL, `SHOW WARNINGS` després d'un `EXPLAIN` també mostra la consulta transformada.

### 6.5 Adaptar la demo a MySQL

```sql
CREATE TABLE clients (
    client_id  INT PRIMARY KEY,
    nom        VARCHAR(60) NOT NULL,
    ciutat     VARCHAR(40) NOT NULL
);

CREATE TABLE comandes (
    comanda_id    INT PRIMARY KEY,
    client_id     INT NOT NULL,
    data_comanda  DATE NOT NULL,
    import        DECIMAL(10,2) NOT NULL,
    descripcio    VARCHAR(100),
    CONSTRAINT fk_com_cli FOREIGN KEY (client_id) REFERENCES clients(client_id)
);

CREATE INDEX idx_comandes_data ON comandes(data_comanda);
```

Generació de dades amb una CTE recursiva (8.0):

```sql
SET SESSION cte_max_recursion_depth = 1000000;

INSERT INTO clients
WITH RECURSIVE n AS (SELECT 1 AS i UNION ALL SELECT i + 1 FROM n WHERE i < 200)
SELECT i, CONCAT('Client ', LPAD(i, 4, '0')), 'Barcelona' FROM n;

INSERT INTO comandes
WITH RECURSIVE n AS (SELECT 1 AS i UNION ALL SELECT i + 1 FROM n WHERE i < 500000)
SELECT i,
       1 + FLOOR(RAND() * 200),
       DATE_ADD('2023-01-01', INTERVAL FLOOR(RAND() * 1096) DAY),
       ROUND(5 + RAND() * 995, 2),
       CONCAT('Comanda de prova ', i)
FROM n;

ANALYZE TABLE clients, comandes;
```

Parell de consultes amb el mateix text i diferent hint:

```sql
-- Lenta: sense l'índex
SELECT /*+ NO_INDEX(o idx_comandes_data) */ COUNT(*), SUM(o.import)
  FROM comandes o
 WHERE o.data_comanda >= '2025-03-15' AND o.data_comanda < '2025-03-16';

-- Ràpida: amb l'índex
SELECT /*+ INDEX(o idx_comandes_data) */ COUNT(*), SUM(o.import)
  FROM comandes o
 WHERE o.data_comanda >= '2025-03-15' AND o.data_comanda < '2025-03-16';

-- Comparar plans i temps reals
EXPLAIN ANALYZE SELECT /*+ NO_INDEX(o idx_comandes_data) */ ...;
EXPLAIN ANALYZE SELECT /*+ INDEX(o idx_comandes_data) */ ...;
```

> Nota: a MySQL, `NO_INDEX` no *força* el full scan, sinó que prohibeix aquest índex; com que a la demo no n'hi ha cap altre útil, l'optimitzador fa un full scan.

---

## 7. Comparativa Oracle vs MySQL

| Aspecte | Oracle | MySQL (InnoDB) |
|---|---|---|
| **Optimitzador** | CBO molt sofisticat amb moltes transformacions. | CBO més simple, menys transformacions. |
| **Recollida d'estadístiques** | `DBMS_STATS` (automàtica + manual), histogrames avançats. | `ANALYZE TABLE`, estadístiques persistents per mostreig, histogrames (8.0). |
| **Veure el pla** | `EXPLAIN PLAN`, `DBMS_XPLAN`, `AUTOTRACE`. | `EXPLAIN`, `EXPLAIN FORMAT=TREE/JSON`, `EXPLAIN ANALYZE`. |
| **Pla amb dades reals** | `GATHER_PLAN_STATISTICS` + `ALLSTATS LAST`. | `EXPLAIN ANALYZE` (8.0.18+). |
| **Mètodes de join** | Nested loops, hash, sort merge. | Nested loop, hash join (8.0.18+), BKA. |
| **Sintaxi dels hints** | `/*+ ... */` després del verb. | `/*+ ... */` (optimizer hints) **i** `USE/FORCE/IGNORE INDEX` (index hints). |
| **Hint amb error** | Ignorat en silenci (informe des de 19c). | Ignorat amb un avís (`SHOW WARNINGS`). |
| **Nombre de hints** | Centenars. | Desenes. |
| **Fixar plans sense tocar el codi** | SQL Plan Management, SQL Patch, SQL Profile. | No hi ha equivalent directe; `Rewriter` o modificar la consulta. |
| **Paral·lelisme d'una consulta** | `PARALLEL` (query paral·lela). | Molt limitat (algunes operacions d'índex i `COUNT(*)` en 8.0.14+). |
| **Bind variables / plans compartits** | Cursor compartit, bind peeking, ACS. | Sentències preparades; el pla es decideix a cada execució. |

---

## 8. Errors habituals que destrueixen el rendiment

### 8.1 Funcions sobre columnes indexades

```sql
-- MAL: no es pot usar l'índex de data_comanda
WHERE TO_CHAR(data_comanda, 'YYYY-MM') = '2025-03'      -- Oracle
WHERE DATE_FORMAT(data_comanda, '%Y-%m') = '2025-03'    -- MySQL
WHERE YEAR(data_comanda) = 2025                         -- MySQL

-- BÉ: comparar la columna directament amb un rang
WHERE data_comanda >= DATE '2025-03-01' AND data_comanda < DATE '2025-04-01'   -- Oracle
WHERE data_comanda >= '2025-03-01'      AND data_comanda < '2025-04-01'        -- MySQL
```

A Oracle també es pot crear un **índex basat en funció** (`CREATE INDEX ... ON t (TO_CHAR(data,'YYYY-MM'))`); a MySQL 8.0.13+ hi ha índexs funcionals amb sintaxi similar.

### 8.2 Conversions implícites de tipus

```sql
-- client_id és numèric i el comparem amb text: pot forçar conversions
-- i inutilitzar l'índex (a Oracle, si la conversió cau sobre la columna)
WHERE codi_text = 12345        -- MAL: la columna és VARCHAR2 i el valor un número
WHERE codi_text = '12345'      -- BÉ
```

### 8.3 `LIKE` amb comodí al principi

```sql
WHERE nom LIKE '%garcia'   -- no pot usar un índex normal
WHERE nom LIKE 'garcia%'   -- sí (rang)
```

### 8.4 Subconsultes correlacionades a la llista de columnes

```sql
-- Es re-executa per cada fila de clients (test 1A de la primera demo)
SELECT c.nom,
       (SELECT COUNT(*) FROM comandes o WHERE o.client_id = c.client_id)
  FROM clients c;

-- Millor: un únic join amb agregació
SELECT c.nom, COUNT(o.comanda_id)
  FROM clients c LEFT JOIN comandes o ON o.client_id = c.client_id
 GROUP BY c.client_id, c.nom;
```

### 8.5 `NOT IN` amb valors nuls

`NOT IN` retorna cap fila si la subconsulta conté algun `NULL`, i sol optimitzar-se pitjor. Preferir `NOT EXISTS`.

### 8.6 `SELECT *`

Llegeix columnes innecessàries, impedeix índexs cobridors i augmenta la xarxa i la memòria. Selecciona només el que necessites.

### 8.7 Paginació amb `OFFSET` gran

```sql
SELECT * FROM comandes ORDER BY comanda_id LIMIT 20 OFFSET 1000000;   -- MySQL
```

El motor ha de llegir i descartar un milió de files. Alternativa: **paginació per clau** (*keyset pagination*):

```sql
SELECT * FROM comandes WHERE comanda_id > :ultim_id ORDER BY comanda_id LIMIT 20;
```

### 8.8 Falta d'índex a les claus foranes

A Oracle, una FK sense índex pot provocar bloquejos a la taula filla i joins lents. A MySQL/InnoDB, l'índex de FK es crea automàticament.

### 8.9 Massa índexs

Cada índex accelera lectures però **alenteix** `INSERT`, `UPDATE` i `DELETE` i ocupa espai. No indexis totes les columnes «per si de cas».

### 8.10 Índexs compostos amb l'ordre de columnes equivocat

Un índex `(client_id, data_comanda)` serveix per a `WHERE client_id = X` i per a `WHERE client_id = X AND data_comanda ...`, però **no** és eficient per a `WHERE data_comanda = ...` sol. Posa primer les columnes amb filtre d'igualtat i després les de rang.

---

## 9. Metodologia d'optimització

1. **Mesurar, no endevinar.** Identifica la consulta concreta que va lenta (AWR/ASH a Oracle; `slow query log` i `performance_schema` / `sys` a MySQL).
2. **Obtenir el pla real** (`ALLSTATS LAST` / `EXPLAIN ANALYZE`) i buscar:
   - Operacions amb un cost o temps desproporcionat.
   - Diferències grans entre files **estimades** i **reals**.
   - Full scans on esperaves un índex, o l'invers.
3. **Comprovar les estadístiques**: estan actualitzades? Cal un histograma?
4. **Revisar la consulta**: funcions sobre columnes, conversions, subconsultes correlacionades, `SELECT *`.
5. **Revisar els índexs**: n'hi ha un que cobreixi els filtres i els joins? Es pot fer cobridor?
6. **Provar amb un hint** per confirmar la hipòtesi («i si forcés aquest índex o aquest join?»). Compara temps **i** lectures de blocs.
7. **Decidir la solució definitiva**: preferentment corregir estadístiques, índexs o consulta; si cal, hint o mecanisme de fixació de plans.
8. **Validar** amb dades de volum realista, i amb valors diferents dels paràmetres (dades esbiaixades!).
9. **Documentar** per què hi ha un hint, i tornar-lo a revisar amb els canvis de versió o de volum.

> Prova sempre amb un volum de dades semblant al de producció: una consulta pot ser instantània amb 1.000 files i inviable amb 100 milions.

---

## 10. Resum ràpid

- L'optimitzador tria el pla **estimant costos amb estadístiques**; estadístiques dolentes donen plans dolents.
- **El pla és el que explica el temps**: llegeix-lo amb `DBMS_XPLAN` (Oracle) o `EXPLAIN ANALYZE` (MySQL) i compara files estimades amb reals.
- **Full scan no és dolent i índex no és bo per definició**: depèn del percentatge de files que llegeixes.
- Un **hint** és una influència sobre l'optimitzador, no una ordre; si és invàlid o impossible, s'ignora (Oracle en silenci, MySQL amb avís).
- **Oracle:** `/*+ FULL(t) INDEX(t idx) LEADING(...) USE_NL(t) USE_HASH(t) FIRST_ROWS(n) PARALLEL(t n) APPEND ... */`; les alternatives sense tocar el codi són SPM, SQL Profile i SQL Patch.
- **MySQL:** `/*+ INDEX(t idx) NO_INDEX JOIN_ORDER(...) NO_BNL MAX_EXECUTION_TIME(ms) SET_VAR(...) */` més els clàssics `USE/FORCE/IGNORE INDEX` i `STRAIGHT_JOIN`.
- Els hints són **l'últim recurs**: primer estadístiques, índexs i consulta ben escrita.
- Els mals hàbits més cars: funcions sobre columnes indexades, conversions implícites, `LIKE '%x'`, subconsultes correlacionades, `SELECT *` i `OFFSET` gran.
