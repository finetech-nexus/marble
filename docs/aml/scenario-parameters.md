# Paramètres des scénarios

Les valeurs sont des exemples. Elles ne deviennent pas des seuils réglementaires parce qu’elles figurent dans le dépôt.

## Scénario Marble « AML transaction check »

Déclencheur : objet `transactions`. Seuils de score de l’itération : revue 30, blocage et revue 80, refus 100.

| Règle | Formule déployée | Score |
|---|---|---|
| Large amount | `amount >= 10000` | 40 |
| High-risk country | `country = KP` ou `country = IR` | 100 |

Ces nombres sont écrits dans `init-config.yaml`. Ils ne sont pas des paramètres éditables du module `bankaml`. Les modifier demande une nouvelle itération Marble.

## Scénario Marble « AML screening »

Déclencheur : objet `clients`. Seuil de similarité de la configuration de screening : 70. Issue forcée : `review`. Type d’entité : `Person`. Requête : concaténation de `full_name`. Une règle « Customer name is present » ajoute 1 point si le nom n’est pas vide. La publication a été refusée par le contrôle de licence sanctions. Le scénario peut être enregistré sans être live.

## UC du module

Source : `DefaultScenarios` dans `api/bankaml/scenarios.go`. Tous sont `enabled: false`, `status: draft`, version 1.

| Code | Paramètres d’exemple |
|---|---|
| UC01 | `individual_min` 1000, `individual_max` 9000, `cumulative` 10000, `min_ops` 3, `window_hours` 24, `tolerance` 0,15 |
| UC02 | `baseline_days` 30, `observation_days` 7, `multiplier` 3, `min_volume` 5000, `min_ops` 5 |
| UC03 | `max_delay_hours` 48, `proportion` 0,8, `min_amount` 1000, `min_counterparties` 1 |
| UC04 | `depth` 4, `window_hours` 72, `tolerance` 0,1 |
| UC05 | `min_amount` 2000, `min_ops` 3, `window_days` 7, `history_multiplier` 3 |
| UC06 | `max_counterparties` 10, `concentration` 0,8, `window_days` 30 |
| UC07 | `countries` : `KP`, `IR` |
| UC08 | `inactive_days` 90, `min_amount` 1000 |
| UC09 | `volume_multiplier` 2, `count_multiplier` 2 |
| UC10 | `min_clients` 3, `window_days` 30 |
| UC11 | aucun paramètre |
| UC12 | aucun paramètre ; le score a sa propre formule |

Un nouveau brouillon de scénario passe par l’API du module. Il reste inactif tant qu’un autre utilisateur ne l’approuve pas. Le schéma cible (`minimum`, `maximum`, `currency`, `required`, `description`) n’est pas stocké. `params` est un objet JSON libre validé par le code de l’UC, pas par un JSON Schema.

## Formule de score

Poids initiaux : KYC 0,35, transactions 0,40, alertes 0,25. La somme doit valoir 1. Classes : `low` 30, `moderate` 60, `high` 85, strictement croissantes et au plus 100.

Facteurs initiaux : type de client, pays du profil, ancienneté KYC, volume, fréquence, alertes ouvertes. La liste de pays à risque du score est le paramètre de banque `high_risk_countries`, pas la liste codée de UC07.

## Déduplication

Le modèle cible demande une clé et un délai. Le module déduplique les dossiers par `(org_id, alert_key)`. Les recherches OpenSanctions ont un délai d’une minute, sauf `force`. Les scénarios UC n’ont pas de cooldown de 1440 minutes.

## Devises, dates, annulations

Les sommes UC n’utilisent pas `amount_base_currency`. Elles n’ont pas de taux ni de date de taux. Les fenêtres sont calculées avec `time.Time` en UTC du processus si l’appelant n’envoie pas d’instant. Les transactions annulées ne sont pas exclues.
