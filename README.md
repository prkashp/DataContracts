# Data Contracts
Data contracts helps in defining a fixed format between a data producer and consumer.

# Motivation(Why we require this?)
Even if data design documents can solve issue caused by bad data quality, its not the case with large organisation where teams work in parrallel and independent. In most cases, an organization has separate teams either producing or providing new data and teams that use that data for reporting or other data products. And these teams are usually disconnected. For data consumers, it is utmost important to have stable data pipelines and dashboards, which can be improve by not having any unknowns in data structure, format, or semantics. 
- Data pipelines regularly break due to data quality issues.
- Communication gap between system implementors, data engineers, and data consumers.
- No team ownership of data defects, leading to a constant untouched backlog.

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

# Solution(How data contracts can be used?)
We have an open source library to lint all kinds of schema and quality checks.To create a contract we would require a yaml file to store all information about dataset. Soda core is being used at the backend.
```yaml

dataContractSpecification: 0.1.1
id: urn:datacontract:checkout:orders-latest
info:
  title: Orders Latest
  version: 1.0.0
  description: |
    Customers orders placed from the website 
  owner: Checkout Team
  contact:
    name: John Doe (Data Product Owner)
    url: https://teams.microsoft.com/l/channel/example/checkout
servers:
  production:
    type: s3
    location: s3://datacontract-example-orders-latest/data/{model}/*.json
    format: json
    delimiter: new_line
terms:
  usage: >
    Data can be used for reports, analytics and machine learning use cases.
    Order may be linked and joined by other tables
  limitations: >
    Not suitable for real-time use cases.
    Data may not be used to identify individual customers.
    Max data processing per day: 10 TiB
  billing: 5000 USD per month
  noticePeriod: P3M
models:
  orders:
    description: One record per order. Includes cancelled and deleted orders.
    type: table
    fields:
      order_id:
        $ref: '#/definitions/order_id'
        required: true
        unique: true
      order_timestamp:
        description: The business timestamp in UTC when the order was successfully registered in the source system and the payment was successful.
        type: timestamp
        required: true
      order_total:
        description: Total amount the smallest monetary unit (e.g., cents).
        type: long
        required: true
      customer_id:
        description: Unique identifier for the customer.
        type: text
        minLength: 10
        maxLength: 20
      customer_email_address:
        description: The email address, as entered by the customer. The email address was not verified.
        type: text
        format: email
        required: true
  line_items:
    description: A single article that is part of an order.
    type: table
    fields:
      lines_item_id:
        type: text
        description: Primary key of the lines_item_id table
        required: true
        unique: true
      order_id:
        $ref: '#/definitions/order_id'
      sku:
        description: The purchased article number
        $ref: '#/definitions/sku'
definitions:
  order_id:
    domain: checkout
    name: order_id
    title: Order ID
    type: text
    format: uuid
    description: An internal ID that identifies an order in the online shop.
    example: 243c25e5-a081-43a9-aeab-6d5d5b6cb5e2
    pii: true
    classification: restricted
  sku:
    domain: inventory
    name: sku
    title: Stock Keeping Unit
    type: text
    pattern: ^[A-Za-z0-9]{8,14}$
    example: "96385074"
    description: |
      A Stock Keeping Unit (SKU) is an internal unique identifier for an article. 
      It is typically associated with an article's barcode, such as the EAN/GTIN.
examples:
  - type: csv # csv, json, yaml, custom
    model: orders
    data: |- # expressed as string or inline yaml or via "$ref: data.csv"
      order_id,order_timestamp,order_total,customer_id,customer_email_address
      "1001","2030-09-09T08:30:00Z",2500,"1000000001","mary.taylor82@example.com"
      "1002","2030-09-08T15:45:00Z",1800,"1000000002","michael.miller83@example.com"
      "1003","2030-09-07T12:15:00Z",3200,"1000000003","michael.smith5@example.com"
      "1004","2030-09-06T19:20:00Z",1500,"1000000004","elizabeth.moore80@example.com"
      "1005","2030-09-05T10:10:00Z",4200,"1000000004","elizabeth.moore80@example.com"
      "1006","2030-09-04T14:55:00Z",2800,"1000000005","john.davis28@example.com"
      "1007","2030-09-03T21:05:00Z",1900,"1000000006","linda.brown67@example.com"
      "1008","2030-09-02T17:40:00Z",3600,"1000000007","patricia.smith40@example.com"
      "1009","2030-09-01T09:25:00Z",3100,"1000000008","linda.wilson43@example.com"
      "1010","2030-08-31T22:50:00Z",2700,"1000000009","mary.smith98@example.com"
  - type: csv
    model: line_items
    data: |-
      lines_item_id,order_id,sku
      "LI-1","1001","5901234123457"
      "LI-2","1001","4001234567890"
      "LI-3","1002","5901234123457"
      "LI-4","1002","2001234567893"
      "LI-5","1003","4001234567890"
      "LI-6","1003","5001234567892"
      "LI-7","1004","5901234123457"
      "LI-8","1005","2001234567893"
      "LI-9","1005","5001234567892"
      "LI-10","1005","6001234567891"
quality:
  type: SodaCL   # data quality check format: SodaCL, montecarlo, custom
  specification: # expressed as string or inline yaml or via "$ref: checks.yaml"
    checks for orders:
      - freshness(order_timestamp) < 24h
      - row_count >= 5000
      - duplicate_count(order_id) = 0
    checks for line_items:
      - values in (order_id) must exist in orders (order_id)
      - row_count >= 5000

```

After we have our contract yaml ready. To run contracts we have below code to run.

```python
from datacontract.data_contract import DataContract

data_contract = DataContract(data_contract_file="datacontract.yaml")
run = data_contract.test()
if not run.has_passed():
    print("Data quality validation failed.")
    # Abort pipeline, alert, or take corrective actions...
```
Currently this library supports major databases and local format. To start with Postgres, Snowflake, S3, Databricks are supported along with JSON, CSV and parquet.

# Benefits
- Not too much is required to implement data contracts; having a compulsory deployment pipeline before feature merge would be sufficient.
- This also removes a huge backlog of data bugs, and new deployments only pass when data contracts are intact.
- Accurate data ownership and categorization for better data governance and data products.
- This process is completely automated and decentralized by minimizing dependencies across teams.

