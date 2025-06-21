# Magento 2 Database Schema - Comprehensive Learning Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Database Schema Architecture](#database-schema-architecture)
3. [Schema Evolution in Magento 2](#schema-evolution-in-magento-2)
4. [Declarative Schema (db_schema.xml)](#declarative-schema-db_schemaxml)
5. [Legacy InstallSchema & UpgradeSchema](#legacy-installschema--upgradeschema)
6. [Data Patches vs Schema Patches](#data-patches-vs-schema-patches)
7. [EAV System Deep Dive](#eav-system-deep-dive)
8. [Table Types and Relationships](#table-types-and-relationships)
9. [Indexing Strategy](#indexing-strategy)
10. [Foreign Keys and Constraints](#foreign-keys-and-constraints)
11. [Performance Considerations](#performance-considerations)
12. [Migration Strategies](#migration-strategies)
13. [Best Practices](#best-practices)
14. [Real-World Examples](#real-world-examples)
15. [Troubleshooting](#troubleshooting)

---

## Introduction

Magento 2's database schema is the foundation of the entire platform. Understanding how to properly design, implement, and maintain database schemas is crucial for building performant, scalable, and maintainable e-commerce applications.

**Key Evolution**: Magento 2.3+ introduced **Declarative Schema**, moving away from procedural InstallSchema/UpgradeSchema to a more maintainable, declarative approach.

## Database Schema Architecture

### Core Principles:

1. **Modular Design**: Each module manages its own schema
2. **Declarative Approach**: Define desired state, not migration steps
3. **EAV Pattern**: Flexible attribute system for entities
4. **Normalized Structure**: Proper relational database design
5. **Performance Optimization**: Strategic indexing and constraints

### Schema Management Layers:

```
┌─────────────────────────────────────┐
│         Application Layer           │ ← Models, Collections, Repositories
├─────────────────────────────────────┤
│         ORM Layer (Active Record)   │ ← Resource Models, Abstract Models
├─────────────────────────────────────┤
│         Schema Definition Layer     │ ← db_schema.xml, Patches
├─────────────────────────────────────┤
│         Database Adapter Layer      │ ← MySQL, PostgreSQL adapters
├─────────────────────────────────────┤
│         Physical Database Layer     │ ← MySQL/MariaDB, Tables, Indexes
└─────────────────────────────────────┘
```

## Schema Evolution in Magento 2

### Timeline and Approach Changes:

| Version | Approach | Method | Status |
|---------|----------|---------|---------|
| 2.0-2.2 | Procedural | InstallSchema/UpgradeSchema | Legacy |
| 2.3+ | Declarative | db_schema.xml + Patches | Current |
| Future | Enhanced | GraphQL schema integration | Planned |

### Benefits of Declarative Schema:

1. **Version Independence**: Define target state, not migration path
2. **Rollback Support**: Easy schema rollbacks
3. **Performance**: Optimized schema comparison
4. **Maintainability**: Single source of truth
5. **Dry-run Capability**: Preview changes before execution

## Declarative Schema (db_schema.xml)

### Basic Structure:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    
    <!-- Table Definition -->
    <table name="vendor_module_entity" resource="default" engine="innodb" comment="Custom Entity Table">
        
        <!-- Primary Key -->
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="Entity ID"/>
        
        <!-- Standard Columns -->
        <column xsi:type="varchar" name="name" nullable="false" length="255" comment="Entity Name"/>
        <column xsi:type="text" name="description" nullable="true" comment="Entity Description"/>
        <column xsi:type="decimal" name="price" scale="4" precision="12" unsigned="false" nullable="false" default="0" comment="Price"/>
        <column xsi:type="smallint" name="status" unsigned="true" nullable="false" default="1" comment="Status"/>
        
        <!-- Timestamps -->
        <column xsi:type="timestamp" name="created_at" on_update="false" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" on_update="true" nullable="false" default="CURRENT_TIMESTAMP" comment="Updated At"/>
        
        <!-- Foreign Key -->
        <column xsi:type="int" name="store_id" unsigned="true" nullable="false" comment="Store ID"/>
        
        <!-- Primary Key Constraint -->
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
        
        <!-- Foreign Key Constraint -->
        <constraint xsi:type="foreign" referenceId="VENDOR_MODULE_ENTITY_STORE_ID_STORE_STORE_ID" 
                   table="vendor_module_entity" column="store_id" 
                   referenceTable="store" referenceColumn="store_id" 
                   onDelete="CASCADE"/>
        
        <!-- Indexes -->
        <index referenceId="VENDOR_MODULE_ENTITY_NAME" indexType="btree">
            <column name="name"/>
        </index>
        
        <index referenceId="VENDOR_MODULE_ENTITY_STATUS_STORE_ID" indexType="btree">
            <column name="status"/>
            <column name="store_id"/>
        </index>
        
        <!-- Unique Index -->
        <constraint xsi:type="unique" referenceId="VENDOR_MODULE_ENTITY_NAME_STORE_ID">
            <column name="name"/>
            <column name="store_id"/>
        </constraint>
    </table>

    <!-- Junction Table for Many-to-Many Relationships -->
    <table name="vendor_module_entity_category" resource="default" engine="innodb" comment="Entity Category Relation">
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" comment="Entity ID"/>
        <column xsi:type="int" name="category_id" unsigned="true" nullable="false" comment="Category ID"/>
        
        <!-- Composite Primary Key -->
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
            <column name="category_id"/>
        </constraint>
        
        <!-- Foreign Keys -->
        <constraint xsi:type="foreign" referenceId="VENDOR_MODULE_ENTITY_CATEGORY_ENTITY_ID" 
                   table="vendor_module_entity_category" column="entity_id" 
                   referenceTable="vendor_module_entity" referenceColumn="entity_id" 
                   onDelete="CASCADE"/>
        
        <constraint xsi:type="foreign" referenceId="VENDOR_MODULE_ENTITY_CATEGORY_CATEGORY_ID" 
                   table="vendor_module_entity_category" column="category_id" 
                   referenceTable="catalog_category_entity" referenceColumn="entity_id" 
                   onDelete="CASCADE"/>
    </table>
</schema>
```

### Column Types and Attributes:

```xml
<!-- Integer Types -->
<column xsi:type="smallint" name="status" unsigned="true" nullable="false"/>
<column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true"/>
<column xsi:type="bigint" name="large_number" unsigned="true" nullable="true"/>

<!-- String Types -->
<column xsi:type="varchar" name="name" length="255" nullable="false"/>
<column xsi:type="text" name="description" nullable="true"/>
<column xsi:type="mediumtext" name="content" nullable="true"/>
<column xsi:type="longtext" name="large_content" nullable="true"/>

<!-- Decimal Types -->
<column xsi:type="decimal" name="price" scale="4" precision="12" unsigned="false" nullable="false" default="0"/>
<column xsi:type="float" name="weight" unsigned="true" nullable="true"/>
<column xsi:type="double" name="precise_value" unsigned="false" nullable="true"/>

<!-- Date/Time Types -->
<column xsi:type="date" name="event_date" nullable="true"/>
<column xsi:type="datetime" name="event_datetime" nullable="true"/>
<column xsi:type="timestamp" name="created_at" on_update="false" nullable="false" default="CURRENT_TIMESTAMP"/>

<!-- Binary Types -->
<column xsi:type="blob" name="binary_data" nullable="true"/>
<column xsi:type="mediumblob" name="medium_binary" nullable="true"/>

<!-- Boolean -->
<column xsi:type="boolean" name="is_active" nullable="false" default="true"/>

<!-- JSON (MySQL 5.7+) -->
<column xsi:type="json" name="metadata" nullable="true"/>
```

## Legacy InstallSchema & UpgradeSchema

### InstallSchema Example:

```php
<?php
namespace Vendor\Module\Setup;

use Magento\Framework\Setup\InstallSchemaInterface;
use Magento\Framework\Setup\ModuleContextInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\DB\Ddl\Table;

class InstallSchema implements InstallSchemaInterface
{
    public function install(SchemaSetupInterface $setup, ModuleContextInterface $context)
    {
        $installer = $setup;
        $installer->startSetup();

        // Create main entity table
        $table = $installer->getConnection()->newTable(
            $installer->getTable('vendor_module_entity')
        )->addColumn(
            'entity_id',
            Table::TYPE_INTEGER,
            null,
            ['identity' => true, 'nullable' => false, 'primary' => true, 'unsigned' => true],
            'Entity ID'
        )->addColumn(
            'name',
            Table::TYPE_TEXT,
            255,
            ['nullable' => false],
            'Entity Name'
        )->addColumn(
            'description',
            Table::TYPE_TEXT,
            '64k',
            ['nullable' => true],
            'Description'
        )->addColumn(
            'price',
            Table::TYPE_DECIMAL,
            '12,4',
            ['nullable' => false, 'default' => '0.0000'],
            'Price'
        )->addColumn(
            'status',
            Table::TYPE_SMALLINT,
            null,
            ['nullable' => false, 'default' => '1', 'unsigned' => true],
            'Status'
        )->addColumn(
            'created_at',
            Table::TYPE_TIMESTAMP,
            null,
            ['nullable' => false, 'default' => Table::TIMESTAMP_INIT],
            'Created At'
        )->addColumn(
            'updated_at',
            Table::TYPE_TIMESTAMP,
            null,
            ['nullable' => false, 'default' => Table::TIMESTAMP_INIT_UPDATE],
            'Updated At'
        )->addIndex(
            $installer->getIdxName('vendor_module_entity', ['name']),
            ['name']
        )->addIndex(
            $installer->getIdxName('vendor_module_entity', ['status']),
            ['status']
        )->setComment(
            'Vendor Module Entity Table'
        );

        $installer->getConnection()->createTable($table);
        $installer->endSetup();
    }
}
```

### UpgradeSchema Example:

```php
<?php
namespace Vendor\Module\Setup;

use Magento\Framework\Setup\UpgradeSchemaInterface;
use Magento\Framework\Setup\ModuleContextInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\DB\Ddl\Table;

class UpgradeSchema implements UpgradeSchemaInterface
{
    public function upgrade(SchemaSetupInterface $setup, ModuleContextInterface $context)
    {
        $setup->startSetup();

        if (version_compare($context->getVersion(), '1.0.1', '<')) {
            $this->upgradeToVersion101($setup);
        }

        if (version_compare($context->getVersion(), '1.0.2', '<')) {
            $this->upgradeToVersion102($setup);
        }

        $setup->endSetup();
    }

    private function upgradeToVersion101(SchemaSetupInterface $setup)
    {
        $connection = $setup->getConnection();
        $tableName = $setup->getTable('vendor_module_entity');

        // Add new column
        $connection->addColumn(
            $tableName,
            'sort_order',
            [
                'type' => Table::TYPE_INTEGER,
                'nullable' => false,
                'default' => 0,
                'comment' => 'Sort Order'
            ]
        );

        // Add index
        $connection->addIndex(
            $tableName,
            $setup->getIdxName('vendor_module_entity', ['sort_order']),
            ['sort_order']
        );
    }

    private function upgradeToVersion102(SchemaSetupInterface $setup)
    {
        $connection = $setup->getConnection();
        $tableName = $setup->getTable('vendor_module_entity');

        // Modify column
        $connection->modifyColumn(
            $tableName,
            'description',
            [
                'type' => Table::TYPE_TEXT,
                'length' => '2M',
                'comment' => 'Extended Description'
            ]
        );
    }
}
```

## Data Patches vs Schema Patches

### Schema Patch (Magento 2.3+):

```php
<?php
namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\Patch\SchemaPatchInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\DB\Ddl\Table;

class AddSortOrderColumn implements SchemaPatchInterface
{
    private SchemaSetupInterface $schemaSetup;

    public function __construct(SchemaSetupInterface $schemaSetup)
    {
        $this->schemaSetup = $schemaSetup;
    }

    public function apply()
    {
        $this->schemaSetup->startSetup();

        $connection = $this->schemaSetup->getConnection();
        $tableName = $this->schemaSetup->getTable('vendor_module_entity');

        if ($connection->isTableExists($tableName)) {
            if (!$connection->tableColumnExists($tableName, 'sort_order')) {
                $connection->addColumn(
                    $tableName,
                    'sort_order',
                    [
                        'type' => Table::TYPE_INTEGER,
                        'nullable' => false,
                        'default' => 0,
                        'comment' => 'Sort Order',
                        'after' => 'status'
                    ]
                );
            }

            // Add index if not exists
            $indexName = $connection->getIndexName($tableName, ['sort_order']);
            if (!$connection->isTableExists($indexName)) {
                $connection->addIndex($tableName, $indexName, ['sort_order']);
            }
        }

        $this->schemaSetup->endSetup();
    }

    public function getAliases(): array
    {
        return [];
    }

    public static function getDependencies(): array
    {
        return [];
    }
}
```

### Data Patch:

```php
<?php
namespace Vendor\Module\Setup\Patch\Data;

use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Vendor\Module\Model\EntityFactory;
use Vendor\Module\Api\EntityRepositoryInterface;

class CreateDefaultEntities implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;
    private EntityFactory $entityFactory;
    private EntityRepositoryInterface $entityRepository;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EntityFactory $entityFactory,
        EntityRepositoryInterface $entityRepository
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->entityFactory = $entityFactory;
        $this->entityRepository = $entityRepository;
    }

    public function apply()
    {
        $this->moduleDataSetup->startSetup();

        // Create default entities
        $defaultEntities = [
            ['name' => 'Default Entity 1', 'status' => 1, 'sort_order' => 10],
            ['name' => 'Default Entity 2', 'status' => 1, 'sort_order' => 20],
        ];

        foreach ($defaultEntities as $entityData) {
            $entity = $this->entityFactory->create();
            $entity->setData($entityData);
            $this->entityRepository->save($entity);
        }

        $this->moduleDataSetup->endSetup();
    }

    public function getAliases(): array
    {
        return [];
    }

    public static function getDependencies(): array
    {
        return [
            \Vendor\Module\Setup\Patch\Schema\AddSortOrderColumn::class
        ];
    }
}
```

## EAV System Deep Dive

### EAV Table Structure:

```xml
<!-- Entity Table -->
<table name="vendor_module_entity" resource="default" engine="innodb">
    <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true"/>
    <column xsi:type="smallint" name="entity_type_id" unsigned="true" nullable="false"/>
    <column xsi:type="smallint" name="attribute_set_id" unsigned="true" nullable="false"/>
    <column xsi:type="datetime" name="created_at" nullable="false"/>
    <column xsi:type="datetime" name="updated_at" nullable="false"/>
    <!-- Other entity-specific columns -->
</table>

<!-- Attribute Value Tables (one per data type) -->
<table name="vendor_module_entity_varchar" resource="default" engine="innodb">
    <column xsi:type="int" name="value_id" unsigned="true" nullable="false" identity="true"/>
    <column xsi:type="int" name="entity_type_id" unsigned="true" nullable="false"/>
    <column xsi:type="smallint" name="attribute_id" unsigned="true" nullable="false"/>
    <column xsi:type="smallint" name="store_id" unsigned="true" nullable="false"/>
    <column xsi:type="int" name="entity_id" unsigned="true" nullable="false"/>
    <column xsi:type="varchar" name="value" length="255" nullable="true"/>
    
    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="value_id"/>
    </constraint>
    
    <constraint xsi:type="unique" referenceId="VENDOR_MODULE_ENTITY_VARCHAR_ENTITY_ID_ATTRIBUTE_ID_STORE_ID">
        <column name="entity_id"/>
        <column name="attribute_id"/>
        <column name="store_id"/>
    </constraint>
</table>

<table name="vendor_module_entity_int" resource="default" engine="innodb">
    <!-- Similar structure with int value column -->
    <column xsi:type="int" name="value" nullable="true"/>
</table>

<table name="vendor_module_entity_decimal" resource="default" engine="innodb">
    <!-- Similar structure with decimal value column -->
    <column xsi:type="decimal" name="value" scale="4" precision="12" nullable="true"/>
</table>

<table name="vendor_module_entity_datetime" resource="default" engine="innodb">
    <!-- Similar structure with datetime value column -->
    <column xsi:type="datetime" name="value" nullable="true"/>
</table>

<table name="vendor_module_entity_text" resource="default" engine="innodb">
    <!-- Similar structure with text value column -->
    <column xsi:type="text" name="value" nullable="true"/>
</table>
```

### EAV Model Implementation:

```php
<?php
namespace Vendor\Module\Model;

use Magento\Eav\Model\Entity\AbstractEntity;
use Magento\Eav\Model\Entity\Context;
use Magento\Store\Model\StoreManagerInterface;

class Entity extends AbstractEntity
{
    protected function _construct()
    {
        $this->setType('vendor_module_entity');
        $this->setConnection('vendor_module_entity_read', 'vendor_module_entity_write');
    }

    public function getDefaultAttributes()
    {
        return [
            'entity_type_id',
            'attribute_set_id',
            'created_at',
            'updated_at'
        ];
    }
}
```

### EAV Entity Model:

```php
<?php
namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractModel;
use Vendor\Module\Api\Data\EntityInterface;

class Entity extends AbstractModel implements EntityInterface
{
    protected function _construct()
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Entity::class);
    }

    // Implement interface methods...
}
```

## Table Types and Relationships

### 1. Entity Tables (Main Data):

```xml
<!-- Core entity data -->
<table name="vendor_module_product" resource="default" engine="innodb">
    <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true"/>
    <column xsi:type="varchar" name="sku" length="64" nullable="false"/>
    <column xsi:type="varchar" name="name" length="255" nullable="false"/>
    <column xsi:type="decimal" name="price" scale="4" precision="12" nullable="false"/>
    
    <constraint xsi:type="unique" referenceId="VENDOR_MODULE_PRODUCT_SKU">
        <column name="sku"/>
    </constraint>
</table>
```

### 2. Relationship Tables (Many-to-Many):

```xml
<!-- Product-Category relationship -->
<table name="vendor_module_product_category" resource="default" engine="innodb">
    <column xsi:type="int" name="product_id" unsigned="true" nullable="false"/>
    <column xsi:type="int" name="category_id" unsigned="true" nullable="false"/>
    <column xsi:type="int" name="position" unsigned="true" nullable="false" default="0"/>
    
    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="product_id"/>
        <column name="category_id"/>
    </constraint>
</table>
```

### 3. Store-Scoped Tables:

```xml
<!-- Store-specific data -->
<table name="vendor_module_product_store" resource="default" engine="innodb">
    <column xsi:type="int" name="product_id" unsigned="true" nullable="false"/>
    <column xsi:type="smallint" name="store_id" unsigned="true" nullable="false"/>
    <column xsi:type="varchar" name="name" length="255" nullable="true"/>
    <column xsi:type="text" name="description" nullable="true"/>
    
    <constraint xsi:type="primary" referenceId="PRIMARY">
        <column name="product_id"/>
        <column name="store_id"/>
    </constraint>
</table>
```

### 4. Log/History Tables:

```xml
<!-- Audit trail -->
<table name="vendor_module_product_history" resource="default" engine="innodb">
    <column xsi:type="int" name="history_id" unsigned="true" nullable="false" identity="true"/>
    <column xsi:type="int" name="product_id" unsigned="true" nullable="false"/>
    <column xsi:type="varchar" name="action" length="50" nullable="false"/>
    <column xsi:type="json" name="old_values" nullable="true"/>
    <column xsi:type="json" name="new_values" nullable="true"/>
    <column xsi:type="int" name="user_id" unsigned="true" nullable="true"/>
    <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP"/>
    
    <index referenceId="VENDOR_MODULE_PRODUCT_HISTORY_PRODUCT_ID_CREATED_AT">
        <column name="product_id"/>
        <column name="created_at"/>
    </index>
</table>
```

### 5. Configuration Tables:

```xml
<!-- Module configuration -->
<table name="vendor_module_config" resource="default" engine="innodb">
    <column xsi:type="int" name="config_id" unsigned="true" nullable="false" identity="true"/>
    <column xsi:type="varchar" name="path" length="255" nullable="false"/>
    <column xsi:type="text" name="value" nullable="true"/>
    <column xsi:type="smallint" name="scope_id" unsigned="true" nullable="false" default="0"/>
    <column xsi:type="varchar" name="scope" length="8" nullable="false" default="default"/>
    
    <constraint xsi:type="unique" referenceId="VENDOR_MODULE_CONFIG_PATH_SCOPE_SCOPE_ID">
        <column name="path"/>
        <column name="scope"/>
        <column name="scope_id"/>
    </constraint>
</table>
```

## Indexing Strategy

### Primary Indexes:

```xml
<!-- Single Column Index -->
<index referenceId="VENDOR_MODULE_ENTITY_STATUS" indexType="btree">
    <column name="status"/>
</index>

<!-- Composite Index -->
<index referenceId="VENDOR_MODULE_ENTITY_STATUS_STORE_CREATED" indexType="btree">
    <column name="status"/>
    <column name="store_id"/>
    <column name="created_at"/>
</index>

<!-- Covering Index (includes additional columns) -->
<index referenceId="VENDOR_MODULE_ENTITY_SEARCH_COVERING" indexType="btree">
    <column name="status"/>
    <column name="store_id"/>
    <column name="name"/>        <!-- Additional coverage -->
    <column name="created_at"/>  <!-- Additional coverage -->
</index>
```

### Full-Text Search Indexes:

```xml
<!-- Full-text search capability -->
<index referenceId="VENDOR_MODULE_ENTITY_FULLTEXT" indexType="fulltext">
    <column name="name"/>
    <column name="description"/>
</index>
```

### Index Best Practices:

1. **Query-Driven Design**: Create indexes based on actual query patterns
2. **Composite Key Order**: Most selective column first
3. **Covering Indexes**: Include frequently accessed columns
4. **Avoid Over-Indexing**: Each index has maintenance overhead

## Foreign Keys and Constraints

### Foreign Key Definitions:

```xml
<!-- Standard Foreign Key -->
<constraint xsi:type="foreign" 
           referenceId="VENDOR_MODULE_ENTITY_STORE_ID_STORE_STORE_ID" 
           table="vendor_module_entity" 
           column="store_id" 
           referenceTable="store" 
           referenceColumn="store_id" 
           onDelete="CASCADE"/>

<!-- Foreign Key with RESTRICT -->
<constraint xsi:type="foreign" 
           referenceId="VENDOR_MODULE_PRODUCT_CATEGORY_ID_CATALOG_CATEGORY_ENTITY_ID" 
           table="vendor_module_product" 
           column="category_id" 
           referenceTable="catalog_category_entity" 
           referenceColumn="entity_id" 
           onDelete="RESTRICT"/>

<!-- Foreign Key with SET NULL -->
<constraint xsi:type="foreign" 
           referenceId="VENDOR_MODULE_ENTITY_PARENT_ID_ENTITY_ID" 
           table="vendor_module_entity" 
           column="parent_id" 
           referenceTable="vendor_module_entity" 
           referenceColumn="entity_id" 
           onDelete="SET NULL"/>
```

### Check Constraints (MySQL 8.0+):

```xml
<!-- Value validation -->
<constraint xsi:type="check" referenceId="VENDOR_MODULE_ENTITY_STATUS_CHECK">
    <column name="status"/>
    <expression>status IN (0, 1, 2)</expression>
</constraint>

<constraint xsi:type="check" referenceId="VENDOR_MODULE_ENTITY_PRICE_CHECK">
    <column name="price"/>
    <expression>price >= 0</expression>
</constraint>
```

### Unique Constraints:

```xml
<!-- Single column unique -->
<constraint xsi:type="unique" referenceId="VENDOR_MODULE_ENTITY_SKU">
    <column name="sku"/>
</constraint>

<!-- Composite unique -->
<constraint xsi:type="unique" referenceId="VENDOR_MODULE_ENTITY_NAME_STORE_ID">
    <column name="name"/>
    <column name="store_id"/>
</constraint>
```

## Performance Considerations

### 1. Table Engine Selection:

```xml
<!-- InnoDB for transactional data -->
<table name="vendor_module_orders" resource="default" engine="innodb">
    <!-- ACID compliance, row-level locking -->
</table>

<!-- MyISAM for read-heavy data (legacy) -->
<table name="vendor_module_logs" resource="default" engine="myisam">
    <!-- Table-level locking, fast reads -->
</table>

<!-- Memory for temporary data -->
<table name="vendor_module_session" resource="default" engine="memory">
    <!-- In-memory storage -->
</table>
```

### 2. Partitioning Strategy:

```sql
-- Range partitioning by date
CREATE TABLE vendor_module_logs (
    log_id INT AUTO_INCREMENT,
    message TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (log_id, created_at)
)
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION pmax VALUES LESS THAN MAXVALUE
);

-- Hash partitioning for distribution
CREATE TABLE vendor_module_user_data (
    user_id INT,
    data JSON,
    PRIMARY KEY (user_id)
)
PARTITION BY HASH(user_id)
PARTITIONS 16;
```

### 3. Query Optimization:

```xml
<!-- Optimized indexes for common queries -->

<!-- For: SELECT * FROM entity WHERE status = 1 AND store_id = 2 ORDER BY created_at DESC -->
<index referenceId="VENDOR_MODULE_ENTITY_STATUS_STORE_CREATED">
    <column name="status"/>
    <column name="store_id"/>
    <column name="created_at"/>
</index>

<!-- For: SELECT name, price FROM entity WHERE category_id = ? AND status = 1 -->
<index referenceId="VENDOR_MODULE_ENTITY_CATEGORY_STATUS_COVERING">
    <column name="category_id"/>
    <column name="status"/>
    <column name="name"/>   <!-- Covering -->
    <column name="price"/>  <!-- Covering -->
</index>
```

### 4. Data Type Optimization:

```xml
<!-- Use appropriate sizes -->
<column xsi:type="tinyint" name="status" unsigned="true"/>           <!-- 0-255 -->
<column xsi:type="smallint" name="store_id" unsigned="true"/>        <!-- 0-65535 -->
<column xsi:type="mediumint" name="category_id" unsigned="true"/>    <!-- 0-16M -->
<column xsi:type="int" name="entity_id" unsigned="true"/>            <!-- 0-4B -->
<column xsi:type="bigint" name="large_id" unsigned="true"/>          <!-- 0-18Q -->

<!-- Fixed vs Variable length -->
<column xsi:type="char" name="country_code" length="2"/>             <!-- Fixed: US, UK -->
<column xsi:type="varchar" name="name" length="255"/>                <!-- Variable -->

<!-- Precision for decimals -->
<column xsi:type="decimal" name="price" scale="4" precision="12"/>   <!-- 99999999.9999 -->
```

## Migration Strategies

### From Legacy Schema to Declarative:

#### Step 1: Generate Initial db_schema.xml

```bash
# Generate schema from existing database
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module
```

#### Step 2: Create db_schema.xml

```xml
<!-- Generated from existing tables -->
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <!-- Tables based on current database state -->
</schema>
```

#### Step 3: Validate Schema

```bash
# Dry run to see what changes would be made
bin/magento setup:db-declaration:generate-patch --dry-run

# Apply changes
bin/magento setup:upgrade
```

### Incremental Migration Approach:

```php
<?php
namespace Vendor\Module\Setup\Patch\Schema;

use Magento\Framework\Setup\Patch\SchemaPatchInterface;

class MigrateToDeclarativeSchema implements SchemaPatchInterface
{
    public function apply()
    {
        // Step 1: Create new tables with declarative schema
        // Step 2: Migrate data from old tables
        // Step 3: Drop old tables (in subsequent patch)
    }

    public static function getDependencies(): array
    {
        return [
            // Ensure all old schema upgrades are applied first
        ];
    }
}
```

## Best Practices

### 1. Naming Conventions:

```xml
<!-- Table Naming -->
<table name="vendor_module_entity">                    <!-- module_entity -->
<table name="vendor_module_entity_store">              <!-- entity_store -->
<table name="vendor_module_entity_category">           <!-- entity_category (relationship) -->

<!-- Column Naming -->
<column name="entity_id"/>                             <!-- Primary key -->
<column name="parent_entity_id"/>                      <!-- Foreign key -->
<column name="is_active"/>                             <!-- Boolean prefix -->
<column name="created_at"/>                            <!-- Timestamp suffix -->

<!-- Index Naming -->
<index referenceId="VENDOR_MODULE_ENTITY_STATUS"/>                           <!-- Simple -->
<index referenceId="VENDOR_MODULE_ENTITY_STATUS_STORE_CREATED"/>             <!-- Composite -->

<!-- Constraint Naming -->
<constraint referenceId="VENDOR_MODULE_ENTITY_STORE_ID_STORE_STORE_ID"/>     <!-- FK -->
<constraint referenceId="VENDOR_MODULE_ENTITY_NAME_STORE_UNIQUE"/>           <!-- Unique -->
```

### 2. Schema Design Principles:

```xml
<!-- 1. Use appropriate data types -->
<column xsi:type="smallint" name="status" unsigned="true" nullable="false" default="1"/>

<!-- 2. Always specify nullability -->
<column xsi:type="varchar" name="name" length="255" nullable="false"/>
<column xsi:type="text" name="description" nullable="true"/>

<!-- 3. Use comments for clarity -->
<table name="vendor_module_entity" comment="Main Entity Table">
    <column name="entity_id" comment="Unique Entity Identifier"/>
</table>

<!-- 4. Include timestamps -->
<column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP"/>
<column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true"/>

<!-- 5. Plan for internationalization -->
<column xsi:type="smallint" name="store_id" unsigned="true" nullable="false" comment="Store View ID"/>
```

### 3. Performance Optimization:

```xml
<!-- Index frequently queried columns -->
<index referenceId="VENDOR_MODULE_ENTITY_STATUS_CREATED">
    <column name="status"/>
    <column name="created_at"/>
</index>

<!-- Use composite indexes wisely -->
<index referenceId="VENDOR_MODULE_PRODUCT_CATEGORY_POSITION">
    <column name="category_id"/>    <!-- Filter first -->
    <column name="position"/>       <!-- Sort second -->
</index>

<!-- Add covering indexes for read-heavy queries -->
<index referenceId="VENDOR_MODULE_ENTITY_LIST_COVERING">
    <column name="status"/>
    <column name="store_id"/>
    <column name="name"/>           <!-- Include in index -->
    <column name="created_at"/>     <!-- Include in index -->
</index>
```

### 4. Data Integrity:

```xml
<!-- Always use foreign keys -->
<constraint xsi:type="foreign" 
           referenceId="VENDOR_MODULE_ENTITY_STORE_ID" 
           table="vendor_module_entity" 
           column="store_id" 
           referenceTable="store" 
           referenceColumn="store_id" 
           onDelete="CASCADE"/>

<!-- Ensure data consistency -->
<constraint xsi:type="unique" referenceId="VENDOR_MODULE_ENTITY_SKU_STORE">
    <column name="sku"/>
    <column name="store_id"/>
</constraint>

<!-- Use check constraints when available -->
<constraint xsi:type="check" referenceId="VENDOR_MODULE_ENTITY_STATUS_VALID">
    <expression>status IN (0, 1, 2)</expression>
</constraint>
```

## Real-World Examples

### Example 1: E-commerce Product Catalog

```xml
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <!-- Main Product Table -->
    <table name="custom_catalog_product" resource="default" engine="innodb" comment="Custom Product Catalog">
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="Product ID"/>
        <column xsi:type="varchar" name="sku" length="64" nullable="false" comment="Product SKU"/>
        <column xsi:type="varchar" name="name" length="255" nullable="false" comment="Product Name"/>
        <column xsi:type="text" name="description" nullable="true" comment="Product Description"/>
        <column xsi:type="decimal" name="price" scale="4" precision="12" unsigned="false" nullable="false" default="0" comment="Base Price"/>
        <column xsi:type="decimal" name="special_price" scale="4" precision="12" unsigned="false" nullable="true" comment="Special Price"/>
        <column xsi:type="date" name="special_from_date" nullable="true" comment="Special Price From"/>
        <column xsi:type="date" name="special_to_date" nullable="true" comment="Special Price To"/>
        <column xsi:type="smallint" name="status" unsigned="true" nullable="false" default="1" comment="Product Status"/>
        <column xsi:type="smallint" name="visibility" unsigned="true" nullable="false" default="4" comment="Product Visibility"/>
        <column xsi:type="varchar" name="type_id" length="32" nullable="false" default="simple" comment="Product Type"/>
        <column xsi:type="int" name="attribute_set_id" unsigned="true" nullable="false" comment="Attribute Set ID"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true" comment="Updated At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="CUSTOM_CATALOG_PRODUCT_SKU">
            <column name="sku"/>
        </constraint>

        <index referenceId="CUSTOM_CATALOG_PRODUCT_STATUS_VISIBILITY">
            <column name="status"/>
            <column name="visibility"/>
        </index>

        <index referenceId="CUSTOM_CATALOG_PRODUCT_TYPE_STATUS">
            <column name="type_id"/>
            <column name="status"/>
        </index>

        <index referenceId="CUSTOM_CATALOG_PRODUCT_PRICE_RANGE">
            <column name="price"/>
        </index>
    </table>

    <!-- Product-Category Relationship -->
    <table name="custom_catalog_product_category" resource="default" engine="innodb" comment="Product Category Relations">
        <column xsi:type="int" name="product_id" unsigned="true" nullable="false" comment="Product ID"/>
        <column xsi:type="int" name="category_id" unsigned="true" nullable="false" comment="Category ID"/>
        <column xsi:type="int" name="position" unsigned="true" nullable="false" default="0" comment="Position"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="product_id"/>
            <column name="category_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_CATALOG_PRODUCT_CATEGORY_PRODUCT_ID" 
                   table="custom_catalog_product_category" 
                   column="product_id" 
                   referenceTable="custom_catalog_product" 
                   referenceColumn="entity_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_CATALOG_PRODUCT_CATEGORY_CATEGORY_POSITION">
            <column name="category_id"/>
            <column name="position"/>
        </index>
    </table>

    <!-- Inventory Table -->
    <table name="custom_catalog_product_inventory" resource="default" engine="innodb" comment="Product Inventory">
        <column xsi:type="int" name="product_id" unsigned="true" nullable="false" comment="Product ID"/>
        <column xsi:type="smallint" name="website_id" unsigned="true" nullable="false" comment="Website ID"/>
        <column xsi:type="decimal" name="qty" scale="4" precision="12" nullable="false" default="0" comment="Quantity"/>
        <column xsi:type="smallint" name="is_in_stock" unsigned="true" nullable="false" default="0" comment="Stock Status"/>
        <column xsi:type="decimal" name="min_qty" scale="4" precision="12" nullable="false" default="0" comment="Minimum Quantity"/>
        <column xsi:type="smallint" name="manage_stock" unsigned="true" nullable="false" default="1" comment="Manage Stock"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="product_id"/>
            <column name="website_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_CATALOG_PRODUCT_INVENTORY_PRODUCT_ID" 
                   table="custom_catalog_product_inventory" 
                   column="product_id" 
                   referenceTable="custom_catalog_product" 
                   referenceColumn="entity_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_CATALOG_PRODUCT_INVENTORY_STOCK_STATUS">
            <column name="is_in_stock"/>
            <column name="manage_stock"/>
        </index>
    </table>

    <!-- Product Images -->
    <table name="custom_catalog_product_gallery" resource="default" engine="innodb" comment="Product Image Gallery">
        <column xsi:type="int" name="value_id" unsigned="true" nullable="false" identity="true" comment="Value ID"/>
        <column xsi:type="int" name="product_id" unsigned="true" nullable="false" comment="Product ID"/>
        <column xsi:type="varchar" name="file" length="255" nullable="false" comment="File Path"/>
        <column xsi:type="varchar" name="label" length="255" nullable="true" comment="Image Label"/>
        <column xsi:type="int" name="position" unsigned="true" nullable="false" default="0" comment="Position"/>
        <column xsi:type="smallint" name="disabled" unsigned="true" nullable="false" default="0" comment="Is Disabled"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="value_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_CATALOG_PRODUCT_GALLERY_PRODUCT_ID" 
                   table="custom_catalog_product_gallery" 
                   column="product_id" 
                   referenceTable="custom_catalog_product" 
                   referenceColumn="entity_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_CATALOG_PRODUCT_GALLERY_PRODUCT_POSITION">
            <column name="product_id"/>
            <column name="position"/>
        </index>
    </table>
</schema>
```

### Example 2: Order Management System

```xml
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <!-- Orders Table -->
    <table name="custom_sales_order" resource="default" engine="innodb" comment="Sales Orders">
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="Order ID"/>
        <column xsi:type="varchar" name="increment_id" length="50" nullable="false" comment="Order Number"/>
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="true" comment="Customer ID"/>
        <column xsi:type="varchar" name="customer_email" length="128" nullable="true" comment="Customer Email"/>
        <column xsi:type="smallint" name="store_id" unsigned="true" nullable="false" comment="Store ID"/>
        <column xsi:type="varchar" name="status" length="32" nullable="false" comment="Order Status"/>
        <column xsi:type="varchar" name="state" length="32" nullable="false" comment="Order State"/>
        <column xsi:type="decimal" name="grand_total" scale="4" precision="20" nullable="false" comment="Grand Total"/>
        <column xsi:type="decimal" name="subtotal" scale="4" precision="20" nullable="false" comment="Subtotal"/>
        <column xsi:type="decimal" name="tax_amount" scale="4" precision="20" nullable="false" default="0" comment="Tax Amount"/>
        <column xsi:type="decimal" name="shipping_amount" scale="4" precision="20" nullable="false" default="0" comment="Shipping Amount"/>
        <column xsi:type="decimal" name="discount_amount" scale="4" precision="20" nullable="false" default="0" comment="Discount Amount"/>
        <column xsi:type="varchar" name="order_currency_code" length="3" nullable="false" comment="Order Currency"/>
        <column xsi:type="text" name="billing_address" nullable="true" comment="Billing Address JSON"/>
        <column xsi:type="text" name="shipping_address" nullable="true" comment="Shipping Address JSON"/>
        <column xsi:type="varchar" name="shipping_method" length="120" nullable="true" comment="Shipping Method"/>
        <column xsi:type="varchar" name="payment_method" length="120" nullable="true" comment="Payment Method"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true" comment="Updated At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="CUSTOM_SALES_ORDER_INCREMENT_ID">
            <column name="increment_id"/>
        </constraint>

        <index referenceId="CUSTOM_SALES_ORDER_CUSTOMER_ID">
            <column name="customer_id"/>
        </index>

        <index referenceId="CUSTOM_SALES_ORDER_STATUS_CREATED">
            <column name="status"/>
            <column name="created_at"/>
        </index>

        <index referenceId="CUSTOM_SALES_ORDER_STORE_CREATED">
            <column name="store_id"/>
            <column name="created_at"/>
        </index>
    </table>

    <!-- Order Items -->
    <table name="custom_sales_order_item" resource="default" engine="innodb" comment="Sales Order Items">
        <column xsi:type="int" name="item_id" unsigned="true" nullable="false" identity="true" comment="Item ID"/>
        <column xsi:type="int" name="order_id" unsigned="true" nullable="false" comment="Order ID"/>
        <column xsi:type="int" name="product_id" unsigned="true" nullable="false" comment="Product ID"/>
        <column xsi:type="varchar" name="product_type" length="255" nullable="false" comment="Product Type"/>
        <column xsi:type="varchar" name="sku" length="255" nullable="false" comment="Product SKU"/>
        <column xsi:type="varchar" name="name" length="255" nullable="false" comment="Product Name"/>
        <column xsi:type="decimal" name="price" scale="4" precision="12" nullable="false" comment="Product Price"/>
        <column xsi:type="decimal" name="qty_ordered" scale="4" precision="12" nullable="false" comment="Quantity Ordered"/>
        <column xsi:type="decimal" name="qty_shipped" scale="4" precision="12" nullable="false" default="0" comment="Quantity Shipped"/>
        <column xsi:type="decimal" name="qty_invoiced" scale="4" precision="12" nullable="false" default="0" comment="Quantity Invoiced"/>
        <column xsi:type="decimal" name="qty_canceled" scale="4" precision="12" nullable="false" default="0" comment="Quantity Canceled"/>
        <column xsi:type="decimal" name="row_total" scale="4" precision="12" nullable="false" comment="Row Total"/>
        <column xsi:type="text" name="product_options" nullable="true" comment="Product Options JSON"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="item_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_SALES_ORDER_ITEM_ORDER_ID" 
                   table="custom_sales_order_item" 
                   column="order_id" 
                   referenceTable="custom_sales_order" 
                   referenceColumn="entity_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_SALES_ORDER_ITEM_ORDER_ID">
            <column name="order_id"/>
        </index>

        <index referenceId="CUSTOM_SALES_ORDER_ITEM_PRODUCT_ID">
            <column name="product_id"/>
        </index>
    </table>

    <!-- Order Status History -->
    <table name="custom_sales_order_status_history" resource="default" engine="innodb" comment="Order Status History">
        <column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true" comment="History ID"/>
        <column xsi:type="int" name="order_id" unsigned="true" nullable="false" comment="Order ID"/>
        <column xsi:type="varchar" name="status" length="32" nullable="false" comment="Order Status"/>
        <column xsi:type="text" name="comment" nullable="true" comment="Status Comment"/>
        <column xsi:type="smallint" name="is_customer_notified" unsigned="true" nullable="false" default="0" comment="Customer Notified"/>
        <column xsi:type="smallint" name="is_visible_on_front" unsigned="true" nullable="false" default="0" comment="Visible on Frontend"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_SALES_ORDER_STATUS_HISTORY_ORDER_ID" 
                   table="custom_sales_order_status_history" 
                   column="order_id" 
                   referenceTable="custom_sales_order" 
                   referenceColumn="entity_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_SALES_ORDER_STATUS_HISTORY_ORDER_CREATED">
            <column name="order_id"/>
            <column name="created_at"/>
        </index>
    </table>
</schema>
```

### Example 3: Customer Loyalty System

```xml
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">

    <!-- Loyalty Accounts -->
    <table name="custom_loyalty_account" resource="default" engine="innodb" comment="Customer Loyalty Accounts">
        <column xsi:type="int" name="account_id" unsigned="true" nullable="false" identity="true" comment="Account ID"/>
        <column xsi:type="int" name="customer_id" unsigned="true" nullable="false" comment="Customer ID"/>
        <column xsi:type="int" name="points_balance" nullable="false" default="0" comment="Current Points Balance"/>
        <column xsi:type="int" name="lifetime_points" unsigned="true" nullable="false" default="0" comment="Lifetime Points Earned"/>
        <column xsi:type="varchar" name="tier_level" length="50" nullable="false" default="bronze" comment="Loyalty Tier"/>
        <column xsi:type="date" name="tier_expires_at" nullable="true" comment="Tier Expiration Date"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true" comment="Updated At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="account_id"/>
        </constraint>

        <constraint xsi:type="unique" referenceId="CUSTOM_LOYALTY_ACCOUNT_CUSTOMER_ID">
            <column name="customer_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_LOYALTY_ACCOUNT_CUSTOMER_ID_CUSTOMER_ENTITY_ID" 
                   table="custom_loyalty_account" 
                   column="customer_id" 
                   referenceTable="customer_entity" 
                   referenceColumn="entity_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_LOYALTY_ACCOUNT_TIER_LEVEL">
            <column name="tier_level"/>
        </index>

        <index referenceId="CUSTOM_LOYALTY_ACCOUNT_POINTS_BALANCE">
            <column name="points_balance"/>
        </index>
    </table>

    <!-- Point Transactions -->
    <table name="custom_loyalty_transaction" resource="default" engine="innodb" comment="Loyalty Point Transactions">
        <column xsi:type="int" name="transaction_id" unsigned="true" nullable="false" identity="true" comment="Transaction ID"/>
        <column xsi:type="int" name="account_id" unsigned="true" nullable="false" comment="Account ID"/>
        <column xsi:type="int" name="points" nullable="false" comment="Points Amount (positive = earned, negative = redeemed)"/>
        <column xsi:type="varchar" name="type" length="50" nullable="false" comment="Transaction Type"/>
        <column xsi:type="varchar" name="reference_type" length="50" nullable="true" comment="Reference Type (order, review, etc.)"/>
        <column xsi:type="int" name="reference_id" unsigned="true" nullable="true" comment="Reference ID"/>
        <column xsi:type="text" name="description" nullable="true" comment="Transaction Description"/>
        <column xsi:type="json" name="metadata" nullable="true" comment="Additional Metadata"/>
        <column xsi:type="date" name="expires_at" nullable="true" comment="Points Expiration Date"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="transaction_id"/>
        </constraint>

        <constraint xsi:type="foreign" 
                   referenceId="CUSTOM_LOYALTY_TRANSACTION_ACCOUNT_ID" 
                   table="custom_loyalty_transaction" 
                   column="account_id" 
                   referenceTable="custom_loyalty_account" 
                   referenceColumn="account_id" 
                   onDelete="CASCADE"/>

        <index referenceId="CUSTOM_LOYALTY_TRANSACTION_ACCOUNT_CREATED">
            <column name="account_id"/>
            <column name="created_at"/>
        </index>

        <index referenceId="CUSTOM_LOYALTY_TRANSACTION_TYPE_CREATED">
            <column name="type"/>
            <column name="created_at"/>
        </index>

        <index referenceId="CUSTOM_LOYALTY_TRANSACTION_REFERENCE">
            <column name="reference_type"/>
            <column name="reference_id"/>
        </index>

        <index referenceId="CUSTOM_LOYALTY_TRANSACTION_EXPIRES">
            <column name="expires_at"/>
        </index>
    </table>

    <!-- Loyalty Rules -->
    <table name="custom_loyalty_rule" resource="default" engine="innodb" comment="Points Earning Rules">
        <column xsi:type="int" name="rule_id" unsigned="true" nullable="false" identity="true" comment="Rule ID"/>
        <column xsi:type="varchar" name="name" length="255" nullable="false" comment="Rule Name"/>
        <column xsi:type="text" name="description" nullable="true" comment="Rule Description"/>
        <column xsi:type="varchar" name="event_type" length="100" nullable="false" comment="Event Type"/>
        <column xsi:type="int" name="points_amount" nullable="false" comment="Points to Award"/>
        <column xsi:type="varchar" name="calculation_type" length="50" nullable="false" default="fixed" comment="Calculation Type"/>
        <column xsi:type="decimal" name="calculation_rate" scale="4" precision="8" nullable="true" comment="Calculation Rate"/>
        <column xsi:type="text" name="conditions" nullable="true" comment="Rule Conditions JSON"/>
        <column xsi:type="int" name="usage_limit" unsigned="true" nullable="true" comment="Usage Limit per Customer"/>
        <column xsi:type="smallint" name="is_active" unsigned="true" nullable="false" default="1" comment="Is Active"/>
        <column xsi:type="date" name="from_date" nullable="true" comment="Valid From Date"/>
        <column xsi:type="date" name="to_date" nullable="true" comment="Valid To Date"/>
        <column xsi:type="int" name="sort_order" nullable="false" default="0" comment="Sort Order"/>
        <column xsi:type="timestamp" name="created_at" nullable="false" default="CURRENT_TIMESTAMP" comment="Created At"/>
        <column xsi:type="timestamp" name="updated_at" nullable="false" default="CURRENT_TIMESTAMP" on_update="true" comment="Updated At"/>

        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="rule_id"/>
        </constraint>

        <index referenceId="CUSTOM_LOYALTY_RULE_ACTIVE_DATES">
            <column name="is_active"/>
            <column name="from_date"/>
            <column name="to_date"/>
        </index>

        <index referenceId="CUSTOM_LOYALTY_RULE_EVENT_TYPE">
            <column name="event_type"/>
            <column name="is_active"/>
        </index>

        <index referenceId="CUSTOM_LOYALTY_RULE_SORT_ORDER">
            <column name="sort_order"/>
        </index>
    </table>
</schema>
```

## Troubleshooting

### Common Issues and Solutions:

#### 1. Schema Validation Errors

```bash
# Error: Invalid schema declaration
bin/magento setup:db-declaration:generate-patch --dry-run

# Common causes:
# - Missing referenceId attributes
# - Invalid column types
# - Circular foreign key dependencies
# - Missing required attributes
```

**Solution**: Validate schema syntax:
```xml
<!-- Correct -->
<column xsi:type="int" name="entity_id" unsigned="true" nullable="false" identity="true"/>

<!-- Incorrect -->
<column type="int" name="entity_id" unsigned="true" nullable="false" identity="true"/>
```

#### 2. Foreign Key Constraint Failures

```sql
-- Error: Cannot add foreign key constraint
-- Common causes:
-- - Referenced table doesn't exist
-- - Column types don't match
-- - Existing data violates constraint
```

**Solution**: Check data integrity:
```sql
-- Find orphaned records
SELECT t1.* FROM child_table t1 
LEFT JOIN parent_table t2 ON t1.parent_id = t2.id 
WHERE t2.id IS NULL;

-- Clean up before adding constraint
DELETE FROM child_table WHERE parent_id NOT IN (SELECT id FROM parent_table);
```

#### 3. Index Creation Failures

```sql
-- Error: Duplicate key name
-- Cause: Index already exists with same name
```

**Solution**: Check existing indexes:
```sql
SHOW INDEX FROM table_name;

-- Drop conflicting index if safe
DROP INDEX index_name ON table_name;
```

#### 4. Column Type Mismatches

```bash
# Error: Data truncation or type conversion issues
```

**Solution**: Migrate data safely:
```php
// In a data patch
$connection = $setup->getConnection();
$tableName = $setup->getTable('table_name');

// Add temporary column
$connection->addColumn($tableName, 'temp_column', [
    'type' => Table::TYPE_TEXT,
    'nullable' => true
]);

// Migrate data
$connection->update($tableName, ['temp_column' => new \Zend_Db_Expr('old_column')]);

// Drop old column
$connection->dropColumn($tableName, 'old_column');

// Rename temp column
$connection->changeColumn($tableName, 'temp_column', 'old_column', [
    'type' => Table::TYPE_TEXT
]);
```

#### 5. Performance Issues After Schema Changes

```sql
-- Check query performance
EXPLAIN SELECT * FROM table_name WHERE column = 'value';

-- Analyze table statistics
ANALYZE TABLE table_name;

-- Rebuild indexes if needed
OPTIMIZE TABLE table_name;
```

### Debugging Tools:

```bash
# Generate whitelist for existing schema
bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_Module

# Preview schema changes
bin/magento setup:db-declaration:generate-patch --dry-run

# Validate current schema
bin/magento setup:db-schema:upgrade --dry-run

# Check schema differences
bin/magento setup:db-schema:upgrade --safe-mode=1
```

---

## Conclusion

Mastering Magento 2's database schema system is crucial for building robust, scalable e-commerce applications. Key takeaways:

### Essential Principles:

1. **Use Declarative Schema** for new modules (Magento 2.3+)
2. **Plan for Performance** with proper indexing strategy
3. **Maintain Data Integrity** with foreign keys and constraints
4. **Follow Naming Conventions** for consistency
5. **Design for Scale** considering future growth
6. **Test Thoroughly** in development environments

### Best Practices Summary:

- **Start with db_schema.xml** for all new schema definitions
- **Use appropriate data types** and sizes
- **Index frequently queried columns** but avoid over-indexing
- **Implement proper foreign key relationships**
- **Include timestamps** for auditing
- **Plan for internationalization** with store-scoped data
- **Document your schema** with meaningful comments

### Performance Considerations:

- **Choose correct storage engines** (InnoDB for most cases)
- **Design efficient indexes** based on query patterns
- **Consider partitioning** for large datasets
- **Monitor query performance** and optimize as needed
- **Use covering indexes** for read-heavy operations

Understanding database schema in Magento 2 enables you to build professional, maintainable, and high-performance e-commerce solutions that can scale with business needs while maintaining data integrity and optimal performance.