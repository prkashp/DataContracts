# Data Contracts
Data contracts helps in defining a fixed format between a data producer and consumer.

# Motivation
In most of case an organisation has separate teams either producing or providing new data and teams who use that data for reporting or other data products. And these teams are usually disconnected. For Data consumers it is utmost important to have a stable data pipelines and dashboards which can be improved by not having any unknowns in data structure, format and semantics. 

# Agreements
Data contracts can be roughly divided into 4 sub-parts
- Schema: This can have following coverage
  * Field name (Column name)
  * Mandate: Any constraints like default and nullability
  * Data Types
    - Format
    - Length: Incase of integer precision and scale
    - Categorical Column: Sets of allowed values
- Semantics: Generally business rules that required enforcement
- SLAs: Commitment on availablity and freshness of data
- Governance: Keeping compliance in check with local laws


