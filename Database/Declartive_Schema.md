# Declarative Schema in Magento 2

Declarative schema aims to simplify the Adobe Commerce and Magento Open Source installation and upgrade processes. Previously, developers had to write database scripts in PHP for each new version of Adobe Commerce and Magento Open Source. Various scripts were required for. Magento 2 introduced Declarative Schema as a more modern and efficient approach to defining the database structure compared to the traditional Setup Scripts approach. The declarative schema was introduced in Magento 2.3 and is designed to simplify database schema management, improve upgrade processes, and reduce the complexity of handling different versions of the database schema.

This guide will provide an in-depth explanation of Declarative Schema, how it works, and how you can use it in your Magento 2 module development.

- What is Declarative Schema?
- Structure of a Declarative Schema
- How to Define a Declarative Schema in Magento 2
- What Happens During Module Installation/Upgrade?
- How to Manage Indexers with Declarative Schema
- Best Practices for Using Declarative Schema


Magento 2 introduced **Declarative Schema** as a more modern and efficient approach to defining the database structure compared to the traditional **Setup Scripts** approach. The declarative schema was introduced in **Magento 2.3** and is designed to simplify database schema management, improve upgrade processes, and reduce the complexity of handling different versions of the database schema.

This guide will provide an in-depth explanation of **Declarative Schema**, how it works, and how you can use it in your Magento 2 module development.

---

## **1. What is Declarative Schema?**

**Declarative Schema** is a method for defining the database structure of your Magento 2 module in an XML file rather than using PHP-based setup scripts. It is based on a **declarative** approach where the structure of the database (tables, indexes, columns, constraints, etc.) is defined in a single XML file. Magento's upgrade process then takes care of making the necessary changes to the database schema, such as creating tables, adding columns, or modifying constraints.

**Advantages of Declarative Schema:**
- **Simpler schema management**: You no longer need to manually create upgrade scripts for each schema change.
- **Database state consistency**: Magento tracks the final state of the schema, making it easier to ensure that your database is in sync with the defined schema.
- **Faster upgrades**: Magento uses the declarative schema to optimize the upgrade process and apply only the required changes to the database.
- **Less room for errors**: Since you describe the schema directly, there is less chance of having errors in the upgrade process, which was common in the older setup scripts approach.

---

### **2. Structure of a Declarative Schema**

In Magento 2, declarative schema definitions are placed inside a **`db_schema.xml`** file, located in the **`etc`** directory of the module.

Here is a basic structure of `db_schema.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Db\Schema:etc/db_schema.xsd">
    <table name="example_table" resource="default" engine="innodb" comment="Example Table for Declarative Schema">
        <column xsi:type="int" name="entity_id" nullable="false" identity="true" unsigned="true" primary="true" comment="Entity ID"/>
        <column xsi:type="varchar" name="name" nullable="false" length="255" comment="Name"/>
        <column xsi:type="text" name="description" nullable="true" comment="Description"/>
        <index name="NAME" type="unique">
            <column name="name"/>
        </index>
        <constraint xsi:type="foreign" referenceTable="another_table" referenceColumn="entity_id">
            <column name="another_table_id"/>
        </constraint>
    </table>
</schema>
```

#### **Explanation of the Main Tags:**

1. **`<schema>`**: The root element of the file. It defines the schema version and location of the XML schema for validation (`db_schema.xsd`).
   
2. **`<table>`**: Defines a database table. Each table can contain several child elements, such as columns, indexes, and constraints.

   - **`name`**: The name of the table.
   - **`resource`**: Specifies the database connection resource (typically "default").
   - **`engine`**: The storage engine of the table (e.g., `innodb` or `myisam`).
   - **`comment`**: An optional comment for the table.

3. **`<column>`**: Defines a column in the table.

   - **`xsi:type`**: Specifies the data type of the column (e.g., `int`, `varchar`, `text`).
   - **`name`**: The name of the column.
   - **`nullable`**: Whether the column allows null values (set to `"false"` for non-nullable columns).
   - **`identity`**: Defines if the column is auto-incrementing (usually for primary keys).
   - **`unsigned`**: Whether the column is unsigned (applies to numeric types).
   - **`primary`**: Indicates if the column is part of the primary key.
   - **`length`**: The length of the column (used for types like `varchar`).
   - **`comment`**: An optional comment for the column.

4. **`<index>`**: Defines an index on one or more columns.

   - **`name`**: The name of the index.
   - **`type`**: The type of the index (e.g., `unique`, `index`).
   - **`<column>`**: Defines which columns should be indexed.

5. **`<constraint>`**: Defines a constraint on a table, such as a foreign key.

   - **`xsi:type`**: The type of the constraint (`foreign` for foreign keys).
   - **`referenceTable`**: The table that is referenced by the foreign key.
   - **`referenceColumn`**: The column in the referenced table.
   - **`<column>`**: Defines the column that will hold the foreign key.

---

### **3. How to Define a Declarative Schema in Magento 2**

Let’s walk through how to use the declarative schema in your own Magento 2 module.

#### **Step 1: Create the `db_schema.xml` File**

Inside your custom module, create a `db_schema.xml` file under the `etc` folder.

For example:
```bash
app/code/YourVendor/YourModule/etc/db_schema.xml
```

#### **Step 2: Define the Database Structure**

Let’s say you want to create a simple table called `yourmodule_sample_table`. Here's how you can define it:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Db\Schema:etc/db_schema.xsd">
    <table name="yourmodule_sample_table" resource="default" engine="innodb" comment="Sample Table for YourModule">
        <column xsi:type="int" name="sample_id" nullable="false" identity="true" unsigned="true" primary="true" comment="Sample ID"/>
        <column xsi:type="varchar" name="sample_name" nullable="false" length="255" comment="Sample Name"/>
        <column xsi:type="text" name="sample_description" nullable="true" comment="Sample Description"/>
    </table>
</schema>
```

This creates a table with the following columns:
- `sample_id` (primary key, auto-incrementing, unsigned integer)
- `sample_name` (non-nullable string, up to 255 characters)
- `sample_description` (nullable text field)

#### **Step 3: Add Upgrade Logic Using `db_schema.xml`**

In case you want to add upgrades to your schema (e.g., adding new columns or modifying existing ones), you don’t need separate `UpgradeSchema` scripts. Instead, Magento handles the changes in a declarative way. For example, if you want to add a new column to the table, you can simply modify the `db_schema.xml` file like so:

```xml
<column xsi:type="varchar" name="sample_status" nullable="false" length="100" comment="Sample Status"/>
```

Magento will automatically compare the existing database schema with the new definition and add the `sample_status` column during the next upgrade.

---

### **4. What Happens During Module Installation/Upgrade?**

Magento uses the **`setup:upgrade`** command to handle declarative schema updates. When you run the command, Magento compares the current state of the database schema with the definitions in the `db_schema.xml` file and applies the necessary changes (like creating tables, adding columns, or updating indexes).

- **First installation**: The schema is created as defined in `db_schema.xml`.
- **Subsequent upgrades**: Magento compares the current schema with the new `db_schema.xml` and performs necessary updates (like adding or modifying columns, creating new indexes, etc.).

---

### **5. How to Manage Indexers with Declarative Schema**

In the `db_schema.xml` file, you can define **related indexers** for your tables. This helps ensure that the Magento indexers are updated after schema changes, keeping your data in sync for faster lookups.

For example, you can define an indexer for the `yourmodule_sample_table` table like this:

```xml
<relatedIndexer entity="yourmodule_sample" name="yourmodule_sample_grid"/>
```

This ensures that whenever the `yourmodule_sample_table` is modified, the associated `yourmodule_sample_grid` index is also updated.

---

### **6. Best Practices for Using Declarative Schema**

- **Always use declarative schema for creating and modifying tables**: It simplifies your codebase and ensures the schema stays in sync.
- **Avoid using setup scripts for schema changes**: Instead, prefer declarative XML configurations for managing database structure changes.
- **Make sure your `db_schema.xml` is well-organized**: Keep it clean and readable, especially when dealing with more complex schema definitions with relationships and constraints.
- **Version control**: Keep your `db_schema.xml` file under version control, as it defines the actual database structure for your module.

---

### **7. Conclusion**

Declarative Schema in Magento 2 is a powerful and efficient way to define your module's database structure. It eliminates the need for complex setup scripts and ensures smoother upgrades. By using `db_schema.xml`, you can easily define tables, columns, indexes, and constraints, and Magento will automatically take care of upgrading the database schema during the upgrade process.

By adopting declarative schema, Magento module developers can create more maintainable, efficient, and error-free database structures, reducing manual intervention and making upgrades less error-prone.

## examples of common database operations

Let's walk through **examples of common database operations** you can perform in Magento 2 using the **Declarative Schema** (`db_schema.xml`). These operations include:

1. **Create a Table**
2. **Drop a Table**
3. **Rename a Table**
4. **Add a Column to a Table**
5. **Drop a Column from a Table**
6. **Change the Column Type**
7. **Rename a Column**
8. **Add an Index**
9. **Create a Foreign Key**
10. **Drop a Foreign Key**
11. **Recreate a Foreign Key**
12. **Other Tasks**: Disable a Module

---

### **1. Create a Table**

To create a new table, you define a `<table>` tag within `db_schema.xml`. Below is an example for creating a `customer_example` table:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Db\Schema:etc/db_schema.xsd">
    <table name="customer_example" resource="default" engine="innodb" comment="Example Table for Customer">
        <column xsi:type="int" name="entity_id" nullable="false" identity="true" unsigned="true" primary="true" comment="Entity ID"/>
        <column xsi:type="varchar" name="customer_name" nullable="false" length="255" comment="Customer Name"/>
        <column xsi:type="varchar" name="email" nullable="false" length="255" comment="Customer Email"/>
    </table>
</schema>
```

- This creates a table named `customer_example` with columns for `entity_id`, `customer_name`, and `email`.

---

### **2. Drop a Table**

Dropping a table is not done directly in the declarative schema but through an upgrade script or module uninstall. However, in the **old approach** using `InstallSchema` or `UpgradeSchema`, you'd define:

```php
$this->setup->getConnection()->dropTable($this->setup->getTable('customer_example'));
```

Since we use **Declarative Schema**, the table will be dropped when it's removed from the schema definition.

---

### **3. Rename a Table**

Magento's **Declarative Schema** does not directly support table renaming. You would need to use an upgrade script for this. However, in **older Magento versions** with setup scripts, you'd rename a table like this:

```php
$this->setup->getConnection()->renameTable(
    $this->setup->getTable('old_table_name'),
    $this->setup->getTable('new_table_name')
);
```

---

### **4. Add a Column to a Table**

To add a column to an existing table, use the `<column>` tag within the `<table>` definition. Here’s an example:

```xml
<column xsi:type="varchar" name="phone" nullable="true" length="20" comment="Phone Number"/>
```

This adds a `phone` column (nullable) to the `customer_example` table.

---

### **5. Drop a Column from a Table**

To drop a column, you can't do it directly in **Declarative Schema**. Instead, you need to modify the `db_schema.xml` and Magento will compare and drop the column during the upgrade. Here's how you would define the removal of the `phone` column:

```xml
<!-- Simply remove the definition of the 'phone' column from db_schema.xml -->
```

Magento will automatically remove the column during the `setup:upgrade` process.

---

### **6. Change the Column Type**

To change the column type, you modify the column definition in `db_schema.xml`. Here’s an example where we change the `customer_name` column to be a `TEXT` type:

```xml
<column xsi:type="text" name="customer_name" nullable="false" comment="Customer Name"/>
```

Magento will compare the existing schema and modify the column type during an upgrade.

---

### **7. Rename a Column**

Renaming a column requires modifying the column name in `db_schema.xml`. This can be done by removing the old column and adding a new one with the desired name.

```xml
<column xsi:type="varchar" name="customer_first_name" nullable="false" length="255" comment="Customer First Name"/>
```

Magento will handle this during an upgrade, but you'll need to manage data migration if necessary (e.g., transferring data from the old column name to the new column name).

---

### **8. Add an Index**

Indexes help speed up database lookups. Here’s how to add an index on the `email` column:

```xml
<index name="EMAIL_INDEX" type="index">
    <column name="email"/>
</index>
```

This creates an index on the `email` column in the `customer_example` table.

---

### **9. Create a Foreign Key**

A foreign key constraint links columns from one table to another. Here’s how to add a foreign key from `customer_example` to the `customer` table:

```xml
<constraint xsi:type="foreign" referenceTable="customer" referenceColumn="entity_id">
    <column name="customer_id"/>
</constraint>
```

This creates a foreign key on the `customer_id` column in `customer_example` that references the `entity_id` column in the `customer` table.

---

### **10. Drop a Foreign Key**

To drop a foreign key, simply remove the foreign key constraint from the `db_schema.xml`. Magento will automatically drop it during the upgrade.

```xml
<!-- Remove the <constraint> definition -->
```

---

### **11. Recreate a Foreign Key**

To recreate a foreign key, you’d define the foreign key constraint again in the `db_schema.xml` file. This can be done like so:

```xml
<constraint xsi:type="foreign" referenceTable="another_table" referenceColumn="entity_id">
    <column name="foreign_key_column"/>
</constraint>
```

This reintroduces the foreign key between `customer_example` and `another_table`.

---

### **12. Other Tasks: Disable a Module**

Disabling a module is not related to the database schema directly but is a Magento 2 administrative operation. You can disable a module with the following command:

```bash
php bin/magento module:disable YourVendor_YourModule
```

This disables the module and prevents it from being executed in Magento.

---

### **Conclusion**

In Magento 2, **Declarative Schema** simplifies the process of managing database structure changes by using XML-based schema definitions. Operations like **creating tables, adding columns, creating foreign keys, and indexing** are now easier to handle, as Magento automatically manages upgrades based on the schema changes. However, for operations like **renaming tables or columns**, you may need to rely on upgrade scripts when working with **Declarative Schema**.

This approach minimizes manual intervention, reduces errors, and simplifies the upgrade process. Using declarative schema is a modern best practice when developing Magento modules.

