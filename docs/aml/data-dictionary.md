# Dictionnaire de données AML

Les types Marble réellement acceptés par le data model sont `Bool`, `Int`, `Float`, `String`, `Timestamp`, `IpAddress` et `Coords` (`api/models/data_model.go`). Il n’existe pas de type `Decimal`, `Array` ni `Enum` comme type de colonne. Une sémantique `enum`, `country`, `currency` ou `monetary_amount` peut décorer un champ (`api/models/data_model_field_semantic.go`). Les listes du modèle cible doivent donc être des chaînes, plusieurs champs, ou du JSON hors du data model Marble.

Légende : **déployé** = créé par `init-config.yaml` ; **module** = struct ou table `bankaml` ; **absent** = prévu par la cible et non trouvé dans le code.

## Customer — table Marble `clients` (déployé)

| Cible | Réel | Type réel | État |
|---|---|---|---|
| id | `object_id`, plus `id` réservé Marble | String, sémantique `id` | déployé |
| customer_reference | `object_id` sert de référence | String | déployé, un seul identifiant |
| legal_name / first_name / last_name | `full_name` | String, sémantique `name` | partiel |
| date_of_birth | `birth_date` | String, sans sémantique `date_of_birth` | partiel |
| nationality_codes | `nationality` | String, sémantique `country` | une seule valeur |
| customer_type, place_of_birth, incorporation_date, registration_number, tax_identifier, country_of_residence, registered_country, occupation, industry_code, customer_since, customer_status | — | — | absents |
| created_at | — | — | absent de la table |
| updated_at | `updated_at` | Timestamp, sémantique `last_update` | déployé |

Le module `ClientInput` ajoute `client_ref`, `entity_type`, `name`, `alias`, `birth_date`, `nationality`, `country`, `identifiers`. Ces champs voyagent dans l’appel API. Ils ne sont pas des colonnes Marble.

## CustomerProfile

`ClientProfile` (`api/bankaml/types.go`) contient `type`, `country`, `sector`, `expected_monthly_volume`, `expected_monthly_count`, `kyc_updated_at`, `relationship`. Il est porté par la requête de score. Il n’a pas de table, pas de `version`, pas de `valid_from` / `valid_to`, pas de `kyc_status`, pas de `source_of_funds` ni de `source_of_wealth`.

Marble conserve `valid_from` et `valid_until` sur chaque objet ingéré. Cela historise la ligne ingérée, pas un profil KYC approuvé.

## Account — absent

Aucun objet `accounts`. `bankaml.Transaction.AccountID` est une chaîne libre. Pas de titulaire multiple, pas de `AccountHolder`, pas de devise de compte, pas de statut.

## Transaction

### Table Marble `transactions` (déployé)

| Champ | Type | Sémantique | Obligatoire |
|---|---|---|---|
| object_id | String | id | oui |
| updated_at | Timestamp | last_update | oui |
| amount | Float | monetary_amount | oui |
| currency | String | currency | non |
| country | String | country | oui |
| client_id | String | foreign_key vers `clients` | non |

Convention de signe : non documentée dans l’initialisation. La règle native compare `amount >= 10000` sans distinguer débit et crédit. Pas de clé d’idempotence `(source_system, transaction_reference)`. Pas de statut, de date comptable distincte, de taux de change, de contrepartie, ni d’annulation.

### Struct `bankaml.Transaction` (module)

`id`, `client_id`, `account_id`, `direction`, `amount` (`float64`), `currency`, `country`, `channel`, `counterparty`, `counterparty_account`, `type`, `at`. La population est fournie par l’appelant. Le module ne lit pas la table Marble `transactions`.

Les agrégations du module additionnent `amount` même si `currency` diffère. C’est une limite connue, pas une conversion.

## Counterparty — absent comme entité

`counterparty` et `counterparty_account` sont des chaînes sur la transaction du module. Deux noms proches ne sont pas fusionnés, parce qu’aucun rapprochement n’est implémenté. Il n’y a pas de `normalized_name`, ni de hash de compte, ni de `first_seen_at`.

## Relations — absentes

`CustomerRelationship`, `AccountHolder` et `BeneficialOwner` n’ont ni table Marble ni table `bankaml`. Le seul lien déployé est `transactions` **belongs_to** `clients` par `client_id`.

## ScreeningResult — table `bankaml_matches` (module)

| Cible | Colonne réelle |
|---|---|
| id | `id` |
| customer_id | `client_ref` |
| provider | `source` sur `bankaml_searches` (`opensanctions`) |
| external_entity_id | `entity_id` |
| matched_name | `entity_name` |
| match_score | `score` double precision, vide si l’API ne le fournit pas |
| match_reasons | `explanation`, `matched`, `mismatched` |
| categories | `datasets` |
| searched_at | `consulted_at` |
| status | `qualification` : `to_review`, `in_analysis`, `dismissed`, `confirmed`, `escalate`, `error` |
| reviewed_by / reviewed_at / review_reason | `qualified_by`, `qualified_at`, `qualification_reason` |
| raw_response_reference | `raw` |
| dataset_version, screening_type, previous_result_id, source_references | absents |

Les recherches sont dans `bankaml_searches` (`query`, `query_hash`, `transmitted`, `status`, `error`). Un résultat ultérieur ne remplace pas la ligne précédente. Une disparition est un écart `missing`, pas une radiation.

Le screening natif Marble a ses propres tables de scénario. Leur publication reste refusée sans entitlement sanctions. Leurs colonnes internes ne sont pas redupliquées ici.

## RiskAssessment — table `bankaml_scores` (module)

`client_ref`, `total`, `components`, `factors`, `formula_version`, `sanctions_status`, `review_required`, `missing`, `reference_at`, `reason`, `created_by`, `created_at`.

Le total utilise les poids initiaux KYC 0,35 / transactions 0,40 / alertes 0,25. Les classes par défaut sont 30, 60 et 85. Ce n’est pas un modèle réglementaire. Un match `confirmed`, `to_review`, `in_analysis`, `escalate` ou `error` positionne `sanctions_status` et `review_required` sans entrer dans le total.

`assessment_type`, `valid_until`, `reason_codes` et `source_data_snapshot_reference` sont absents. Les scores sont ajoutés, pas écrasés.

## Alert et InvestigationCase

Il n’y a pas de table `bankaml_alerts`. L’alerte est le JSON `bankaml_cases.alert`, avec `alert_key` unique par organisation.

`bankaml_cases` contient `client_ref`, `status`, `priority`, `assignee`, `team`, `due_at`, `scenario_code`, `score`, `screening`, `closure_reason`, `investigator`, `level1_actor`, `level2_actor`, `opened_by`, `created_at`, `updated_at`.

Statuts du module : `new`, `to_qualify`, `investigating`, `waiting_info`, `escalated`, `second_level`, `validated`, `closed`.

Statuts du cas Marble natif : `pending`, `investigating`, `closed` (`api/models/case.go`). Pas de quatre yeux sur ce circuit.

`CaseEvent` du module est `bankaml_events` : `object_type`, `object_id`, `action`, `actor`, `reason`, `payload`, `created_at`. Pas de `previous_value` / `new_value` séparés : le changement est dans `payload`.

`alert_reference`, `transaction_id`, `scenario_version_id`, `execution_id`, `deduplication_key`, `final_disposition` et `case_reference` dédiés sont absents. La déduplication des dossiers utilise `(org_id, alert_key)`.

## ScenarioDefinition, ScenarioVersion, ScenarioExecution

`bankaml_scenarios` : clé `(org_id, code, version)`, `description`, `params`, `severity`, `priority`, `status`, `enabled`, `author`, `approver`, `updated_at`. Les scénarios UC naissent en brouillon, désactivés. Une nouvelle version est `pending_approval` et n’est pas activée par son auteur.

Marble natif versionne par itération de scénario, puis commit et publication. Les deux scénarios initialisés sont « AML screening » sur `clients` et « AML transaction check » sur `transactions`.

`ScenarioExecution` n’a pas de table dans le module. `bankaml_batches` décrit un batch de screening (`mode`, `status`, compteurs, `population`), pas une exécution de scénario transactionnel. Marble enregistre des décisions lors d’une décision de scénario natif. Le nombre d’enregistrements lus par UC01–UC10 n’est pas persisté.

## AuditEvent et DataSourceRecord

`bankaml_events` couvre les actions du module. Il n’a pas `correlation_id` ni de références de snapshots avant/après.

`DataSourceRecord` est absent. `source_system` n’est pas stocké sur `clients` ni `transactions`.

## Données manquantes

Une date ou un pays vide reste vide. Le score liste les champs manquants dans `missing`. L’absence d’un résultat de screening n’est pas un risque nul et n’est pas une radiation. Une donnée KYC absente n’est pas convertie en score zéro silencieux : elle est signalée dans `missing` lorsque le calcul passe par le profil.
