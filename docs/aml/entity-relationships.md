# Relations entre entités

## Relation déployée

```mermaid
erDiagram
  clients {
    string object_id
    timestamp updated_at
    string full_name
    string birth_date
    string nationality
  }
  transactions {
    string object_id
    timestamp updated_at
    float amount
    string currency
    string country
    string client_id
  }
  clients ||--o{ transactions : "belongs_to"
```

Source : `k8s-gitops/apps/aml/base/init-config.yaml`. Le lien est `link_type: belongs_to`, champ enfant `client_id`, parent `clients`. Un compte, une contrepartie ou un second titulaire ne participent pas à ce graphe.

Marble peut exprimer d’autres liens de data model. Aucun autre lien n’est créé par l’initialisation actuelle.

## Relations du module `bankaml`

```mermaid
erDiagram
  organizations ||--o{ bankaml_searches : org_id
  bankaml_searches ||--o{ bankaml_matches : search_id
  organizations ||--o{ bankaml_cases : org_id
  organizations ||--o{ bankaml_events : org_id
  organizations ||--o{ bankaml_scores : org_id
  organizations ||--o{ bankaml_scenarios : org_id
  organizations ||--o{ bankaml_formulas : org_id
  organizations ||--o{ bankaml_settings : org_id
  organizations ||--o{ bankaml_batches : org_id
  bankaml_batches ||--o{ bankaml_batch_items : batch_id
```

`bankaml_events.org_id` et `bankaml_batch_items.org_id` ne sont pas des clés étrangères dans la migration. Les autres `org_id` référencent `organizations(id)`.

`bankaml_cases` ne référence pas `bankaml_matches` ni `bankaml_scenarios` par clé étrangère. Le lien est `client_ref`, `scenario_code` et le JSON `alert` / `screening`.

## Relations cibles non créées

| Relation cible | État |
|---|---|
| Customer 1—n CustomerProfile versionné | le profil est un objet de requête |
| Account n—n Customer via AccountHolder | absent |
| Transaction vers deux comptes et deux contreparties | absent |
| CustomerRelationship | absent |
| BeneficialOwner | absent |
| ScreeningResult.previous_result_id | absent ; l’historique est une nouvelle ligne |
| Alert vers ScenarioVersion et ScenarioExecution | l’alerte est dans le dossier |
| InvestigationCase vers plusieurs alertes | un dossier a un `alert_key` |

## Rapprochement

Le module ne fusionne pas deux contreparties parce que leurs noms se ressemblent. Il ne fusionne pas non plus un client Marble et un `client_ref` du module : les deux identifiants doivent être alimentés ensemble par l’appelant. Cette stratégie est explicite parce qu’aucune autre n’est codée.

Les liens inférés et les liens confirmés par un document KYC ne sont pas distingués, parce que les tables de relation n’existent pas.
