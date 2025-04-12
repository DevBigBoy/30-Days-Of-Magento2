# 📜 Declarative Schema in Magento 2

Magento 2.3+ introduced **Declarative Schema** as a modern, cleaner way to define and manage database tables. This replaces the old `InstallSchema` and `UpgradeSchema` scripts with an XML-based approach that simplifies database versioning and structural changes.

---

## 🧠 What is Declarative Schema?

Declarative Schema is a way to **declare your database structure using XML** files. Magento compares this declared structure with the actual database and **automatically applies any needed changes** during deployment or upgrades.

> No more manual table versioning or upgrade scripts! 🎉

---

## 📁 File Location

Declarative schema files live inside your module:

```
app/code/Vendor/ModuleName/etc/db_schema.xml
```

Magento scans this file when you run:
```bash
bin/magento setup:upgrade
```

---

## 🧱 Basic Example

Here’s a simple example of a table definition using declarative schema:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <table name="vendor_todo_item" resource="default" engine="innodb" comment="To-Do Item Table">
        <column xsi:type="int" name="todo_id" nullable="false" identity="true" unsigned="true" comment="Todo ID"/>
        <column xsi:type="varchar" name="title" length="255" nullable="false" comment="Title"/>
        <column xsi:type="text" name="description" nullable="true" comment="Description"/>
        <column xsi:type="timestamp" name="created_at" default="CURRENT_TIMESTAMP" nullable="false" comment="Created At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="todo_id"/>
        </constraint>
    </table>
</schema>
```

---

## 🔗 Defining Indexes and Foreign Keys

### ➕ Index
```xml
<index referenceId="VENDOR_TODO_TITLE_IDX" indexType="btree">
    <column name="title"/>
</index>
```

### 🔐 Foreign Key
```xml
<constraint xsi:type="foreign" referenceId="VENDOR_TODO_CUSTOMER_ID_FK"
            table="vendor_todo_item" column="customer_id"
            referenceTable="customer_entity" referenceColumn="entity_id"
            onDelete="CASCADE"/>
```

---

## 🧰 Whitelist File

Magento keeps track of what schema your module is responsible for via a `db_schema_whitelist.json` file. This is auto-generated and should be **committed to version control**.

Generate/update it with:

```bash
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module
```

This ensures Magento won’t touch tables your module doesn’t own.

---

## 🔄 Applying Changes

To apply any schema changes:

```bash
bin/magento setup:upgrade
```

Magento compares your XML schema with the actual DB and applies only the required changes. You don’t have to manually check or run incremental upgrades.

---

## 🧪 Data Patches for Inserting Data

Declarative Schema handles structure only. To insert/update data, use **Data Patches**:

```php
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddDefaultTodos implements DataPatchInterface
{
    public function apply()
    {
        // Your insert logic
    }
}
```

---

## 🔄 Migration from Legacy Schema Scripts

If you're upgrading a legacy module:

1. Convert `InstallSchema`/`UpgradeSchema` logic into `db_schema.xml`.
2. Run:
   ```bash
   bin/magento setup:upgrade
   ```
3. Generate whitelist:
   ```bash
   bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module
   ```

You can safely remove the old `Setup` classes once migration is verified.

---

## 💡 Best Practices

- Use descriptive table and column names
- Always update your `db_schema_whitelist.json`
- Group XML blocks by table to keep schema readable
- Never manually edit DB tables – always use `db_schema.xml`

---

## ✅ Benefits Recap

| Benefit | Description |
|--------|-------------|
| 🔄 Auto-migrations | Magento handles diffs and syncs automatically |
| 🧼 Cleaner structure | No more `InstallSchema` or upgrade chains |
| 🔍 Track ownership | Whitelist file tracks what your module controls |
| 🚀 CI/CD ready | Safe for automated deployment pipelines |

---

## 🔗 Resources

- Magento DevDocs: [Declarative Schema](https://developer.adobe.com/commerce/php/development/components/declarative-schema/)
- Magento's XML Schema Reference: `urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd`
- CLI: `bin/magento setup:db-declaration:generate-whitelist`
