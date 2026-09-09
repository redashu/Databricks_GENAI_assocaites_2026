# Dataset README

## 1. incidents_1000.csv

Purpose: historical IT incident records.

### Fields

- `incident_id`
- `service`
- `severity`
- `status`
- `assigned_team`
- `incident_date`
- `description`
- `customer_impact`
- `resolution_minutes`

### Example question

> How many P1 incidents affected the payment service?

This is structured data, so it is best suited for SQL or Genie-style analysis rather than pure RAG.

## 2. support_tickets_1000.csv

Purpose: IT helpdesk and support request data.

### Fields

- `ticket_id`
- `created_date`
- `category`
- `priority`
- `status`
- `support_team`
- `ticket_text`
- `customer_impact`
- `resolution_minutes`

### Example question

> What are the most common VPN-related support issues?

This is also structured data, but the `ticket_text` field can later be useful for semantic search or retrieval-based analysis.

## 3. service_catalog_1000.csv

Purpose: enterprise service metadata and operational characteristics.

### Fields

- `service_id`
- `service_name`
- `business_domain`
- `owner_team`
- `criticality`
- `sla_hours`
- `environment`
- `support_window`

### Example question

> What is the SLA for a critical production payment service?

This dataset is ideal for operational reporting and business context enrichment for GenAI applications.