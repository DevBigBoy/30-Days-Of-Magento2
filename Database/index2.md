# 🧠 Introduction

## 🔹 **Introduction**

Give a quick overview:

- What Magento's database is used for (storing products, customers, orders, configs)
- Mention it's modular, extensible, and supports multiple connections (read/write)

---

## 🔹 **Configuration**

- Show where DB config is stored: `app/etc/env.php`
- Explain typical fields: host, dbname, username, password
- Introduce connection types (`default`, `checkout`, etc.)

> Include a code block showing a sample `env.php` DB config.

---

### 🔹 **Read and Write Connections**

- Magento supports **separate read and write connections** for scaling
- Configured under `resources` in `env.php`
- Mention how load balancers or replicas might be used

```php
'resource' => [
    'default_setup' => [
        'connection' => 'default',
    ],
    'default' => [
        'host' => 'master-db',
        ...
    ],
    'read' => [
        'host' => 'replica-db',
        ...
    ]
]
```

---

### 🔹 **Running SQL Queries**

Explain **safe and unsafe** ways to interact with the DB.

#### Using Multiple Database Connections

- You can get connections from the `\Magento\Framework\App\ResourceConnection` class.
- Example: how to get a connection and run a raw query.

```php
$connection = $resource->getConnection('read');
$sql = "SELECT * FROM customer_entity";
$results = $connection->fetchAll($sql);
```

---

#### Listening for Query Events

- Magento dispatches events like `db_query_executed`
- You can create observers to listen and log specific queries

---

#### Monitoring Cumulative Query Time

- Talk about profiling tools (`bin/magento dev:profiler:enable`)
- Or using `n98-magerun2` for debugging slow queries

---

### 🔹 **Database Transactions**

- Explain `beginTransaction()`, `commit()`, `rollBack()` in Magento
- Mention when you'd use them (e.g., batch operations)

```php
$connection->beginTransaction();
try {
    // perform queries
    $connection->commit();
} catch (\Exception $e) {
    $connection->rollBack();
    throw $e;
}
```

---

### 🔹 **Connecting to the Database CLI**

- Show how to connect via CLI using credentials from `env.php`
- Mention tools like:
  - MySQL CLI
  - `n98-magerun2 db:console`
  - DBeaver or TablePlus

---

### 🔹 **Inspecting Your Databases**

- Tools to explore table structures
- Commands to list tables, view columns, analyze indexes
- ER diagrams (if applicable)

---

### 🔹 **Monitoring Your Databases**

- Cron jobs that affect DB health (reindexing, log cleaning)
- Use of slow query logs or query profiling
- Magento logging: `var/log/`, custom logging strategies

---

## 💡 Bonus Suggestion

At the end of your article, include a **“What to Read Next”** or “Next Steps” section, leading into more advanced database topics or Magento concepts like:

- Declarative schema
- Custom module DB tables
- Using repositories over direct queries
