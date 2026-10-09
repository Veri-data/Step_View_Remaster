# BNI · Pilotage des projets — modèle de données et mesures DAX

Ce guide accompagne la maquette (canevas Design) et le thème `BNI_theme.json`.
Il décrit comment brancher les 3 pages sur **Planner Premium** (stockage Dataverse) et donne toutes les mesures utilisées par les visuels.

> Les noms logiques Dataverse ci-dessous sont ceux des tables de Project for the web / Planner Premium. Vérifiez-les dans votre environnement (Power Apps › Tables) avant de lancer la connexion : certains libellés peuvent différer selon la version.

---

## 1. Connexion

1. Power BI Desktop › **Obtenir les données › Dataverse**.
2. Choisir l'environnement qui héberge les plans Premium (souvent l'environnement par défaut du tenant).
3. Cocher les tables ci-dessous, puis **Transformer les données** pour renommer les colonnes.

| Table Power BI | Table Dataverse | Colonnes à garder → nom dans le modèle |
|---|---|---|
| **Projets** | `msdyn_project` | `msdyn_projectid` → Projet ID · `msdyn_subject` → Projet · chef de projet (lookup `msdyn_projectmanager`, développer le nom) → Chef de projet · `msdyn_scheduledstart` → Début · `msdyn_finish` → Fin |
| **Tâches** | `msdyn_projecttask` | `msdyn_projecttaskid` → Tâche ID · `_msdyn_project_value` → Projet ID · `msdyn_subject` → Tâche · `msdyn_scheduledstart` → Début · `msdyn_scheduledend` → Fin · `msdyn_progress` → % Avancement (0 à 1) · `msdyn_effort` → Charge (h) · `msdyn_effortcompleted` → Réalisé (h) · `msdyn_summary` → Récapitulative · `msdyn_duration` → Durée (j) |
| **Affectations** | `msdyn_resourceassignment` | `msdyn_resourceassignmentid` → Affectation ID · `msdyn_taskid` → Tâche ID · `msdyn_bookableresourceid` → Ressource ID · `msdyn_plannedwork` (JSON du travail planifié) |
| **Ressources** | `bookableresource` | `bookableresourceid` → Ressource ID · `name` → Ressource |

Étapes Power Query indispensables :

- **Tâches** : filtrer `Récapitulative = false` pour ne pas compter deux fois les tâches parentes.
- **Projets** : ajouter une colonne **Date d'achèvement** si votre environnement expose une date de fin réelle ; sinon, utiliser la date de modification de la dernière tâche passée à 100 % (approximation à valider avec le PMO).
- **Actualisation** : nouvelle requête vide `= #table({"Date"}, {{DateTime.LocalNow()}})`. Côte d'Ivoire = UTC+0, l'heure affichée dans le service Power BI est donc correcte.

### Table « Charge planifiée » (heatmap de la page 2)

`msdyn_plannedwork` contient le travail planifié jour par jour au format JSON. Dupliquer la requête Affectations puis :

```powerquery
let
    Source   = Affectations,
    Filtre   = Table.SelectRows(Source, each [msdyn_plannedwork] <> null),
    Json     = Table.AddColumn(Filtre, "Travail", each Json.Document([msdyn_plannedwork])),
    Lignes   = Table.ExpandListColumn(Json, "Travail"),
    Champs   = Table.ExpandRecordColumn(Lignes, "Travail", {"Start", "Hours"}),
    Date     = Table.AddColumn(Champs, "Date", each Date.From(
                   #datetime(1970, 1, 1, 0, 0, 0)
                   + #duration(0, 0, 0, Number.From(Text.BetweenDelimiters([Start], "(", ")")) / 1000)), type date),
    Heures   = Table.TransformColumnTypes(Date, {{"Hours", type number}}),
    Resultat = Table.RenameColumns(
                   Table.SelectColumns(Heures, {"Ressource ID", "Tâche ID", "Date", "Hours"}),
                   {{"Hours", "Heures"}})
in
    Resultat
```

Si le texte de `Start` contient un fuseau (`/Date(1760000000000+0000)/`), remplacer le délimiteur de fin `")"` par `"+"`.

---

## 2. Table calendrier

```dax
Calendrier =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2025, 1, 1 ), DATE ( 2028, 12, 31 ) ),
    "Année", YEAR ( [Date] ),
    "Mois", DATE ( YEAR ( [Date] ), MONTH ( [Date] ), 1 ),
    "Libellé mois", FORMAT ( [Date], "mmm yyyy" ),
    "Trimestre", "T" & QUARTER ( [Date] ) & " " & YEAR ( [Date] )
)
```

Trier `Libellé mois` par `Mois` (Outils de colonne › Trier par colonne).

## 3. Relations (toutes en filtrage unidirectionnel, 1 → plusieurs)

| De (1) | Vers (plusieurs) |
|---|---|
| Projets[Projet ID] | Tâches[Projet ID] |
| Tâches[Tâche ID] | Affectations[Tâche ID] |
| Ressources[Ressource ID] | Affectations[Ressource ID] |
| Tâches[Tâche ID] | Charge planifiée[Tâche ID] |
| Ressources[Ressource ID] | Charge planifiée[Ressource ID] |
| Calendrier[Date] | Charge planifiée[Date] |
| Projets[Projet ID] | Événements[Projet ID] |

Les mesures de la matrice « Projet › Tâche › Ressource » activent ponctuellement le filtrage bidirectionnel avec `CROSSFILTER`, ce qui évite de le laisser actif dans tout le modèle.

---

## 4. Colonnes calculées

### Tâches

```dax
Statut tâche =
VAR _aujourdhui = TODAY ()
RETURN
    SWITCH (
        TRUE (),
        Tâches[% Avancement] >= 1, "Terminé",
        Tâches[Fin] < _aujourdhui, "En retard",
        Tâches[% Avancement] > 0, "En cours",
        "Pas commencé"
    )

Jalon = Tâches[Durée (j)] = 0
```

### Projets

```dax
% Avancement =
DIVIDE (
    SUMX ( RELATEDTABLE ( Tâches ), Tâches[Réalisé (h)] ),
    SUMX ( RELATEDTABLE ( Tâches ), Tâches[Charge (h)] )
)

Statut =
VAR _aujourdhui = TODAY ()
RETURN
    SWITCH (
        TRUE (),
        Projets[% Avancement] >= 0.999, "Terminé",
        Projets[Fin] < _aujourdhui, "En retard",
        Projets[% Avancement] > 0 || Projets[Début] <= _aujourdhui, "En cours",
        "Pas commencé"
    )

Ordre statut = SWITCH ( Projets[Statut], "En retard", 1, "En cours", 2, "Pas commencé", 3, "Terminé", 4 )

Clé de tri = Projets[Ordre statut] * 100000 + INT ( Projets[Fin] )

Durée (j ouvrés) = NETWORKDAYS ( Projets[Début], Projets[Fin] )

Année = YEAR ( Projets[Début] )

Fin réelle = IF ( Projets[Statut] = "Terminé", Projets[Date d'achèvement] )
```

Trier `Projets[Projet]` par `Projets[Clé de tri]` : la matrice affiche alors les projets en retard en premier, puis par date de fin.

**URL du projet.** Ouvrez un plan Premium dans Planner, copiez l'adresse affichée par le navigateur et repérez l'identifiant du projet (un GUID). Construisez la colonne en remplaçant ce GUID par `Projets[Projet ID]` :

```dax
URL projet = "<début de l'adresse copiée>" & Projets[Projet ID] & "<fin de l'adresse copiée>"
```

Catégorie de données : **URL web**.

### Affectations

```dax
Nb affectés = CALCULATE ( COUNTROWS ( Affectations ), ALLEXCEPT ( Affectations, Affectations[Tâche ID] ) )
Charge (h)  = DIVIDE ( RELATED ( Tâches[Charge (h)] ), Affectations[Nb affectés] )
% tâche     = RELATED ( Tâches[% Avancement] )
Réalisé (h) = Affectations[Charge (h)] * Affectations[% tâche]
Statut tâche = RELATED ( Tâches[Statut tâche] )
```

Si `msdyn_plannedwork` est renseigné, vous pouvez remplacer `Charge (h)` par la somme des heures de la table Charge planifiée pour l'affectation.

### Événements (page 3)

```dax
Événements =
UNION (
    SELECTCOLUMNS ( Projets, "Projet ID", Projets[Projet ID], "Date", Projets[Début], "Événement", "Démarrage" ),
    SELECTCOLUMNS ( Projets, "Projet ID", Projets[Projet ID], "Date", Projets[Fin], "Événement", "Fin du projet" ),
    SELECTCOLUMNS (
        FILTER ( Tâches, Tâches[Jalon] ),
        "Projet ID", Tâches[Projet ID], "Date", Tâches[Fin], "Événement", "Jalon : " & Tâches[Tâche]
    )
)
```

Colonnes à ajouter sur Événements :

```dax
Jours restants = DATEDIFF ( TODAY (), Événements[Date], DAY )
Libellé J = "J-" & Événements[Jours restants]
```

---

## 5. Mesures

Créer une table vide « Mesures » (Accueil › Entrer des données) pour les ranger.

### Page 1 · Nos projets

```dax
Nb projets = DISTINCTCOUNT ( Projets[Projet ID] )
Nb chefs de projet = DISTINCTCOUNT ( Projets[Chef de projet] )

Projets en cours = CALCULATE ( [Nb projets], Projets[Statut] = "En cours" )
Projets terminés = CALCULATE ( [Nb projets], Projets[Statut] = "Terminé" )
Projets en retard = CALCULATE ( [Nb projets], Projets[Statut] = "En retard" )
Projets pas commencés = CALCULATE ( [Nb projets], Projets[Statut] = "Pas commencé" )

% En cours = DIVIDE ( [Projets en cours], [Nb projets] )
% Terminés = DIVIDE ( [Projets terminés], [Nb projets] )
% En retard = DIVIDE ( [Projets en retard], [Nb projets] )

Terminés dans les délais =
CALCULATE (
    [Nb projets],
    FILTER ( Projets, Projets[Statut] = "Terminé" && Projets[Fin réelle] <= Projets[Fin] )
)
% Dans les délais = DIVIDE ( [Terminés dans les délais], [Projets terminés] )

Durée totale (j) = SUM ( Projets[Durée (j ouvrés)] )
Durée moyenne (j) = AVERAGE ( Projets[Durée (j ouvrés)] )

Charge planifiée projets (h) = SUM ( Tâches[Charge (h)] )
Charge réalisée projets (h) = SUM ( Tâches[Réalisé (h)] )
% Avancement global = DIVIDE ( [Charge réalisée projets (h)], [Charge planifiée projets (h)] )
```

La carte « Tâches en retard » utilise la mesure `[Tâches en retard]` définie plus bas avec la matrice.

Sous-libellés des cartes (champ **Libellé de référence** de la Carte (nouveau)) :

```dax
Sous-titre projets = [Nb chefs de projet] & " chefs de projet"
Sous-titre durée = "moy. " & FORMAT ( [Durée moyenne (j)], "0" ) & " j ouvrés / projet"
Sous-titre en cours = [Projets en cours] & " projets sur " & [Nb projets]
Sous-titre terminés = [Projets terminés] & " projets sur " & [Nb projets]
Sous-titre retard = [Projets en retard] & " projets sur " & [Nb projets]
Sous-titre délais = [Terminés dans les délais] & " sur " & [Projets terminés] & " livrés à temps"
Sous-titre avancement =
    FORMAT ( [Charge réalisée projets (h)], "#,##0" ) & " h / " & FORMAT ( [Charge planifiée projets (h)], "#,##0" ) & " h"
Sous-titre tâches retard = "sur " & COUNTROWS ( Tâches ) & " tâches au total"

Libellé actualisation =
VAR _d = MAX ( Actualisation[Date] )
RETURN "Actualisé le " & FORMAT ( _d, "dd/mm/yyyy" ) & " à " & FORMAT ( _d, "hh:nn" )
```

Graphique « Avancement par projet » (barres groupées) :

```dax
% Délai consommé =
VAR _debut = MIN ( Projets[Début] )
VAR _fin = MAX ( Projets[Fin] )
VAR _total = DATEDIFF ( _debut, _fin, DAY )
VAR _ecoule = DATEDIFF ( _debut, MIN ( TODAY (), _fin ), DAY )
RETURN IF ( _total > 0, DIVIDE ( MAX ( 0, _ecoule ), _total ) )

Couleur statut =
SWITCH (
    SELECTEDVALUE ( Projets[Statut] ),
    "En cours", "#3F86C8",
    "En retard", "#EF5B3A",
    "Pas commencé", "#B4BFB9",
    "Terminé", "#00734E"
)

Libellé heures =
FORMAT ( [Charge réalisée projets (h)], "#,##0" ) & " / " & FORMAT ( [Charge planifiée projets (h)], "#,##0" ) & " h"
```

Réglages : Valeurs = `[% Avancement global]` · Couleur des barres › fx › Valeur du champ = `[Couleur statut]` · Analytique › Barres d'erreur : limite supérieure et inférieure = `[% Délai consommé]`, style marqueur · Étiquettes de données › Détail = `[Libellé heures]`.

### Matrice « Récapitulatif des projets »

Lignes : `Projets[Projet]` › `Tâches[Tâche]` › `Ressources[Ressource]`. Valeurs dans cet ordre :

```dax
Icône statut =
VAR _statut =
    IF (
        ISINSCOPE ( Tâches[Tâche] ),
        CALCULATE (
            SELECTEDVALUE ( Tâches[Statut tâche] ),
            CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH )
        ),
        SELECTEDVALUE ( Projets[Statut] )
    )
VAR _fond = SWITCH ( _statut, "En cours", "%23E6F0FA", "En retard", "%23FDEAE5", "Pas commencé", "%23EEF1EF", "Terminé", "%23E1F2E8" )
VAR _encre = SWITCH ( _statut, "En cours", "%231F5F99", "En retard", "%23A93218", "Pas commencé", "%2356635C", "Terminé", "%2300673F" )
RETURN
    IF (
        NOT ISBLANK ( _statut ),
        "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='116' height='24' viewBox='0 0 116 24'>"
            & "<rect width='116' height='24' rx='12' fill='" & _fond & "'/>"
            & "<circle cx='14' cy='12' r='4' fill='" & _encre & "'/>"
            & "<text x='24' y='16' font-family='Segoe UI' font-size='12' font-weight='600' fill='" & _encre & "'>"
            & _statut & "</text></svg>"
    )

URL projet (matrice) =
IF ( ISINSCOPE ( Projets[Projet] ) && NOT ISINSCOPE ( Tâches[Tâche] ), SELECTEDVALUE ( Projets[URL projet] ) )

Début (matrice) =
CALCULATE ( MIN ( Tâches[Début] ), CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH ) )

Fin (matrice) =
CALCULATE ( MAX ( Tâches[Fin] ), CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH ) )

% Achèvement =
CALCULATE (
    DIVIDE ( SUM ( Tâches[Réalisé (h)] ), SUM ( Tâches[Charge (h)] ) ),
    CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH )
)

Tâches total =
CALCULATE ( COUNTROWS ( Tâches ), CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH ) )

Tâches en cours =
VAR _n = CALCULATE ( [Tâches total], Tâches[Statut tâche] = "En cours" )
RETURN IF ( NOT ISBLANK ( [Tâches total] ), _n + 0 )

Tâches en retard =
VAR _n = CALCULATE ( [Tâches total], Tâches[Statut tâche] = "En retard" )
RETURN IF ( NOT ISBLANK ( [Tâches total] ), _n + 0 )

Tâches pas commencées =
VAR _n = CALCULATE ( [Tâches total], Tâches[Statut tâche] = "Pas commencé" )
RETURN IF ( NOT ISBLANK ( [Tâches total] ), _n + 0 )

Tâches terminées =
VAR _n = CALCULATE ( [Tâches total], Tâches[Statut tâche] = "Terminé" )
RETURN IF ( NOT ISBLANK ( [Tâches total] ), _n + 0 )
```

Le `+ 0` sous condition affiche « 0 » sur les lignes existantes sans faire apparaître toutes les combinaisons projet × ressource.

Réglages de la matrice :

- `Icône statut` : catégorie de données **URL d'image** ; Format › Grille › taille d'image 24 × 116.
- `URL projet (matrice)` : catégorie **URL web** ; Format › Valeurs › Icône d'URL = activé.
- `% Achèvement` : format pourcentage sans décimale ; Mise en forme conditionnelle › **Barres de données** (min 0, max 1, couleur #009043).
- `Tâches en retard` : couleur de police par règles (si valeur > 0 alors #C63E22) ; zéros en #8A968F.
- Disposition **hiérarchique**, icônes +/- activées (chevrons), total de lignes affiché.

### Page 2 · Nos ressources

```dax
Nb ressources = DISTINCTCOUNT ( Affectations[Ressource ID] )
Tâches assignées = COUNTROWS ( Affectations )
Tâches par ressource = DIVIDE ( [Tâches assignées], [Nb ressources] )

Affectations en retard = CALCULATE ( COUNTROWS ( Affectations ), Affectations[Statut tâche] = "En retard" )
Ressources en retard = CALCULATE ( [Nb ressources], Affectations[Statut tâche] = "En retard" )

Volume total (h) = SUM ( Affectations[Charge (h)] )

Volume réalisé (h) =
IF ( NOT ISBLANK ( [Volume total (h)] ), SUM ( Affectations[Réalisé (h)] ) + 0 )

Volume en cours (h) =
VAR _v =
    CALCULATE (
        SUM ( Affectations[Charge (h)] ) - SUM ( Affectations[Réalisé (h)] ),
        Affectations[% tâche] > 0,
        Affectations[% tâche] < 1
    )
RETURN IF ( NOT ISBLANK ( [Volume total (h)] ), _v + 0 )

Volume restant (h) =
VAR _v = CALCULATE ( SUM ( Affectations[Charge (h)] ), Affectations[% tâche] = 0 )
RETURN IF ( NOT ISBLANK ( [Volume total (h)] ), _v + 0 )

% Achèvement ressource = DIVIDE ( [Volume réalisé (h)], [Volume total (h)] )

Début (ressource) =
CALCULATE ( MIN ( Tâches[Début] ), CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH ) )

Fin (ressource) =
CALCULATE ( MAX ( Tâches[Fin] ), CROSSFILTER ( Affectations[Tâche ID], Tâches[Tâche ID], BOTH ) )
```

Contrôle : `Volume total = Volume réalisé + Volume en cours + Volume restant` sur chaque ligne.

- **En cours** : heures restantes sur les tâches démarrées.
- **Restant** : heures des tâches pas encore démarrées.

Charge mensuelle et surcharge :

```dax
Charge planifiée (h) = SUM ( 'Charge planifiée'[Heures] )

Seuil surcharge (h) = 160

Ressources en surcharge =
VAR _debut = EOMONTH ( TODAY (), -1 ) + 1
VAR _fin = EOMONTH ( TODAY (), 5 )
RETURN
    CALCULATE (
        COUNTROWS (
            FILTER (
                VALUES ( Ressources[Ressource ID] ),
                MAXX ( VALUES ( Calendrier[Mois] ), [Charge planifiée (h)] ) > [Seuil surcharge (h)]
            )
        ),
        DATESBETWEEN ( Calendrier[Date], _debut, _fin )
    )
```

Heatmap (matrice Ressource × `Calendrier[Libellé mois]`, valeur `[Charge planifiée (h)]`) : Mise en forme conditionnelle › Couleur d'arrière-plan › **Règles** :

| Si la valeur est | Couleur |
|---|---|
| = 0 | #F4F7F5 |
| > 0 et ≤ 40 | #E3F2E9 |
| > 40 et ≤ 80 | #C4E6D1 |
| > 80 et ≤ 120 | #97D2B0 |
| > 120 et ≤ 160 | #5DB88A |
| > 160 | #FF9F2E |

Filtre du visuel : `Calendrier[Date]` entre le 1er du mois courant et +6 mois (filtre de date relative).

### Page 3 · Timeline projets

```dax
Projets actifs = [Projets en cours] + [Projets en retard]

Retard (j) =
IF (
    SELECTEDVALUE ( Projets[Statut] ) = "En retard",
    DATEDIFF ( SELECTEDVALUE ( Projets[Fin] ), TODAY (), DAY )
)

Retard cumulé (j) =
SUMX ( FILTER ( Projets, Projets[Statut] = "En retard" ), DATEDIFF ( Projets[Fin], TODAY (), DAY ) )

Fins prévues au trimestre =
VAR _debut = DATE ( YEAR ( TODAY () ), ( QUARTER ( TODAY () ) - 1 ) * 3 + 1, 1 )
VAR _fin = EOMONTH ( _debut, 2 )
RETURN CALCULATE ( [Nb projets], FILTER ( Projets, Projets[Fin] >= _debut && Projets[Fin] <= _fin ) )

Échéances 60 j =
CALCULATE (
    COUNTROWS ( Événements ),
    Événements[Jours restants] >= 0,
    Événements[Jours restants] <= 60
)
```

**Visuel Gantt** (AppSource › « Gantt » publié par Microsoft) :

| Zone du visuel | Champ |
|---|---|
| Parent | Projets[Chef de projet] |
| Tâche | Projets[Projet] |
| Date de début | Projets[Début] |
| Date de fin | Projets[Fin] |
| % d'achèvement | Projets[% Avancement] |
| Légende | Projets[Statut] |

Format : type de date « Mois », ligne Aujourd'hui en #FF8D00, couleurs de légende = couleurs des statuts, largeur des libellés 300 px. Pour reproduire exactement la maquette (dépassement hachuré, jalons en losange, regroupement par chef), utiliser le visuel **Deneb** (Vega-Lite).

---

## 6. Boutons et signets

- **Navigation** : Insérer › Boutons › Navigateur › Navigateur de pages. Le thème applique les états Par défaut / Survol / Sélectionné.
- **Nouveau projet** : Insérer › Boutons › Vide. Texte « + Nouveau projet », remplissage #FF8D00, rayon 22. Action › Type « URL web » › `https://planner.cloud.microsoft/` (ou l'adresse de création de plan de votre tenant).
- **Réinitialiser** : vider tous les segments, Affichage › Signets › Ajouter « Réinitialiser » (cocher Données, décocher Page actuelle). Bouton › Action › Signet « Réinitialiser ». Synchroniser les segments entre les 3 pages (Affichage › Synchroniser les segments).

## 7. Couleurs des statuts (à reprendre dans chaque visuel)

| Statut | Couleur | Fond de puce | Texte de puce |
|---|---|---|---|
| En cours | #3F86C8 | #E6F0FA | #1F5F99 |
| En retard | #EF5B3A | #FDEAE5 | #A93218 |
| Pas commencé | #B4BFB9 | #EEF1EF | #56635C |
| Terminé | #00734E | #E1F2E8 | #00673F |
