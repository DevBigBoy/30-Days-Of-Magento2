# 📘 Declarative Schema Field & Constraint Reference – With Examples

---

## 🧱 Basic Schema Layout (Boilerplate)

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
</schema>
```

This is the root of every `db_schema.xml` file.

---

## 🧩 Data Types (All Common Column Types)

### 🔢 Integer (int)

```xml
<column xsi:type="int" name="entity_id" nullable="false" identity="true" unsigned="true" comment="ID"/>
```

- `identity="true"` makes it AUTO_INCREMENT
- `unsigned="true"` is best practice for IDs

---

### 🔡 Varchar (String)

```xml
<column xsi:type="varchar" name="title" length="255" nullable="false" comment="Title"/>
```

- Requires `length`
- For shorter strings like names or labels

---

### 📜 Text (Long string)

```xml
<column xsi:type="text" name="description" nullable="true" comment="Description"/>
```

- No length required
- Used for large content like CMS blocks, comments, etc.

---

### 📅 Timestamp

```xml
<column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="false" comment="Created At"/>
```

- Use `default="CURRENT_TIMESTAMP"` for automatic creation
- `on_update="true"` adds auto-update on row update

---

### 🔘 Boolean

```xml
<column xsi:type="smallint" name="is_active" nullable="false" default="1" unsigned="true" comment="Is Active"/>
```

- Magento doesn't use real `BOOLEAN` type; it uses `tinyint` or `smallint` instead

---

### 💰 Decimal

```xml
<column xsi:type="decimal" name="price" precision="12" scale="4" nullable="false" default="0.0000" comment="Price"/>
```

- Used for money values
- `precision="12"` is total digits, `scale="4"` is decimals

---

### 📦 JSON

```xml
<column xsi:type="json" name="config_data" nullable="true" comment="Configuration as JSON"/>
```

- Great for storing dynamic config or settings
- Supported in MySQL 5.7+

---

### 📆 Date / Datetime

```xml
<column xsi:type="datetime" name="publish_date" nullable="true" comment="Publish Date"/>
```

- Use for full date+time tracking (vs `timestamp` which is usually for created/updated)

---

## 🔗 Constraints & Keys

### 🧷 Primary Key

```xml
<constraint xsi:type="primary" referenceId="PRIMARY">
    <column name="entity_id"/>
</constraint>
```

---

### 🔒 Unique Key

```xml
<constraint xsi:type="unique" referenceId="UNQ_VENDOR_TODO_TITLE">
    <column name="title"/>
</constraint>
```

---

### 🔐 Foreign Key

```xml
<column xsi:type="int" name="customer_id" nullable="true" unsigned="true" comment="Customer ID"/>
<constraint xsi:type="foreign" referenceId="FK_VENDOR_TODO_CUSTOMER_ID"
            table="vendor_todo_item" column="customer_id"
            referenceTable="customer_entity" referenceColumn="entity_id"
            onDelete="CASCADE"/>
```

- Enforces referential integrity
- `onDelete="CASCADE"` will delete dependent records if parent is deleted

---

## 🧮 Indexes

### Simple Index

```xml
<index indexType="btree" referenceId="IDX_VENDOR_TODO_TITLE">
    <column name="title"/>
</index>
```

### Composite Index

```xml
<index indexType="btree" referenceId="IDX_VENDOR_TODO_TITLE_DATE">
    <column name="title"/>
    <column name="created_at"/>
</index>
```

- Good for search performance

---

## 🧪 Complete Example

Here’s a full example table using most of the above:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <table name="vendor_todo_item" resource="default" engine="innodb" comment="To-Do Item Table">
        <column xsi:type="int" name="todo_id" identity="true" unsigned="true" nullable="false" comment="Todo ID"/>
        <column xsi:type="varchar" name="title" length="255" nullable="false" comment="Title"/>
        <column xsi:type="text" name="description" nullable="true" comment="Description"/>
        <column xsi:type="json" name="settings" nullable="true" comment="JSON Config"/>
        <column xsi:type="decimal" name="priority" precision="12" scale="2" default="1.00" nullable="false" comment="Priority Level"/>
        <column xsi:type="timestamp" name="created_at" default="CURRENT_TIMESTAMP" nullable="false" comment="Created At"/>
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="true" comment="Customer Reference"/>
        
        <!-- Primary Key -->
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="todo_id"/>
        </constraint>

        <!-- Unique Key -->
        <constraint xsi:type="unique" referenceId="UNQ_VENDOR_TODO_TITLE">
            <column name="title"/>
        </constraint>

        <!-- Foreign Key -->
        <constraint xsi:type="foreign" referenceId="FK_TODO_CUSTOMER_ID"
                    table="vendor_todo_item" column="customer_id"
                    referenceTable="customer_entity" referenceColumn="entity_id"
                    onDelete="SET NULL"/>
        
        <!-- Index -->
        <index indexType="btree" referenceId="IDX_VENDOR_TODO_TITLE">
            <column name="title"/>
        </index>
    </table>
</schema>
```

---

## 🔗 RELATIONSHIP TYPES IN DECLARATIVE SCHEMA (Magento 2)

Magento doesn’t have explicit `RELATIONSHIP` keywords like some ORMs, but **you implement relationships via foreign keys and table design**. Here's how to represent:

---

## 1️⃣ One-to-One Relationship

### 💡 Concept

Each record in Table A corresponds to *one and only one* record in Table B.

### 🧱 Schema Example

```xml
<table name="vendor_user_profile" resource="default" engine="innodb" comment="User Profile">
    <column xsi:type="int" name="user_id" nullable="false" unsigned="true" comment="User ID"/>
    <column xsi:type="varchar" name="bio" length="500" nullable="true" comment="User Bio"/>

    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="user_id"/>
    </constraint>

    <constraint xsi:type="foreign" referenceId="FK_PROFILE_USER"
                table="vendor_user_profile" column="user_id"
                referenceTable="vendor_user" referenceColumn="user_id"
                onDelete="CASCADE"/>
</table>
```

### ✅ Notes

- Both tables share the same `user_id`
- Use `user_id` as both **PK** and **FK** in `vendor_user_profile`

---

## 2️⃣ One-to-Many Relationship

### 💡 Concept

One record in Table A relates to *many* records in Table B.

### 🧱 Example

```xml
<table name="vendor_blog_post" resource="default" engine="innodb" comment="Blog Posts">
    <column xsi:type="int" name="author_id" nullable="false" unsigned="true" comment="Author ID"/>
    <!-- ... -->

    <constraint xsi:type="foreign" referenceId="FK_POST_AUTHOR"
                table="vendor_blog_post" column="author_id"
                referenceTable="vendor_author" referenceColumn="author_id"
                onDelete="CASCADE"/>
</table>
```

### ✅ Notes

- Each `vendor_author` can have **many** `vendor_blog_post`
- The FK is placed on the **many-side** (`vendor_blog_post`)

---

## 3️⃣ Many-to-Many Relationship

### 💡 Concept

Many records in Table A relate to many in Table B. Requires a **pivot table**.

### 🧱 Example

```xml
<table name="vendor_product_tag" resource="default" engine="innodb" comment="Product-Tag Link">
    <column xsi:type="int" name="product_id" nullable="false" unsigned="true"/>
    <column xsi:type="int" name="tag_id" nullable="false" unsigned="true"/>

    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="product_id"/>
        <column name="tag_id"/>
    </constraint>

    <constraint xsi:type="foreign" referenceId="FK_PRODUCT_TAG_PRODUCT"
                table="vendor_product_tag" column="product_id"
                referenceTable="catalog_product_entity" referenceColumn="entity_id"
                onDelete="CASCADE"/>

    <constraint xsi:type="foreign" referenceId="FK_PRODUCT_TAG_TAG"
                table="vendor_product_tag" column="tag_id"
                referenceTable="vendor_tag" referenceColumn="tag_id"
                onDelete="CASCADE"/>
</table>
```

### ✅ Notes

- The link table uses a **composite primary key**
- This is exactly how `catalog_category_product` works in Magento

---

## 🧠 Composite Primary Key Example

Already shown in the many-to-many table above. Just remember:

```xml
<constraint xsi:type="primary" referenceId="PRIMARY">
    <column name="col1"/>
    <column name="col2"/>
</constraint>
```

---

## 🔘 ENUM-like Constraint (Simulating ENUMs)

Magento doesn’t support native ENUMs, but you can enforce value choices in code or use smallint/varchar + validation.

```xml
<column xsi:type="varchar" name="status" length="20" nullable="false" default="draft" comment="Status"/>
```

In your app logic:

```php
const STATUS_DRAFT = 'draft';
const STATUS_PUBLISHED = 'published';
const STATUS_ARCHIVED = 'archived';
```

You could also manage this through a lookup/reference table for strict FK validation.

---

## 🧼 Nullable vs Not Nullable Columns

### Required Column (Not Nullable)

```xml
<column xsi:type="int" name="user_id" nullable="false"/>
```

### Optional Column (Nullable)

```xml
<column xsi:type="text" name="notes" nullable="true"/>
```

Magento treats nullable columns seriously, especially for validation.

---

## 🛠 Optional Extras You Should Know

### 🔒 Engine Options

```xml
<table engine="innodb"> <!-- Default -->
```

You can also use:

- `memory` (rare)
- `myisam` (not recommended)

---

### 🔧 Auto-Increment IDs (identity)

```xml
<column xsi:type="int" identity="true" ... />
```

---

### 🧮 Default Values

```xml
<column default="0" xsi:type="int" name="stock_qty"/>
<column default="CURRENT_TIMESTAMP" xsi:type="timestamp" ... />
```

---

### 🧰 Full Indexing

```xml
<index referenceId="IDX_PRODUCT_TAG" indexType="btree">
    <column name="product_id"/>
    <column name="tag_id"/>
</index>
```

--

## ✅ TL;DR Summary

| Feature          | Example XML Tag | Notes |
|------------------|------------------|-------|
| `int`            | `xsi:type="int"` | Use for IDs |
| `varchar`        | `xsi:type="varchar" length="255"` | Max 255 characters |
| `text`           | `xsi:type="text"` | For long content |
| `decimal`        | `precision + scale` | Money, metrics |
| `json`           | `xsi:type="json"` | Magento 2.3+ |
| `timestamp`      | `default="CURRENT_TIMESTAMP"` | Tracks changes |
| `primary key`    | `xsi:type="primary"` | One per table |
| `unique`         | `xsi:type="unique"` | No duplicate values |
| `foreign key`    | `xsi:type="foreign"` | Link to other tables |
| `index`          | `xsi:type="btree"` | For performance |
