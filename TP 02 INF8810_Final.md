# Système de recommandation utilisant Neo4j  
**Cours :** INF8810-020 (Traitement et analyse de données massives)  

## Équipe  
- Diallo Aly Mourtada | DIAA01369609  
- Baldé Abdoulaye | BALA17269605  

**Université du Québec à Montréal**  

## Partie 1 : Détails, Analyse et Prétraitement des Données  

### Origine des données  
- **Lien :** https://raw.githubusercontent.com/MengtingWan/marketBias/refs/heads/master/data/df_modcloth.csv  
- **Source :** Les données proviennent d'un projet académique qui se concentre sur l'étude des biais dans les systèmes de recommandation.  

### Contexte des données  
**Domaine :** Vente en ligne. Le jeu de données est basé sur les produits commercialisés sur ModCloth et Amazon, qui peuvent engendrer des biais dans les recommandations (en particulier, des attributs liés à la manière dont les produits sont présentés). En résumé, le jeu de données comprend les interactions entre les utilisateurs et les produits afin de recommander des articles à un utilisateur en fonction des produits qu'il a déjà notés ou appréciés.  

### Analyse et prétraitement des données en Python pour effectuer les recommandations en Neo4J.  

#### 1. Importation des bibliothèques nécessaires  
```python  
import pandas as pd  
```  

#### 2. Chargement et exploration initiale des données  
```python  
# Chargement du fichier CSV  
df = pd.read_csv('https://raw.githubusercontent.com/MengtingWan/marketBias/refs/heads/master/data/df_electronics.csv')  

# Affichage des 5 premières lignes du DataFrame  
df.head()  
```  

#### 3. Vérification de la structure des données  
```python  
# Vérification des informations du dataframe  
df.info()  
```  

#### 4. Nettoyage des données  
```python  
# Vérification des valeurs manquantes  
df.isna().sum().reset_index(name='Nbr_valeurs_manquantes')  

# Suppression des lignes contenant des valeurs manquantes  
df = df.dropna() 

# Vérification après nettoyage  
df.isna().sum().reset_index(name='Nbr_valeurs_manquantes')  

# Réinitialisation de l'index  
df.reset_index(drop=True, inplace=True)  

# Sauvegarde des données nettoyées  
df.to_csv("df_Sample.csv", index=False)  
```  

## Partie 2 : Importation dans Neo4j et création des nœuds et des relations  
Dans notre situation, nous allons directement télécharger les données prétraitées de la première partie sur le compte GitHub (abdoulayegk) sous le nom **sample_df.csv**, après avoir effectué un nettoyage adéquat des données.  

### Structure du graphe  
Pour notre système de recommandation, nous allons créer :  

#### Nœuds :  
- Nœud **User** pour les utilisateurs  
- Nœud **Category** pour les catégories  
- Nœud **Item** pour les produits  

#### Relations :  
- Relation **RATED** entre User et Item  
- Relation **BELONGS** entre Item et Category  

### Code Cypher pour la création du graphe 

Avant de commencer la recommandation, nous allons nous assurer qu'il n'y a pas de données existantes. Si des données sont présentes, elles seront supprimées.

### Suppression de tous les noeuds

``` cypher
 // Optional: Clear existing data
MATCH (n) DETACH DELETE n;
```

### Création des nœuds User

```cypher    

// Création des nœuds User 
// Chargement du fichier CSV depuis Github  
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/abdoulayegk/INF8810/refs/heads/main/sample_df.csv' AS row 
MERGE (u:User {user_id: row.user_id})  
ON CREATE SET u.user_attr = row.user_attr;  
``` 

###  Création des nœuds Category

```cypher
// Création des nœuds Category  
// Chargement du fichier CSV depuis Github  
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/abdoulayegk/INF8810/refs/heads/main/sample_df.csv' AS row
MERGE (c:Category {Category: row.category})  
ON CREATE SET c.category = row.category;  
```

### Création des nœuds Item

```cypher
// Création des nœuds Item 
// Chargement du fichier CSV depuis Github  
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/abdoulayegk/INF8810/refs/heads/main/sample_df.csv' AS row 
MERGE (i:Item {item_id: row.item_id})  
ON CREATE SET i.brand = row.brand, i.year = row.year;  
```

### Création de la relation RATED

```cypher
// Création de la relation RATED 
// Chargement du fichier CSV depuis Github  
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/abdoulayegk/INF8810/refs/heads/main/sample_df.csv' AS row 
MATCH (u:User {user_id: row.user_id})  
MATCH (i:Item {item_id: row.item_id})  
MERGE (u)-[:RATED { rating: row.rating, timestamp: row.timestamp }]->(i);  
```

### Création de la relation BELONGS

```cypher
// Création de la relation BELONGS
MATCH (i:Item)  
MATCH (c:Category)  
MERGE (i)-[:BELONGS]->(c);  
```  

### Visualisation du graphe  
Pour voir 50 les relations entre un User qui note un produit qui appartient a un categorie :  
```cypher  
MATCH (u:User)-[r:RATED]->(i:Item)-[b:BELONGS]->(c:Category)  
RETURN u, r, i, b, c LIMIT 50;  

// Pour voir toutes les données  
MATCH (n) RETURN n;  
```  

## Partie 3 : Création du Système de Recommandation  

### 1. Approche choisie  
**Filtrage collaboratif :** Recommander à un utilisateur sélectionné parmi la liste des utilisateurs de notre DataFrame des produits appréciés par d'autres utilisateurs ayant des préférences similaires.  

### 2. Implémentation de la recommandation  

#### Étape 1 : Identifier les produits aimés par un utilisateur  
```cypher  
MATCH (u:User {user_id: "229752"})-[r:RATED]->(i:Item)  
WHERE r.rating >= "3.5"  
RETURN u, i.brand AS Produits_aimés;  
```  
Ce code va retourner Samsung dans ce cas la.

# Étape 2 : Recommandation par Filtrage collaboratif  
```cypher 
// Trouver des utilisateurs ayant évalué des articles similaires. 
MATCH (user_choisi:User {user_id:"229752"})-[r1:RATED]->(usr_item_rated:Item)<-[rx:RATED]-(autre_user:User)
// Appliquer des filtres pour restreindre les résultats aux évaluations positives et exclure l'utilisateur cible.  
WHERE r1.rating >= "3.5" AND rx.rating >= "3.5" AND user_choisi <> autre_user
// Identifier les articles évalués par les autre_user  
MATCH (autre_user)-[rec:RATED]->(Recomanded_item:Item)  
//Exclure les articles déjà évalués par user_choisi et ne garder que les bien notés(note >3.0).
WHERE NOT (user_choisi)-[:RATED]->(Recomanded_item) AND rec.rating >= "3.0"  
//Récupérer la catégorie des articles évalués par user_choisi 
MATCH (usr_item_rated)-[:BELONGS]->(ctgory_item_rated:Category)  
//Récupérer la catégorie des articles recommandé par user_choisi 
MATCH (Recomanded_item)-[:BELONGS]->(Recomanded_item_category:Category) 
// Retourner les résultats. 
RETURN user_choisi, usr_item_rated, ctgory_item_rated, Recomanded_item, Recomanded_item_category;  
```  

## Cette requête ci-haut recommande des produits en :  
1. Identifiant les produits bien notés par l'utilisateur cible (note ≥ 3.5)  
2. Trouvant d'autres utilisateurs qui ont également bien noté ces mêmes produits  
3. Recommandant les produits bien notés (note ≥ 3.0) par ces utilisateurs similaires  
4. Prenant en compte les catégories des produits pour améliorer la pertinence

# 3. Recommandation basée sur le contenu

``` cypher
// Identifier les articles évalués positivement par l'utilisateur cible
MATCH (user_choisi:User {user_id: "229752"})-[r1:RATED]->(usr_item_rated:Item)
WHERE r1.rating >= "3.5"

// Trouver des articles similaires (par catégorie) non encore évalués par l'utilisateur cible
MATCH (usr_item_rated)-[:BELONGS]->(ctgory_item_rated:Category)<-[:BELONGS]-(Recomanded_item:Item)
WHERE NOT (user_choisi)-[:RATED]->(Recomanded_item)

// S'assurer que les articles recommandés ont été bien notés par d'autres utilisateurs
MATCH (autre_user:User)-[rec:RATED]->(Recomanded_item)
WHERE rec.rating >= "3.0"

// Retourner les articles recommandés avec leurs catégories avec limite de 50
RETURN DISTINCT Recomanded_item, ctgory_item_rated AS Category
Limit 50;
```

# 4 : Recommandation Hybride
Combinez les recommandations des deux approches (collaborative et basée sur le contenu) en supprimant les doublons et en priorisant les résultats les plus pertinents.

cypher

MATCH (u1:User)-[r1:RATED]->(i:Item)
WHERE r1.rating >= "3.0"

// Filtrage collaboratif
MATCH (i)<-[r2:RATED]-(u2:User)-[:RATED]->(other:Item)
WHERE r2.rating >= "3.0" AND NOT (u1)-[:RATED]->(other)

// Basé sur le contenu
MATCH (similaire:Item)
WHERE similaire.category = i.category AND similaire.brand = i.brand AND similaire <> i

RETURN DISTINCT other AS Recommende_Collaborative, similaire AS Recommende_Contenu             

### Le code retourne deux types de recommandations :

1. Recommende_Collaborative : Articles recommandés via le filtrage collaboratif.
2. Recommende_Contenu : Articles recommandés via la recommandation basée sur le contenu.

Cela permet à l'utilisateur de bénéficier d'une approche hybride combinant la similarité des utilisateurs (collaboratif) et des articles (contenu).

