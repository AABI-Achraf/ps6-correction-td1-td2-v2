## 1. Introduction

Ce projet consiste à concevoir une base de données relationnelle destinée à optimiser la gestion d’un réseau ferroviaire national. L’objectif est de stocker et gérer les informations sur les gares, les lignes, les trains, les incidents, et d’offrir des outils analytiques permettant notamment de :
- Calculer les trajets minimisant les correspondances et les temps d’attente.
- Identifier les trains nécessitant une maintenance préventive.
- Analyser la saturation des gares durant les heures de pointe.
- Intégrer de nouvelles lignes en optimisant les correspondances.
- Proposer des trajets alternatifs en cas d’incident.
- Réaffecter des trains pour couvrir des pannes sur d’autres lignes.
- Évaluer l’impact des incidents sur la ponctualité globale.

Le projet a été implémenté sur MySQL et comporte des exemples de requêtes SQL avancées afin de démontrer la robustesse et l’évolutivité de la solution.

---

## 2. Analyse et Reformulation des Besoins

Suite aux négociations avec le client, les besoins ont été précisés comme suit :

### Services rendus par la base de données
- **Gestion des Gares**  
  Stocker les informations essentielles : nom, localisation (de type géographique), équipements, nombre de quais et capacité d’accueil.  
  *Justification :* Ces données permettent d’identifier la saturation et d’optimiser les correspondances.

- **Gestion des Lignes Ferroviaires**  
  Enregistrer le nom, le type de ligne (ex. TGV, TER), et définir l’ordre des gares desservies (via la table *Passage*).  
  *Justification :* L’ordre et les horaires de passage sont essentiels pour calculer les trajets et optimiser les temps d’attente.

- **Gestion des Trains**  
  Conserver le type, la capacité, la date de la dernière maintenance et le cumul des heures de trajet.  
  *Justification :* Ces informations permettent de déclencher des maintenances préventives.

- **Gestion des Incidents**  
  Journaliser les incidents (pannes, retards) avec date, durée, impact et l’association (éventuelle) à un train, une ligne ou une gare.  
  *Justification :* Le suivi des incidents est indispensable pour évaluer l’impact sur la ponctualité et réaffecter les ressources.

- **Optimisation et Analyse**  
  - Calculer les trajets minimisant les correspondances et le temps d’attente.
  - Proposer des alternatives en cas d’incident.
  - Optimiser l’intégration de nouvelles lignes en analysant les correspondances existantes.

### Exigences non fonctionnelles
- **Performance et réactivité** : Requêtes optimisées pour des recherches en temps réel.
- **Scalabilité** : Structure modulaire et normalisée (3NF/BCNF) pour permettre l’extension facile de la base.
- **Maintenabilité** : Documentation complète, utilisation de vues, triggers, fonctions et transactions pour faciliter la gestion et les mises à jour.

---

## 3. Schéma Relationnel et Justification de la Structure

### a. Structure du Schéma Relationnel

Les principales tables sont :

1. **Gare**  
   - `id_gare` INT AUTO_INCREMENT PRIMARY KEY  
   - `nom` VARCHAR(100) NOT NULL  
   - `localisation` POINT  
   - `équipements` TEXT  
   - `nb_quais` INT NOT NULL  
   - `capacité_max` INT NOT NULL  

2. **Ligne**  
   - `id_ligne` INT AUTO_INCREMENT PRIMARY KEY  
   - `nom_ligne` VARCHAR(100) NOT NULL  
   - `type_ligne` VARCHAR(50)

3. **Train**  
   - `id_train` INT AUTO_INCREMENT PRIMARY KEY  
   - `type_train` VARCHAR(50) NOT NULL  
   - `capacité_passagers` INT NOT NULL  
   - `date_maintenance` DATE  
   - `heures_cumulées` INT DEFAULT 0

4. **Incident**  
   - `id_incident` INT AUTO_INCREMENT PRIMARY KEY  
   - `type_incident` VARCHAR(50) NOT NULL  
   - `date_heure` TIMESTAMP NOT NULL  
   - `durée_retard` INT  
   - `impact` VARCHAR(100)  
   - `id_train` INT, `id_ligne` INT, `id_gare` INT  
   - **Contraintes** : Clés étrangères vers Train, Ligne et Gare (optionnelles).

5. **Trajet**  
   - `id_trajet` INT AUTO_INCREMENT PRIMARY KEY  
   - `id_ligne` INT NOT NULL  
   - `id_train` INT NOT NULL  
   - `horaire_départ` TIMESTAMP  
   - `horaire_arrivée` TIMESTAMP

6. **Segment**  
   - `id_segment` INT AUTO_INCREMENT PRIMARY KEY  
   - `id_gare_départ` INT NOT NULL  
   - `id_gare_arrivée` INT NOT NULL  
   - `temps_parcours` INT NOT NULL

7. **Passage**  
   - `id_passage` INT AUTO_INCREMENT PRIMARY KEY  
   - `id_ligne` INT NOT NULL  
   - `id_gare` INT NOT NULL  
   - `ordre_dans_ligne` INT NOT NULL  
   - `temps_attente_moyen` INT

### b. Justification des Choix de la Structure

- **Modularité et Clarté** :  
  Chaque entité (Gare, Ligne, Train, Incident) est stockée dans une table distincte. Les tables *Segment* et *Passage* permettent de dissocier les informations relatives aux itinéraires (temps de parcours et ordre des gares) des entités principales.
  
- **Normalisation** :  
  La structure respecte la 3NF et la BCNF pour éviter toute redondance et dépendance non souhaitée. Chaque attribut non clé dépend directement de la clé primaire de sa table, garantissant ainsi une cohérence des données.
  
- **Extensibilité** :  
  Le schéma est conçu pour évoluer, par exemple en ajoutant des tables complémentaires pour gérer les flux de passagers ou des données en temps réel.  
  *Exemple* : L’utilisation du type de données POINT pour la localisation ouvre la voie à des intégrations SIG.

- **Optimisation des Requêtes** :  
  La séparation des données permet de construire des requêtes efficaces (avec des jointures bien définies) pour répondre aux problématiques d’optimisation des trajets, de maintenance et d’analyse des incidents.

---

## 4. Implémentation en SQL (MySQL)

### a. Création des Tables

```sql
-- Table Gare
CREATE TABLE Gare (
    id_gare INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    localisation POINT,
    équipements TEXT,
    nb_quais INT NOT NULL,
    capacité_max INT NOT NULL
) ENGINE=InnoDB;

-- Table Ligne
CREATE TABLE Ligne (
    id_ligne INT AUTO_INCREMENT PRIMARY KEY,
    nom_ligne VARCHAR(100) NOT NULL,
    type_ligne VARCHAR(50)
) ENGINE=InnoDB;

-- Table Train
CREATE TABLE Train (
    id_train INT AUTO_INCREMENT PRIMARY KEY,
    type_train VARCHAR(50) NOT NULL,
    capacité_passagers INT NOT NULL,
    date_maintenance DATE,
    heures_cumulées INT DEFAULT 0
) ENGINE=InnoDB;

-- Table Incident
CREATE TABLE Incident (
    id_incident INT AUTO_INCREMENT PRIMARY KEY,
    type_incident VARCHAR(50) NOT NULL,
    date_heure TIMESTAMP NOT NULL,
    durée_retard INT,
    impact VARCHAR(100),
    id_train INT,
    id_ligne INT,
    id_gare INT,
    FOREIGN KEY (id_train) REFERENCES Train(id_train),
    FOREIGN KEY (id_ligne) REFERENCES Ligne(id_ligne),
    FOREIGN KEY (id_gare) REFERENCES Gare(id_gare)
) ENGINE=InnoDB;

-- Table Trajet
CREATE TABLE Trajet (
    id_trajet INT AUTO_INCREMENT PRIMARY KEY,
    id_ligne INT NOT NULL,
    id_train INT NOT NULL,
    horaire_départ TIMESTAMP,
    horaire_arrivée TIMESTAMP,
    FOREIGN KEY (id_ligne) REFERENCES Ligne(id_ligne),
    FOREIGN KEY (id_train) REFERENCES Train(id_train)
) ENGINE=InnoDB;

-- Table Segment
CREATE TABLE Segment (
    id_segment INT AUTO_INCREMENT PRIMARY KEY,
    id_gare_départ INT NOT NULL,
    id_gare_arrivée INT NOT NULL,
    temps_parcours INT NOT NULL,
    FOREIGN KEY (id_gare_départ) REFERENCES Gare(id_gare),
    FOREIGN KEY (id_gare_arrivée) REFERENCES Gare(id_gare)
) ENGINE=InnoDB;

-- Table Passage
CREATE TABLE Passage (
    id_passage INT AUTO_INCREMENT PRIMARY KEY,
    id_ligne INT NOT NULL,
    id_gare INT NOT NULL,
    ordre_dans_ligne INT NOT NULL,
    temps_attente_moyen INT,
    FOREIGN KEY (id_ligne) REFERENCES Ligne(id_ligne),
    FOREIGN KEY (id_gare) REFERENCES Gare(id_gare)
) ENGINE=InnoDB;
```

---

## 5. Stratégie de Test et Jeu de Données

### a. Stratégie de Test
La stratégie adoptée s’est déroulée en deux phases :
1. **Phase de Conception et Test Contrôlé**  
   - **Jeu de données réduit** : Création d’un jeu de données restreint (quelques gares, lignes, trains et incidents) afin de tester chaque requête de manière isolée et s’assurer de la cohérence des résultats.
   - **Objectif** : Valider la structure de la base, les contraintes d’intégrité et la logique des requêtes de base.
  
2. **Phase de Validation avec Données Complètes**  
   - **Jeu de données plus volumineux** : Intégration d’un ensemble de données représentatif des conditions réelles (exemple : 20 gares, 10 lignes, 15 trains, 20 incidents, et des enregistrements horaires simulant des pics de fréquentation).
   - **Objectif** : Tester les requêtes en conditions typiques et vérifier que les résultats restent cohérents même avec un volume important de données.
  
*Justification du choix* :  
Le jeu de données contrôlé permet d’identifier rapidement les erreurs de logique ou d’intégrité, tandis que le jeu complet garantit que la base peut gérer des scénarios réalistes (saturation, incidents multiples, correspondances complexes).

### b. Exemple de Jeu de Données (Extrait)
- **Gares** :  
  - *Paris-Nord* (capacité_max = 2000, 20 quais)  
  - *Lyon-Part-Dieu* (capacité_max = 1500, 15 quais)  
  - *Marseille-Saint-Charles*, etc.

- **Lignes** :  
  - *Ligne A* (TGV, reliant Paris à Marseille avec des arrêts intermédiaires)  
  - *Ligne B* (TER, reliant Lyon à d’autres villes régionales)

- **Trains** :  
  - *TGV001* (capacité_passagers = 500, heures_cumulées = 1200)  
  - *TER002* (capacité_passagers = 200, heures_cumulées = 800), etc.

- **Incidents** :  
  - Retard de 30 minutes sur TGV001, panne sur TER002, etc.

- **Passage et Segment** :  
  - Définition précise de l’ordre des arrêts et des temps de parcours entre gares pour valider les correspondances.

---

## 6. Exemples de Requêtes SQL


### 1. Trajets minimisant les correspondances  
**Auteur : Achraf AABI**  
```sql
SELECT p1.id_ligne,
       p1.ordre_dans_ligne AS ordre_depart,
       p2.ordre_dans_ligne AS ordre_arrivee,
       p1.temps_attente_moyen AS depart,
       p2.temps_attente_moyen AS arrivee
FROM Passage p1
JOIN Passage p2 ON p1.id_ligne = p2.id_ligne
WHERE p1.id_gare = [GareDépart]
  AND p2.id_gare = [GareArrivée]
  AND p1.ordre_dans_ligne < p2.ordre_dans_ligne
ORDER BY (p2.temps_attente_moyen - p1.temps_attente_moyen) ASC;
```
*Explication* :  
Cette requête extrait, pour une ligne donnée, les passages correspondant aux gares de départ et d’arrivée en s’assurant que l’ordre de passage respecte la progression du trajet. Le tri sur la différence des temps d’attente permet d’identifier le trajet avec le temps d’attente minimal.

---

### 2. Trains nécessitant une maintenance  
**Auteur : Aurélien GORJUX**  
```sql
SELECT t.id_train,
       t.heures_cumulées,
       COUNT(i.id_incident) AS nb_incidents
FROM Train t
LEFT JOIN Incident i ON t.id_train = i.id_train
GROUP BY t.id_train
HAVING t.heures_cumulées > 1000 OR COUNT(i.id_incident) >= 3;
```
*Explication* :  
Cette requête regroupe les trains par leur identifiant et compte le nombre d’incidents associés. Elle sélectionne ensuite ceux qui dépassent un seuil d’heures cumulées (ici 1000) ou qui ont subi au moins trois incidents, indiquant ainsi la nécessité d’une maintenance préventive.

---

### 3. Gares en saturation aux heures de pointe  
**Auteur : Achraf AABI**  
```sql
SELECT g.id_gare,
       g.nom,
       SUM(f.nb_passagers) AS total_passagers
FROM Gare g
JOIN Flux f ON g.id_gare = f.id_gare
WHERE HOUR(f.horaire_passage) BETWEEN 7 AND 9
GROUP BY g.id_gare, g.nom
HAVING total_passagers > g.capacité_max * 0.8;
```
*Explication* :  
En supposant l’existence d’une table *Flux* enregistrant le nombre de passagers par horaire, cette requête agrège les flux durant la période de pointe (7h-9h) et compare le total aux 80 % de la capacité maximale de chaque gare. Cela permet d’identifier les gares susceptibles d’être saturées.

---

### 4. Intégration d’une nouvelle ligne et optimisation des correspondances  
**Auteur : Aurélien GORJUX**  
```sql
SELECT l.id_ligne,
       COUNT(p.id_gare) AS nb_correspondances
FROM Passage p
JOIN Ligne l ON p.id_ligne = l.id_ligne
WHERE p.id_gare IN (SELECT id_gare FROM NouvelleLigneGares)
GROUP BY l.id_ligne
ORDER BY nb_correspondances DESC;
```
*Explication* :  
Cette requête permet d’identifier les lignes existantes qui offrent le plus d’opportunités de correspondance avec une nouvelle ligne (les gares communes étant définies dans la table *NouvelleLigneGares*). Le résultat aide à optimiser l’intégration de la nouvelle ligne dans le réseau.

---

### 5. Trajets alternatifs en cas d’incident  
**Auteur : Achraf AABI**  
```sql
SELECT DISTINCT p2.id_ligne,
       p2.temps_attente_moyen,
       p2.ordre_dans_ligne
FROM Passage p1
JOIN Passage p2 ON p1.id_gare = p2.id_gare
WHERE p1.id_gare = [GareProblème]
  AND p2.id_ligne <> [LigneIncidente];
```
*Explication* :  
Cette requête recherche, pour une gare affectée par un incident, toutes les lignes alternatives (excluant la ligne en panne) qui desservent cette gare. Cela permet de proposer des itinéraires de remplacement aux usagers.

---

### 6. Réaffectation des trains pour couvrir une panne sur une autre ligne  
**Auteur : Aurélien GORJUX**  
```sql
SELECT t.id_train,
       t.type_train
FROM Train t
WHERE t.id_train NOT IN (
      SELECT id_train
      FROM Incident
      WHERE type_incident = 'panne'
    )
  AND t.id_train IN (
      SELECT DISTINCT tra.id_train
      FROM Trajet tra
      WHERE tra.id_ligne = [LigneACouvrir]
    );
```
*Explication* :  
Cette requête identifie les trains disponibles (non concernés par des incidents de type « panne ») et déjà affectés à la ligne à couvrir. Ainsi, le système peut rapidement réaffecter les ressources disponibles pour compenser une panne sur une autre ligne.

---

### 7. Évaluation de l’impact des incidents sur la ponctualité  
**Auteur : Achraf AABI**  
```sql
SELECT i.type_incident,
       COUNT(*) AS nb_incidents,
       AVG(i.durée_retard) AS retard_moyen
FROM Incident i
GROUP BY i.type_incident
ORDER BY retard_moyen DESC;
```
*Explication* :  
En regroupant les incidents par type, cette requête calcule le nombre d’incidents et la moyenne des durées de retard. Le classement permet d’identifier les incidents les plus impactants sur la ponctualité du réseau.

---

## 7. Requêtes SQL Avancées Supplémentaires

Afin d’intégrer des notions avancées (vues, triggers, fonctions, CTE récursives, transactions), quatre requêtes supplémentaires ont été développées, deux par membre de l’équipe.


#### 8. Vue pour simplifier l’accès aux informations de correspondance  
```sql
CREATE VIEW Vue_Correspondances AS
SELECT l.nom_ligne, g.nom AS nom_gare, p.ordre_dans_ligne, p.temps_attente_moyen
FROM Passage p
JOIN Ligne l ON p.id_ligne = l.id_ligne
JOIN Gare g ON p.id_gare = g.id_gare;
```
*Explication* :  
Cette vue permet de regrouper et d’accéder facilement aux informations relatives aux correspondances entre gares et lignes. Elle simplifie la création de rapports et l’analyse des temps d’attente.  
**Auteur : Aurélien GORJUX**

#### 9. Trigger pour mettre à jour automatiquement les heures cumulées d’un train  
```sql
DELIMITER //
CREATE TRIGGER trg_update_heures_cumulées
AFTER INSERT ON Trajet
FOR EACH ROW
BEGIN
   DECLARE duree INT;
   SET duree = TIMESTAMPDIFF(MINUTE, NEW.horaire_départ, NEW.horaire_arrivée);
   UPDATE Train
   SET heures_cumulées = heures_cumulées + duree/60
   WHERE id_train = NEW.id_train;
END;
//
DELIMITER ;
```
*Explication* :  
Ce trigger se déclenche après l’insertion d’un trajet. Il calcule la durée (en minutes, convertie en heures) et met à jour le cumul des heures pour le train concerné.  
**Auteur : Aurélien GORJUX**

---

#### 10. Fonction stockée pour calculer le temps total de parcours entre deux gares via un CTE récursif  

```sql
WITH RECURSIVE Chemin (id, temps_total) AS (
  SELECT id_gare_départ AS id, temps_parcours
  FROM Segment
  WHERE id_gare_départ = [GareDépart]
  
  UNION ALL
  
  SELECT s.id_gare_arrivée, c.temps_total + s.temps_parcours
  FROM Segment s
  JOIN Chemin c ON s.id_gare_départ = c.id
)
SELECT MIN(temps_total) AS temps_minimum
FROM Chemin
WHERE id = [GareArrivée];
```
*Explication* :  
Cette requête utilise une CTE récursive pour calculers le temps total minimal entre la gare de départ et la gare d’arrivée.
**Auteur : Achraf AABI**

#### 11. Transaction combinant plusieurs opérations critiques  
```sql
START TRANSACTION;
  -- Mise à jour de la maintenance d'un train après un incident critique
  UPDATE Train SET date_maintenance = NOW()
  WHERE id_train = [TrainID] AND heures_cumulées > 1000;
  
  -- Insertion d'un nouvel incident de type 'maintenance'
  INSERT INTO Incident (type_incident, date_heure, durée_retard, impact, id_train)
  VALUES ('maintenance', NOW(), 0, 'Planifiée', [TrainID]);
  
COMMIT;
```
*Explication* :  
Cette transaction permet de regrouper deux opérations critiques : la mise à jour de la date de maintenance d’un train et l’insertion simultanée d’un incident de maintenance. L’utilisation d’une transaction garantit la cohérence des données en cas d’erreur.  
**Auteur : Achraf AABI**

---

## 8. Découpage et Répartition du Travail en Équipe

### Organisation de l’équipe
- **Membres** :  
  - Achraf AABI  
  - Aurélien GORJUX

### Répartition des Tâches
- **Achraf AABI**  
  - Conception du modèle relationnel
  - Analyse des besoins et modélisation des entités et rédaction de la documentation technique.  
  - Réalisation des requêtes concernant l’optimisation des trajets et l’analyse des incidents (requêtes 1, 3, 5, 7).  
  - Développement des requêtes avancées utilisant la récursivité et les transactions (requêtes 10 et 11).

- **Aurélien GORJUX**
  - Conception du modèle relationnel
  - Analyse des besoins et modélisation des entités, et réalisation de la vidéo
  - Développement des requêtes pour la maintenance et la réaffectation des trains (requêtes 2, 4, 6).  
  - Mise en place des vues et triggers pour améliorer l’automatisation et le contrôle (requêtes 8 et 9).

### Difficultés Rencontrées et Retours d’Expérience
- **Difficultés techniques** :  
  - Synchronisation et validation de la cohérence des mises à jour en transaction.

- **Points Positifs** :  
  - Une bonne répartition des tâches permettant de couvrir à la fois la modélisation, l’implémentation SQL et l’intégration de notions avancées.
  - Une communication efficace entre les membres de l’équipe, favorisant le partage de bonnes pratiques.

- **Axes d’Amélioration**  
  Si le projet était à refaire, nous envisagerions :  
  - Une planification encore plus détaillée dès le départ pour éviter certaines itérations sur les requêtes complexes.
  - L’utilisation d’outils collaboratifs plus avancés pour la modélisation (ex. PowerDesigner) et l’intégration continue des tests.

---

## 9. Conclusion Générale et Documentation

Le projet de gestion des infrastructures ferroviaires implémenté sur MySQL répond de manière exhaustive aux besoins exprimés par le client. Grâce à une modélisation précise, une normalisation rigoureuse et l’intégration de requêtes avancées (vues, triggers, fonctions, CTE récursifs, transactions), la base de données offre à la fois performance, évolutivité et facilité de maintenance.

### Documentation et Apprentissage
- **Outils utilisés** :  
  - MySQL pour l’implémentation de la base de données.  
  - Environnements de test SQL (MySQL Workbench).  
  - Outils de modélisation (dessins UML réalisés avec PlantUML).

- **Points Clés Appris** :  
  - L’importance de la normalisation et de la structuration des données pour garantir la cohérence et la performance.
  - L’utilisation des notions avancées SQL (vues, triggers, fonctions stockées, CTE récursifs et transactions) pour automatiser et sécuriser les opérations critiques.
  - La nécessité d’une stratégie de test progressive, passant d’un jeu de données contrôlé à des volumes plus réalistes afin d’assurer la robustesse du système.
  - L’efficacité d’un travail en équipe bien organisé et réparti, avec une communication constante pour surmonter les défis techniques.

En conclusion, ce projet représente une base solide et évolutive pour la gestion d’un réseau ferroviaire, et l’expérience acquise permettra d’optimiser davantage tant la conception technique que l’organisation en équipe pour de futurs développements.

---

Ce document constitue la synthèse complète de notre projet, intégrant à la fois la conception technique, la stratégie de tests, l’organisation de l’équipe et l’utilisation de concepts SQL avancés pour répondre aux exigences fonctionnelles et techniques du client.
