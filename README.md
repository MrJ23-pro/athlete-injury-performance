# Test technique : risque de blessure et capacité à performer

Données de 3 sportifs (p01, p03, p05) entre novembre 2019 et mars 2020 : montre Fitbit, questionnaires, séances d'entraînement et photos de repas. L'objectif est de regrouper ces données par jour, de les nettoyer, puis d'entraîner deux modèles, l'un pour le risque de blessure, l'autre pour un indice de performance entre 0 et 100.

Les données ne sont pas incluses dans le dépôt.

## Installation

Projet développé avec Python 3.13.

    python3 -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt

Sous Windows, l'environnement s'active avec `.venv\Scripts\activate`.

## Données

Les données du test vont dans `data/Test_technique/` et la table Ciqual dans `data/ciqual_table/` :

    data/
      Test_technique/
        p01/  (fitbit, googledocs, pmsys, food-images)
        p03/
        p05/
        participant-overview.xlsx
      ciqual_table/
        alim_2025_11_03.xml
        alim_grp_2025_11_03.xml
        compo_2025_11_03.xml
        const_2025_11_03.xml

La table Ciqual 2025 de l'Anses est téléchargeable gratuitement sur le site de Ciqual, au format XML. Le dossier `data/processed/` est créé automatiquement.

## Notebooks

Les notebooks se trouvent dans `notebooks/` et se lancent dans l'ordre :

- `01_explo` et `02_agg` : premiers essais sur p01 pour comprendre les fichiers, pas nécessaires pour la suite
- `03_dataset` : tableau avec une ligne par participant et par jour, exporté dans `daily_raw.csv`
- `04_food_images` : lecture des dates dans les métadonnées des photos et estimation des calories consommées, exportées dans `photos_meta.csv`, `repas_jour.csv` et `nutri_photos.csv`
- `05_aed` : analyse exploratoire
- `06_cleaning` : nettoyage et export de `test_dataset.csv`
- `07_modeling` : indice de performance et modèles

Pour tout relancer, utiliser Kernel > Restart Kernel and Run All Cells sur chaque notebook. Le notebook 04 télécharge deux modèles depuis Hugging Face (environ 2 Go au total) et prend quelques minutes sur les 643 photos.

## Démarche

Les fichiers n'ont pas tous la même fréquence (toutes les 5 secondes, chaque minute, par séance, par jour ou par semaine), toutes les données ont donc été ramenées au jour.

Au nettoyage, les jours où la montre a été portée moins de 10 h sont écartés, ainsi que les calories estimées par Fitbit les jours sans montre, deux questionnaires envoyés vides, la readiness de p03 et p05 (valeurs par défaut de l'application) et les FC max supérieures à la FC max mesurée de chaque participant. Les valeurs manquantes restent en NaN dans `test_dataset.csv`. Elles sont remplacées par la médiane dans les pipelines des modèles, calculée uniquement sur les données d'entraînement.

Pour les photos de repas, la date de prise de vue est lue dans les métadonnées EXIF de chaque image (champ DateTimeOriginal, ou DateTime à défaut). Les 643 photos ont une date, toutes entre le 1er février et le 31 mars 2020, en heure locale (UTC+1). Chaque photo est rattachée à un participant et à un jour, ce qui donne par jour le nombre de photos et l'heure du premier et du dernier repas. Les métadonnées contiennent aussi des coordonnées GPS, qui ne sont pas utilisées. p01 a photographié ses repas tous les jours (environ 5 photos par jour), p03 et p05 beaucoup moins régulièrement.

Pour les calories consommées, un premier modèle entraîné sur Food-101 se trompait sur 4 photos sur 5 (aucune boisson, peu de plats du quotidien). J'ai donc utilisé SigLIP 2 en lui donnant comme liste d'étiquettes tous les aliments de la table Ciqual. Pour chaque photo, je garde les 5 aliments les plus proches et la moyenne de leurs valeurs pour 100 g, puis j'applique une portion fixe : 250 ml pour une boisson, 40 g pour un aliment très calorique (plus de 350 kcal pour 100 g) et 150 g dans les autres cas.

L'indice de performance combine le bien-être déclaré, le score de sommeil et la fréquence cardiaque de repos. Chaque valeur est comparée à l'habitude du participant, puis ramenée sur une échelle de 0 à 100 où 50 correspond à un jour normal.

## Résultats

Performance, avec prédiction de l'indice du lendemain (entraînement jusqu'au 15/02, test après) :

- prédire toujours 50 : 7.6 points d'erreur moyenne
- prédire la même valeur que la veille : 8.0
- Ridge : 7.2
- Random Forest : 7.7

Blessure, avec prédiction d'un début de blessure dans les 7 jours : seulement 2 épisodes sont utilisables. Une régression logistique, une version avec les variables comparées à l'habitude du participant et un score basé sur des règles ont été testés, sans faire mieux que le hasard. Les deux blessures n'ont pas de profil commun dans les données, et celle de p01 (à la main) ressemble davantage à un accident qu'à une blessure de surcharge.

## Limites

- seulement 3 participants et 2 blessures utilisables
- aucune donnée de fréquence cardiaque pour p03, qui remplit aussi peu ses questionnaires
- changement brutal des réponses de p05 au questionnaire début janvier, probablement lié à l'application
- calories consommées sous-estimées (environ 1 350 kcal par jour pour p01, qui en dépense environ 3 600) à cause des portions fixes, des photos avec plusieurs aliments et des repas non photographiés par p03 et p05. Cette variable est surtout exploitable en relatif.