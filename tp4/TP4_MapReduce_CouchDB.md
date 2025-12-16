# TP n°4 — MapReduce avec CouchDB (Partie 1)

---

## 0) Objectif du TP
L’objectif de cette première partie est de :
- comprendre le concept de **MapReduce** (en mode centralisé),
- comprendre le fonctionnement de **CouchDB** (base NoSQL orientée documents),
- appliquer MapReduce à un modèle inspiré du **web graph** (matrice de liens entre pages),
- proposer des traitements MapReduce pour :
  1) représenter une matrice **M** très grande,
  2) calculer la norme d’un vecteur-ligne,
  3) calculer le produit matrice–vecteur **phi = M × W**.

---

## 1) Présentation de CouchDB (rappels)
CouchDB est un système de gestion de base de données **documentaire** :
- on stocke des **documents JSON** (pas de tables relationnelles),
- on communique avec la base via une **API REST** (HTTP),
- on peut utiliser n’importe quel client HTTP (curl, Postman, navigateur, etc.).
### 1.1 Base NoSQL orientée documents
CouchDB est une base **NoSQL orientée documents** :
- les données sont stockées sous forme de **documents JSON** ;
- chaque document possède un identifiant unique `_id` (et généralement `_rev`) ;
- la structure est flexible (pas de schéma SQL strict).

Exemple de document :
```json
{
  "_id": "page:42",
  "type": "page",
  "pageIndex": 42,
  "title": "Exemple",
  "links": [
    {"to": 3, "w": 0.5},
    {"to": 10, "w": 1.2}
  ]
}
```

### 1.2 Accès via HTTP (API REST)
CouchDB se pilote via HTTP :
- création de base, insertion/lecture/suppression de documents ;
- interrogation de **vues MapReduce** (“views”).

Exemples d’opérations courantes :
- `PUT /ma_base`  (création d’une base)
- `POST /ma_base` (insertion d’un document)
- `GET /ma_base/docId` (lecture)
- `DELETE /ma_base/docId?rev=...` (suppression)
- `GET /ma_base/_design/ddoc/_view/ma_view` (lecture d’une view)

CouchDB écoute par défaut sur le port **5984**.
Avec Docker, il faut penser aux **volumes** (sinon les données peuvent être perdues si le conteneur est supprimé).
Exemple de commande (à adapter) :
```bash
docker run -it --name couchdbdemo  -e COUCHDB_USER=youssef  -e COUCHDB_PASSWORD=samir  -p 5984:5984 couchdb
```

## 1.3 Interface graphique (Fauxton)
CouchDB propose un client graphique accessible dans le navigateur :
- URL typique : `http://localhost:5984/_utils`
- puis login/mot de passe (ceux définis au lancement).

On peut explorer les bases et documents dans cette interface :
- affichage “document JSON”,
- affichage “table”,
- pagination, etc.

## 1.4 Tests REST avec curl (ce qui est fait dans cette partie)

### 1.4.1 Test de base (GET serveur)
```bash
curl -X GET http://login:password@localhost:5984/
```
Cela renvoie une réponse JSON qui prouve que CouchDB répond.

### 1.4.2 Créer une base (PUT)
Exemple : créer une base `films` :
```bash
curl -X PUT http://login:password@localhost:5984/films
```
Si tout va bien, on reçoit un JSON avec `"ok": true`.

### 1.4.3 Lire les infos d’une base (GET)
```bash
curl -X GET http://login:password@localhost:5984/films
```

### 1.4.4 Insérer un document avec un id choisi (PUT)
Exemple : document `doc` :
```bash
curl -X PUT http://login:password@localhost:5984/films/doc  -H "Content-Type: application/json"  -d '{"nom":"youssef","age":23}'
```
Résultat : CouchDB renvoie `ok`, et ajoute aussi un champ de version `_rev`.

### 1.4.5 Conflit si on réutilise le même id
Si on refait `PUT` sur le même id sans gérer `_rev`, CouchDB peut répondre un **conflit** (conflict).
Conclusion simple : **un `_id` doit rester unique** (ou il faut gérer les révisions).

### 1.4.6 Insérer un document depuis un fichier JSON (POST)
On peut envoyer un fichier JSON :
```bash
curl -X POST http://login:password@localhost:5984/films  -H "Content-Type: application/json"  -d @mon_fichier.json
```
Remarques: 
- Si le document ne contient pas `_id`, CouchDB peut générer un identifiant automatiquement.
- CouchDB accepte des documents JSON **hétérogènes** (pas de schéma imposé comme en SQL),
- mais en pratique, une base doit quand même être conçue avec une **structure logique** (un schéma “implicite”),
- sinon l’application devra gérer plein de cas différents, et ça peut devenir compliqué / moins performant.


## 1.5 Insertion en masse (bulk) : importer une collection de films
Dans la , il est montré qu’au lieu d’insérer document par document, on peut importer en masse :
- on utilise l’endpoint **_bulk_docs**,
- et on envoie un JSON de la forme :
 ```json
 { "docs": [ {...}, {...}, ... ] }
 ```

Exemple :
```bash
curl -X POST http://login:password@localhost:5984/films/_bulk_docs  -H "Content-Type: application/json"  -d @films_couchdb.json
```
Après import, on vérifie dans Fauxton qu’on a bien beaucoup de documents (ex : centaines de films).


## 1.6 Documents imbriqués (et différence avec le relationnel)
CouchDB permet des documents **imbriqués** (JSON dans JSON), par exemple :
- un film peut contenir un tableau d’acteurs directement dans le document.

En base relationnelle, on évite ça (1NF) et on sépare souvent en tables (Film, Acteur, etc.) avec des clés étrangères.
Dans CouchDB, on peut “tout mettre dans un seul document”, mais ça peut créer :
- de la **redondance** (le même acteur répété dans plusieurs films),
- donc un risque d’**incohérence** (nom écrit différemment d’un document à l’autre).

### 1.7 Design Documents et Views
Les traitements MapReduce dans CouchDB sont définis dans des **Design Documents** (`_design/...`).
Un design document contient une ou plusieurs **views** :
- une fonction `map(doc)` qui parcourt tous les documents,
- éventuellement une fonction `reduce(keys, values, rereduce)` qui agrège.

> Important : CouchDB indexe les résultats des views (index B-tree) et met à jour l’index quand les documents changent.

---

## 2) MapReduce : principe
![Figure 28](images4/img28.png)

### 2.1 Phase Map
La fonction `map(doc)` :
- reçoit **un document à la fois**,
- peut produire des paires **(clé, valeur)** avec :
  `emit(key, value)`.

### 2.2 Phase Reduce
La fonction `reduce(keys, values, rereduce)` :
- regroupe toutes les valeurs qui partagent la même clé,
- calcule une agrégation (somme, compte, min, max…).

En CouchDB :
- `reduce` doit être **associatif** et adapté au **rereduce** (agrégation de résultats intermédiaires).  
- Exemple typique : somme, compte, min, max.

---

## 3) Contexte du sujet : matrice de liens du Web
On considère une matrice **M** de dimension **N × N** :
- **N** = très grand nombre de pages web,
- **M_ij** = poids (importance) du lien de la page **P_i** vers la page **P_j**.

Comme **N** est énorme, une matrice dense est impossible à stocker.
On utilise une représentation **creuse (sparse)** : stocker seulement les coefficients non nuls (liens existants).

---

# Partie A — Réponses aux questions du sujet

## Question 1 — Modèle sous forme de documents structurés pour représenter la matrice M

### 1) Idée générale (représentation creuse / sparse)
Chaque **ligne i** de la matrice (page `P_i`) devient un **document JSON**.  
Le document stocke uniquement la liste des liens sortants non nuls :
- destination `to = j`
- poids `w = M_ij`

### 2) Structure proposée (collection C)
La collection **C** est l’ensemble des documents “page”.

Schéma :
```json
{
  "_id": "page:i",
  "type": "page",
  "pageIndex": i,
  "links": [
    { "to": j1, "w": Mij1 },
    { "to": j2, "w": Mij2 }
  ]
}
```

- `pageIndex` représente l’indice de ligne `i`.
- `links` contient toutes les colonnes `j` où `M_ij != 0`.

### 3) Exemple concret
Supposons pour la page `P_2` :
- `M_2,1 = 0.2`
- `M_2,4 = 1.5`

Alors :
```json
{
  "_id": "page:2",
  "type": "page",
  "pageIndex": 2,
  "links": [
    { "to": 1, "w": 0.2 },
    { "to": 4, "w": 1.5 }
  ]
}
```

**Avantages :**
- on ne stocke pas les zéros ⇒ gain énorme en mémoire,
- modèle compatible avec des analyses type “PageRank-like” (liens + poids).

---

## Question 2 — MapReduce qui calcule la norme d’un vecteur-ligne à partir de la collection C

On interprète la ligne `i` comme un vecteur `V_i` de dimension `N` (poids des liens sortants).  
Norme (euclidienne / L2) :
- `||V_i|| = sqrt( v1^2 + v2^2 + ... + vN^2 )`

Dans notre modèle, les composantes non nulles sont les `w` de `links`.

### Objectif
Pour chaque page `P_i`, calculer :
- `S_i = sum_j (M_ij^2)`  
puis :
- `||V_i|| = sqrt(S_i)`

### Traitement MapReduce proposé
#### Map (1 émission par page)
On calcule `S_i` directement pour le document, puis on émet `(i, S_i)`.

```js
function (doc) {
  if (doc.type !== "page" || doc.pageIndex === undefined || !doc.links) return;

  var sumSq = 0;
  for (var k = 0; k < doc.links.length; k++) {
    var w = doc.links[k].w;
    sumSq += w * w;
  }

  emit(doc.pageIndex, sumSq);
}
```

#### Reduce (somme)
Si besoin d’agréger (cas multi-docs par page, ou sécurité), on somme :
```js
function (keys, values, rereduce) {
  return sum(values);
}
```

### Résultat
La view renvoie, pour chaque `pageIndex = i`, la valeur :
- `S_i = sum_j (M_ij^2)`

La norme est ensuite :
- `||V_i|| = sqrt(S_i)`

> Souvent, la racine carrée est appliquée côté application (post-traitement), car le reduce “sum” est plus stable/simple.

---

## Question 3 — MapReduce qui calcule phi = M × W avec W en RAM

On veut :
- `phi_i = sum_{j=1..N} (M_ij * w_j)`

Hypothèse de l’énoncé : le vecteur **W** est en RAM et accessible comme variable “statique” dans `map` et `reduce`.

### Idée
Chaque document (page `i`) contient les coefficients `M_ij` non nuls via `links`.
Donc, pour chaque lien vers `j`, on ajoute la contribution `M_ij * w_j`.

### Traitement MapReduce (version “classique” : 1 emit par contribution)
#### Map
```js
// Hypothèse TP : W accessible globalement
var W = /* [w0, w1, w2, ...] */;

function (doc) {
  if (doc.type !== "page" || doc.pageIndex === undefined || !doc.links) return;

  var i = doc.pageIndex;
  for (var k = 0; k < doc.links.length; k++) {
    var j = doc.links[k].to;
    var mij = doc.links[k].w;

    // Contribution à phi_i
    emit(i, mij * W[j]);
  }
}
```

#### Reduce (somme)
```js
function (keys, values, rereduce) {
  return sum(values);
}
```

### Variante plus efficace (1 emit par page)
On somme directement dans `map` puis on émet une seule valeur :
```js
var W = /* ... */;

function (doc) {
  if (doc.type !== "page" || doc.pageIndex === undefined || !doc.links) return;

  var acc = 0;
  for (var k = 0; k < doc.links.length; k++) {
    var j = doc.links[k].to;
    acc += doc.links[k].w * W[j];
  }
  emit(doc.pageIndex, acc);
}
```

Résultat : la view renvoie directement `phi_i` pour chaque page `i`.
![Figure 29](images4/img29.png)

---

## Commentaire de l’image (graphe orienté pondéré ↔ matrice d’adjacence)

Cette image illustre exactement le modèle utilisé dans le TP : un **web graph** représenté par un **graphe orienté pondéré** (à gauche) et sa **matrice d’adjacence M** (à droite).  
Chaque sommet (A, B, C, D, E) représente une **page web** `P_i`, et chaque flèche `i → j` représente un **lien** dont le poids correspond à `M_ij` (importance du lien). Une absence de flèche signifie `M_ij = 0`, ce qui explique que `M` est souvent **creuse (sparse)** pour de grandes valeurs de `N`.

- **Lien avec la question 1 :** la ligne `i` de la matrice (ex : ligne B) décrit tous les liens sortants de la page `P_i`. Dans le TP, on stocke cette ligne en base sous forme d’un document JSON (liste des liens non nuls).
- **Lien avec la question 2 :** une ligne `i` est vue comme un **vecteur** `V_i`. Sa norme est : `||V_i|| = sqrt( sum_j (M_ij^2) )`.
- **Lien avec la question 3 :** le produit matrice–vecteur `phi = M × W` calcule, pour chaque page `i`, la valeur : `phi_i = sum_j (M_ij * w_j)`. Cela revient à additionner (Map) les contributions `M_ij * w_j` puis à sommer (Reduce).

---

# Partie C — Analyse

## C.1 Pourquoi le modèle “sparse” est indispensable ?
- La matrice `N × N` est gigantesque.
- La majorité des coefficients sont 0 (une page ne pointe pas vers toutes les pages).
- Stocker seulement les liens non nuls réduit :
  - la taille des documents,
  - le temps de calcul (Map parcourt moins d’éléments).

## C.2 Intérêt de MapReduce ici
- Chaque page (document) peut être traitée indépendamment en `map`.
- `reduce` agrège simplement les contributions (sommes).
- Cela correspond exactement à des calculs “web graph” classiques : norme, produit, comptages.

## C.3 Limites / remarque pratique (CouchDB)
- En CouchDB, `map(doc)` n’a accès qu’au document courant.
- L’hypothèse “W en RAM accessible” est une hypothèse de TP (type MapReduce théorique).  
  En pratique, si `W` change souvent, on fait souvent ce calcul côté application (qui connaît `W`).

---

# Conclusion
Ce TP montre comment :
- représenter un graphe web (matrice de liens) dans CouchDB via des documents JSON (modèle creux),
- appliquer MapReduce (views map/reduce) pour des calculs analytiques :
  - norme de chaque ligne (page),
  - produit matrice–vecteur `M × W`.

La combinaison **modélisation sparse + agrégation MapReduce** permet de traiter efficacement des structures très grandes.
