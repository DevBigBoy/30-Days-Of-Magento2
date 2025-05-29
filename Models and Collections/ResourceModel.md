# Magento 2 Resource Model - Comprehensive Learning Guide

## Table of Contents
1. [Introduction](#introduction)
2. [What are Resource Models?](#what-are-resource-models)
3. [Architecture and Structure](#architecture-and-structure)
4. [Types of Resource Models](#types-of-resource-models)
5. [Core Capabilities](#core-capabilities)
6. [Implementation Guide](#implementation-guide)
7. [Database Operations](#database-operations)
8. [Advanced Features](#advanced-features)
9. [Integration Patterns](#integration-patterns)
10. [Performance Optimization](#performance-optimization)
11. [Best Practices](#best-practices)
12. [Real-World Examples](#real-world-examples)
13. [Testing Resource Models](#testing-resource-models)
14. [Troubleshooting](#troubleshooting)

---

## Introduction

Resource Models are the **data access layer** in Magento 2's architecture. They serve as the bridge between your application's business logic (Models) and the database, providing a consistent, secure, and optimized way to perform database operations.

**Key Insight**: Resource Models are not just about database access—they're about creating maintainable, secure, and performant data persistence layers that can scale with your business needs.

## What are Resource Models?

A **Resource Model** is a PHP class responsible for:
- **Database Operations**: CRUD (Create, Read, Update, Delete) operations
- **Data Persistence**: Saving and retrieving model data
- **Query Building**: Constructing and executing SQL queries
- **Connection Management**: Handling database connections
- **Data Validation**: Ensuring data integrity at the database level

### Core Responsibilities:
```
Model (Business Logic)
        ↓
Resource Model (Data Access)
        ↓
Database (Data Storage)
```

## Architecture and Structure

### Class Hierarchy:
```
Magento\Framework\Model\ResourceModel\AbstractResource
    ↓
Magento\Framework\Model\ResourceModel\Db\AbstractDb
    ↓
Magento\Framework\Model\ResourceModel\Db\VersionControl\AbstractDb
    ↓
Your\Module\Model\ResourceModel\Entity
```

### Core Components:

#### 1. **Abstract Resource**
Base functionality for all resource models:
```php
namespace Magento\Framework\Model\ResourceModel;

abstract class AbstractResource
{
    abstract public function getIdFieldName();
    abstract public function load(AbstractModel $object, $value, $field = null);
    abstract public function save(AbstractModel $object);
    abstract public function delete(AbstractModel $object);
}
```

#### 2. **Database Resource (AbstractDb)**
Database-specific functionality:
```php
namespace Magento\Framework\Model\ResourceModel\Db;

abstract class AbstractDb extends AbstractResource
{
    protected $_connectionName;           // Database connection name
    protected $_mainTable;               // Primary table name
    protected $_idFieldName;             // Primary key field
    protected $_isPkAutoIncrement = true; // Auto-increment primary key
    
    // Database operations
    protected function _init($mainTable, $idFieldName);
    public function getConnection();
    public function getMainTable();
    public function getTable($tableName);
}
```

#### 3. **Version Control Resource**
Enhanced functionality with version control:
```php
namespace Magento\Framework\Model\ResourceModel\Db\VersionControl;

abstract class AbstractDb extends \Magento\Framework\Model\ResourceModel\Db\AbstractDb
{
    // Version control capabilities
    // Entity snapshots for change tracking
    // Optimistic locking
}
```

## Types of Resource Models

### 1. **Flat Table Resource Models**
For entities stored in single tables:

```php
<?php
namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class SimpleEntity extends AbstractDb
{
    /**
     * Table name
     */
    const MAIN_TABLE = 'vendor_module_simple_entity';
    
    /**
     * Primary key field
     */
    const ID_FIELD_NAME = 'entity_id';

    /**
     * Initialize resource model
     */
    protected function _construct()
    {
        $this->_init(self::MAIN_TABLE, self::ID_FIELD_NAME);
    }
}
```

### 2. **EAV Resource Models**
For Entity-Attribute-Value entities:

```php
<?php
namespace Vendor\Module\Model\ResourceModel;

use Magento\Eav\Model\Entity\AbstractEntity;
use Magento\Eav\Model\Entity\Context;

class EavEntity extends AbstractEntity
{
    /**
     * Entity type code
     */
    protected $_entityTypeCode = 'vendor_module_eav_entity';

    /**
     * Initialize resource model
     */
    protected function _construct()
    {
        $this->setType($this->_entityTypeCode);
        $this->setConnection('vendor_module_read', 'vendor_module_write');
    }

    /**
     * Default attributes that should always be loaded
     */
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

### 3. **Version Control Resource Models**
For entities requiring change tracking:

```php
<?php
namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\VersionControl\AbstractDb;

class VersionedEntity extends AbstractDb
{
    /**
     * Initialize resource model
     */
    protected function _construct()
    {
        $this->_init('vendor_module_versioned_entity', 'entity_id');
    }

    /**
     * Define which fields should be tracked for changes
     */
    protected function _getLoadSelect($field, $value, $object)
    {
        $select = parent::_getLoadSelect($field, $value, $object);
        
        // Add version tracking fields
        $select->columns(['version', 'updated_at']);
        
        return $select;
    }
}
```

## Core Capabilities

### 1. **CRUD Operations**

#### **Create (Save)**
```php
public function save(\Magento\Framework\Model\AbstractModel $object)
{
    $connection = $this->getConnection();
    $connection->beginTransaction();
    
    try {
        // Validate object
        $this->_beforeSave($object);
        
        if ($object->getId()) {
            // Update existing record
            $this->_updateObject($object);
        } else {
            // Insert new record
            $this->_insertObject($object);
        }
        
        $this->_afterSave($object);
        $connection->commit();
        
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }
    
    return $this;
}

/**
 * Custom save method with validation
 */
public function saveWithValidation($object, array $validationRules = [])
{
    // Pre-save validation
    foreach ($validationRules as $field => $rule) {
        if (!$this->validateField($object->getData($field), $rule)) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Validation failed for field: %1', $field)
            );
        }
    }
    
    return $this->save($object);
}
```

#### **Read (Load)**
```php
public function load(\Magento\Framework\Model\AbstractModel $object, $value, $field = null)
{
    $field = $field ?: $this->getIdFieldName();
    
    $connection = $this->getConnection();
    $select = $connection->select()
        ->from($this->getMainTable())
        ->where($field . ' = ?', $value);
    
    $data = $connection->fetchRow($select);
    
    if ($data) {
        $object->setData($data);
        $this->_afterLoad($object);
    }
    
    return $this;
}

/**
 * Load by multiple fields
 */
public function loadByFields($object, array $fields)
{
    $connection = $this->getConnection();
    $select = $connection->select()->from($this->getMainTable());
    
    foreach ($fields as $field => $value) {
        $select->where($field . ' = ?', $value);
    }
    
    $data = $connection->fetchRow($select);
    
    if ($data) {
        $object->setData($data);
        $this->_afterLoad($object);
    }
    
    return $this;
}

/**
 * Load with custom conditions
 */
public function loadWithConditions($object, array $conditions)
{
    $connection = $this->getConnection();
    $select = $connection->select()->from($this->getMainTable());
    
    foreach ($conditions as $condition) {
        $select->where(
            $condition['field'] . ' ' . $condition['operator'] . ' ?',
            $condition['value']
        );
    }
    
    $data = $connection->fetchRow($select);
    
    if ($data) {
        $object->setData($data);
        $this->_afterLoad($object);
    }
    
    return $this;
}
```

#### **Update**
```php
/**
 * Update specific fields
 */
public function updateFields($entityId, array $data)
{
    $connection = $this->getConnection();
    
    return $connection->update(
        $this->getMainTable(),
        $data,
        [$this->getIdFieldName() . ' = ?' => $entityId]
    );
}

/**
 * Bulk update
 */
public function bulkUpdate(array $updates)
{
    $connection = $this->getConnection();
    $connection->beginTransaction();
    
    try {
        foreach ($updates as $entityId => $data) {
            $connection->update(
                $this->getMainTable(),
                $data,
                [$this->getIdFieldName() . ' = ?' => $entityId]
            );
        }
        
        $connection->commit();
        return true;
        
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }
}

/**
 * Conditional update
 */
public function updateWhere(array $data, array $where)
{
    $connection = $this->getConnection();
    
    $whereConditions = [];
    foreach ($where as $field => $value) {
        if (is_array($value)) {
            $whereConditions[] = $connection->prepareSqlCondition($field, $value);
        } else {
            $whereConditions[$field . ' = ?'] = $value;
        }
    }
    
    return $connection->update($this->getMainTable(), $data, $whereConditions);
}
```

#### **Delete**
```php
public function delete(\Magento\Framework\Model\AbstractModel $object)
{
    $connection = $this->getConnection();
    $connection->beginTransaction();
    
    try {
        $this->_beforeDelete($object);
        
        $connection->delete(
            $this->getMainTable(),
            [$this->getIdFieldName() . ' = ?' => $object->getId()]
        );
        
        $this->_afterDelete($object);
        $connection->commit();
        
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }
    
    return $this;
}

/**
 * Bulk delete
 */
public function bulkDelete(array $entityIds)
{
    $connection = $this->getConnection();
    
    return $connection->delete(
        $this->getMainTable(),
        [$this->getIdFieldName() . ' IN (?)' => $entityIds]
    );
}

/**
 * Delete with conditions
 */
public function deleteWhere(array $conditions)
{
    $connection = $this->getConnection();
    
    $whereConditions = [];
    foreach ($conditions as $field => $value) {
        if (is_array($value)) {
            $whereConditions[] = $connection->prepareSqlCondition($field, $value);
        } else {
            $whereConditions[$field . ' = ?'] = $value;
        }
    }
    
    return $connection->delete($this->getMainTable(), $whereConditions);
}
```

### 2. **Query Building and Execution**

#### **Select Queries**
```php
/**
 * Get records with pagination
 */
public function getRecordsWithPagination($page = 1, $pageSize = 20, array $filters = [])
{
    $connection = $this->getConnection();
    $select = $connection->select()
        ->from($this->getMainTable())
        ->limitPage($page, $pageSize);
    
    // Apply filters
    foreach ($filters as $field => $value) {
        $select->where($field . ' = ?', $value);
    }
    
    return $connection->fetchAll($select);
}

/**
 * Get aggregated data
 */
public function getAggregatedData($groupBy, array $aggregates = [])
{
    $connection = $this->getConnection();
    $select = $connection->select()->from($this->getMainTable());
    
    // Add aggregation columns
    foreach ($aggregates as $alias => $expression) {
        $select->columns([$alias => new \Zend_Db_Expr($expression)]);
    }
    
    if ($groupBy) {
        $select->group($groupBy);
    }
    
    return $connection->fetchAll($select);
}

/**
 * Complex joins
 */
public function getRecordsWithRelations(array $joins = [])
{
    $connection = $this->getConnection();
    $select = $connection->select()->from(['main' => $this->getMainTable()]);
    
    foreach ($joins as $join) {
        $joinMethod = $join['type'] . 'Join'; // leftJoin, innerJoin, etc.
        $select->$joinMethod(
            [$join['alias'] => $this->getTable($join['table'])],
            $join['condition'],
            $join['columns'] ?? []
        );
    }
    
    return $connection->fetchAll($select);
}
```

#### **Raw SQL Execution**
```php
/**
 * Execute custom query
 */
public function executeCustomQuery($sql, array $params = [])
{
    $connection = $this->getConnection();
    
    return $connection->query($sql, $params);
}

/**
 * Execute stored procedure
 */
public function executeStoredProcedure($procedureName, array $params = [])
{
    $connection = $this->getConnection();
    
    $sql = "CALL {$procedureName}(" . implode(',', array_fill(0, count($params), '?')) . ")";
    
    return $connection->query($sql, $params);
}

/**
 * Get database statistics
 */
public function getTableStatistics()
{
    $connection = $this->getConnection();
    
    $sql = "SHOW TABLE STATUS LIKE ?";
    $result = $connection->fetchRow($sql, [$this->getMainTable()]);
    
    return [
        'rows' => $result['Rows'] ?? 0,
        'data_length' => $result['Data_length'] ?? 0,
        'index_length' => $result['Index_length'] ?? 0,
        'auto_increment' => $result['Auto_increment'] ?? 0
    ];
}
```

### 3. **Transaction Management**

#### **Manual Transaction Control**
```php
/**
 * Execute operations in transaction
 */
public function executeInTransaction(callable $operations)
{
    $connection = $this->getConnection();
    $connection->beginTransaction();
    
    try {
        $result = $operations($connection);
        $connection->commit();
        return $result;
        
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }
}

/**
 * Batch operations with transaction
 */
public function batchOperations(array $operations)
{
    return $this->executeInTransaction(function($connection) use ($operations) {
        $results = [];
        
        foreach ($operations as $operation) {
            switch ($operation['type']) {
                case 'insert':
                    $results[] = $connection->insert(
                        $this->getTable($operation['table']),
                        $operation['data']
                    );
                    break;
                    
                case 'update':
                    $results[] = $connection->update(
                        $this->getTable($operation['table']),
                        $operation['data'],
                        $operation['where']
                    );
                    break;
                    
                case 'delete':
                    $results[] = $connection->delete(
                        $this->getTable($operation['table']),
                        $operation['where']
                    );
                    break;
            }
        }
        
        return $results;
    });
}
```

### 4. **Data Validation and Integrity**

#### **Validation Hooks**
```php
/**
 * Before save validation
 */
protected function _beforeSave(\Magento\Framework\Model\AbstractModel $object)
{
    // Validate required fields
    $this->validateRequiredFields($object);
    
    // Validate data types
    $this->validateDataTypes($object);
    
    // Validate business rules
    $this->validateBusinessRules($object);
    
    // Set timestamps
    if (!$object->getId()) {
        $object->setCreatedAt(date('Y-m-d H:i:s'));
    }
    $object->setUpdatedAt(date('Y-m-d H:i:s'));
    
    return parent::_beforeSave($object);
}

/**
 * Validate required fields
 */
protected function validateRequiredFields($object)
{
    $requiredFields = ['name', 'email', 'status'];
    
    foreach ($requiredFields as $field) {
        if (!$object->getData($field)) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Required field "%1" is missing.', $field)
            );
        }
    }
}

/**
 * Validate data types
 */
protected function validateDataTypes($object)
{
    // Email validation
    if ($object->getData('email') && !filter_var($object->getData('email'), FILTER_VALIDATE_EMAIL)) {
        throw new \Magento\Framework\Exception\ValidatorException(
            __('Invalid email format.')
        );
    }
    
    // Numeric validation
    if ($object->getData('price') && !is_numeric($object->getData('price'))) {
        throw new \Magento\Framework\Exception\ValidatorException(
            __('Price must be numeric.')
        );
    }
    
    // Date validation
    if ($object->getData('start_date')) {
        $date = \DateTime::createFromFormat('Y-m-d H:i:s', $object->getData('start_date'));
        if (!$date) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Invalid date format.')
            );
        }
    }
}

/**
 * Business rule validation
 */
protected function validateBusinessRules($object)
{
    // Unique email validation
    if ($object->getData('email')) {
        $existingId = $this->getIdByEmail($object->getData('email'));
        if ($existingId && $existingId != $object->getId()) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Email already exists.')
            );
        }
    }
    
    // Status transitions
    if ($object->getId() && $object->getData('status')) {
        $currentStatus = $this->getCurrentStatus($object->getId());
        if (!$this->isValidStatusTransition($currentStatus, $object->getData('status'))) {
            throw new \Magento\Framework\Exception\ValidatorException(
                __('Invalid status transition.')
            );
        }
    }
}
```

## Database Operations

### 1. **Bulk Operations**

#### **Bulk Insert**
```php
/**
 * Insert multiple records
 */
public function bulkInsert(array $data)
{
    if (empty($data)) {
        return 0;
    }
    
    $connection = $this->getConnection();
    
    return $connection->insertMultiple($this->getMainTable(), $data);
}

/**
 * Insert on duplicate key update
 */
public function insertOnDuplicate(array $data, array $updateFields = [])
{
    if (empty($data)) {
        return 0;
    }
    
    $connection = $this->getConnection();
    
    return $connection->insertOnDuplicate(
        $this->getMainTable(),
        $data,
        $updateFields
    );
}

/**
 * Batch insert with validation
 */
public function batchInsertWithValidation(array $records, callable $validator = null)
{
    $validRecords = [];
    $errors = [];
    
    foreach ($records as $index => $record) {
        try {
            if ($validator) {
                $validator($record);
            }
            $this->validateRecord($record);
            $validRecords[] = $record;
            
        } catch (\Exception $e) {
            $errors[$index] = $e->getMessage();
        }
    }
    
    if (!empty($validRecords)) {
        $this->bulkInsert($validRecords);
    }
    
    return [
        'inserted' => count($validRecords),
        'errors' => $errors
    ];
}
```

#### **Bulk Update**
```php
/**
 * Update multiple records with different data
 */
public function bulkUpdateDifferentData(array $updates)
{
    $connection = $this->getConnection();
    $connection->beginTransaction();
    
    try {
        $affectedRows = 0;
        
        foreach ($updates as $id => $data) {
            $affected = $connection->update(
                $this->getMainTable(),
                $data,
                [$this->getIdFieldName() . ' = ?' => $id]
            );
            $affectedRows += $affected;
        }
        
        $connection->commit();
        return $affectedRows;
        
    } catch (\Exception $e) {
        $connection->rollBack();
        throw $e;
    }
}

/**
 * Conditional bulk update
 */
public function bulkUpdateConditional(array $data, array $conditions)
{
    $connection = $this->getConnection();
    
    $whereConditions = [];
    foreach ($conditions as $field => $value) {
        if (is_array($value)) {
            $whereConditions[] = $connection->prepareSqlCondition($field, $value);
        } else {
            $whereConditions[$field . ' = ?'] = $value;
        }
    }
    
    return $connection->update($this->getMainTable(), $data, $whereConditions);
}
```

### 2. **Advanced Queries**

#### **Subqueries and CTEs**
```php
/**
 * Query with subquery
 */
public function getRecordsWithSubquery($subqueryField, $subqueryConditions)
{
    $connection = $this->getConnection();
    
    // Build subquery
    $subquery = $connection->select()
        ->from($this->getTable('related_table'), [$subqueryField])
        ->where('status = ?', 1);
    
    foreach ($subqueryConditions as $field => $value) {
        $subquery->where($field . ' = ?', $value);
    }
    
    // Main query with subquery
    $select = $connection->select()
        ->from($this->getMainTable())
        ->where($subqueryField . ' IN (?)', $subquery);
    
    return $connection->fetchAll($select);
}

/**
 * Complex analytics query
 */
public function getAnalyticsData($dateFrom, $dateTo)
{
    $connection = $this->getConnection();
    
    $select = $connection->select()
        ->from(['main' => $this->getMainTable()])
        ->joinLeft(
            ['related' => $this->getTable('related_table')],
            'main.related_id = related.id',
            []
        )
        ->columns([
            'date' => 'DATE(main.created_at)',
            'total_count' => 'COUNT(*)',
            'active_count' => 'SUM(CASE WHEN main.status = 1 THEN 1 ELSE 0 END)',
            'avg_value' => 'AVG(main.value)',
            'max_value' => 'MAX(main.value)',
            'min_value' => 'MIN(main.value)'
        ])
        ->where('main.created_at >= ?', $dateFrom)
        ->where('main.created_at <= ?', $dateTo)
        ->group('DATE(main.created_at)')
        ->order('DATE(main.created_at) ASC');
    
    return $connection->fetchAll($select);
}

/**
 * Hierarchical data query
 */
public function getHierarchicalData($parentId = null, $maxDepth = 5)
{
    $connection = $this->getConnection();
    
    // Recursive CTE for hierarchical data
    $cte = "WITH RECURSIVE hierarchy AS (
        SELECT *, 0 as depth, CAST(id AS CHAR(1000)) as path
        FROM {$this->getMainTable()}
        WHERE parent_id " . ($parentId ? "= {$parentId}" : "IS NULL") . "
        
        UNION ALL
        
        SELECT child.*, parent.depth + 1, CONCAT(parent.path, ',', child.id)
        FROM {$this->getMainTable()} child
        INNER JOIN hierarchy parent ON child.parent_id = parent.id
        WHERE parent.depth < {$maxDepth}
    )
    SELECT * FROM hierarchy ORDER BY path";
    
    return $connection->fetchAll($cte);
}
```

#### **Performance Optimized Queries**
```php
/**
 * Get records with covering index
 */
public function getRecordsOptimized(array $filters, $limit = 100)
{
    $connection = $this->getConnection();
    $select = $connection->select()
        ->from($this->getMainTable(), ['id', 'name', 'status']) // Only indexed columns
        ->limit($limit);
    
    // Use index hints for large tables
    $select->forceIndex('idx_status_created');
    
    foreach ($filters as $field => $value) {
        $select->where($field . ' = ?', $value);
    }
    
    return $connection->fetchAll($select);
}

/**
 * Streaming large result sets
 */
public function streamLargeResultSet(callable $processor, $batchSize = 1000)
{
    $connection = $this->getConnection();
    $offset = 0;
    $totalProcessed = 0;
    
    do {
        $select = $connection->select()
            ->from($this->getMainTable())
            ->limit($batchSize, $offset);
        
        $batch = $connection->fetchAll($select);
        
        foreach ($batch as $row) {
            $processor($row);
            $totalProcessed++;
        }
        
        $offset += $batchSize;
        
        // Force garbage collection
        if ($offset % 10000 === 0) {
            gc_collect_cycles();
        }
        
    } while (count($batch) === $batchSize);
    
    return $totalProcessed;
}
```

## Advanced Features

### 1. **Event Dispatching**

#### **Custom Events**
```php
/**
 * Save with events
 */
public function save(\Magento\Framework\Model\AbstractModel $object)
{
    // Dispatch before save event
    $this->eventManager->dispatch('vendor_module_entity_save_before', [
        'entity' => $object,
        'resource' => $this
    ]);
    
    $isNew = !$object->getId();
    
    try {
        $result = parent::save($object);
        
        // Dispatch after save event
        $this->eventManager->dispatch('vendor_module_entity_save_after', [
            'entity' => $object,
            'resource' => $this,
            'is_new' => $isNew
        ]);
        
        return $result;
        
    } catch (\Exception $e) {
        // Dispatch error event
        $this->eventManager->dispatch('vendor_module_entity_save_error', [
            'entity' => $object,
            'resource' => $this,
            'exception' => $e
        ]);
        
        throw $e;
    }
}

/**
 * Load with events
 */
public function load(\Magento\Framework\Model\AbstractModel $object, $value, $field = null)
{
    // Dispatch before load event
    $this->eventManager->dispatch('vendor_module_entity_load_before', [
        'entity' => $object,
        'value' => $value,
        'field' => $field
    ]);
    
    $result = parent::load($object, $value, $field);
    
    if ($object->getId()) {
        // Dispatch after load event
        $this->eventManager->dispatch('vendor_module_entity_load_after', [
            'entity' => $object,
            'resource' => $this
        ]);
    }
    
    return $result;
}
```

### 2. **Caching Integration**

#### **Cache-Aware Resource Model**
```php
/**
 * Cache-aware load
 */
public function loadWithCache(\Magento\Framework\Model\AbstractModel $object, $value, $field = null)
{
    $cacheKey = $this->getCacheKey($field ?: $this->getIdFieldName(), $value);
    $cachedData = $this->cache->load($cacheKey);
    
    if ($cachedData) {
        $data = $this->serializer->unserialize($cachedData);
        $object->setData($data);
        $this->_afterLoad($object);
        return $this;
    }
    
    // Load from database
    $this->load($object, $value, $field);
    
    // Cache the result
    if ($object->getId()) {
        $this->cache->save(
            $this->serializer->serialize($object->getData()),
            $cacheKey,
            ['vendor_module_entity'],
            3600
        );
    }
    
    return $this;
}

/**
 * Cache invalidation on save
 */
public function save(\Magento\Framework\Model\AbstractModel $object)
{
    $result = parent::save($object);
    
    // Invalidate cache
    $cacheKey = $this->getCacheKey($this->getIdFieldName(), $object->getId());
    $this->cache->remove($cacheKey);
    
    // Clean cache tags
    $this->cache->clean(['vendor_module_entity']);
    
    return $result;
}

/**
 * Generate cache key
 */
private function getCacheKey($field, $value)
{
    return 'vendor_module_entity_' . $field . '_' . $value;
}
```

### 3. **Multi-Store Support**

#### **Store-Aware Resource Model**
```php
/**
 * Load with store context
 */
public function loadByStore($object, $value, $storeId, $field = null)
{
    $field = $field ?: $this->getIdFieldName();
    
    $connection = $this->getConnection();
    $select = $connection->select()
        ->from(['main' => $this->getMainTable()])
        ->where('main.' . $field . ' = ?', $value);
    
    // Join store-specific data if exists
    $storeTable = $this->getTable($this->getMainTable() . '_store');
    if ($connection->isTableExists($storeTable)) {
        $select->joinLeft(
            ['store' => $storeTable],
            'main.' . $this->getIdFieldName() . ' = store.' . $this->getIdFieldName() . 
            ' AND store.store_id = ' . (int)$storeId,
            []
        );
        
        // Use store-specific values if available
        $storeFields = ['name', 'description', 'status'];
        foreach ($storeFields as $storeField) {
            $select->columns([
                $storeField => new \Zend_Db_Expr(
                    'COALESCE(store.' . $storeField . ', main.' . $storeField . ')'
                )
            ]);
        }
    }
    
    $data = $connection->fetchRow($select);
    
    if ($data) {
        $object->setData($data);
        $object->setStoreId($storeId);
        $this->_afterLoad($object);
    }
    
    return $this;
}

/**
 * Save store-specific data
 */
public function saveStoreData($object, $storeId)
{
    $storeTable = $this->getTable($this->getMainTable() . '_store');
    $connection = $this->getConnection();
    
    if (!$connection->isTableExists($storeTable)) {
        return $this;
    }
    
    $storeData = [
        $this->getIdFieldName() => $object->getId(),
        'store_id' => $storeId
    ];
    
    // Add store-specific fields
    $storeFields = ['name', 'description', 'status'];
    foreach ($storeFields as $field) {
        if ($object->hasData($field . '_store')) {
            $storeData[$field] = $object->getData($field . '_store');
        }
    }
    
    $connection->insertOnDuplicate(
        $storeTable,
        $storeData,
        array_keys($storeData)
    );
    
    return $this;
}
```

### 4. **Indexing Support**

#### **Indexable Resource Model**
```php
/**
 * Mark entity for reindex
 */
public function save(\Magento\Framework\Model\AbstractModel $object)
{
    $originalData = $object->getOrigData();
    $result = parent::save($object);
    
    // Check if indexable fields changed
    $indexableFields = ['name', 'price', 'status', 'category_id'];
    $needsReindex = false;
    
    foreach ($indexableFields as $field) {
        if ($object->getData($field) !== $originalData[$field]) {
            $needsReindex = true;
            break;
        }
    }
    
    if ($needsReindex) {
        $this->markForReindex($object->getId());
    }
    
    return $result;
}

/**
 * Mark entity for reindex
 */
private function markForReindex($entityId)
{
    $indexerIds = [
        'vendor_module_entity_fulltext',
        'vendor_module_entity_category'
    ];
    
    foreach ($indexerIds as $indexerId) {
        $this->indexerRegistry->get($indexerId)->reindexRow($entityId);
    }
}

/**
 * Get data for indexing
 */
public function getDataForIndex($entityIds = null)
{
    $connection = $this->getConnection();
    $select = $connection->select()
        ->from(['main' => $this->getMainTable()])
        ->joinLeft(
            ['category' => $this->getTable('vendor_module_entity_category')],
            'main.entity_id = category.entity_id',
            ['category_ids' => 'GROUP_CONCAT(category.category_id)']
        )
        ->group('main.entity_id');
    
    if ($entityIds) {
        $select->where('main.entity_id IN (?)', $entityIds);
    }
    
    return $connection->fetchAll($select);
}
```

## Integration Patterns

### 1. **Repository Pattern Integration**

#### **Resource Model in Repository**
```php
<?php
namespace Vendor\Module\Model;

use Vendor\Module\Api\EntityRepositoryInterface;
use Vendor\Module\Model\ResourceModel\Entity as EntityResource;
use Vendor\Module\Model\EntityFactory;

class EntityRepository implements EntityRepositoryInterface
{
    private EntityResource $resource;
    private EntityFactory $entityFactory;

    public function __construct(
        EntityResource $resource,
        EntityFactory $entityFactory
    ) {
        $this->resource = $resource;
        $this->entityFactory = $entityFactory;
    }

    public function save(EntityInterface $entity): EntityInterface
    {
        try {
            $this->resource->save($entity);
        } catch (\Exception $exception) {
            throw new CouldNotSaveException(
                __('Could not save entity: %1', $exception->getMessage())
            );
        }
        
        return $entity;
    }

    public function getById(int $entityId): EntityInterface
    {
        $entity = $this->entityFactory->create();
        $this->resource->load($entity, $entityId);
        
        if (!$entity->getId()) {
            throw new NoSuchEntityException(
                __('Entity with id "%1" does not exist.', $entityId)
            );
        }
        
        return $entity;
    }

    public function delete(EntityInterface $entity): bool
    {
        try {
            $this->resource->delete($entity);
        } catch (\Exception $exception) {
            throw new CouldNotDeleteException(
                __('Could not delete entity: %1', $exception->getMessage())
            );
        }
        
        return true;
    }

    public function getByEmail(string $email): EntityInterface
    {
        $entity = $this->entityFactory->create();
        $this->resource->loadByFields($entity, ['email' => $email]);
        
        if (!$entity->getId()) {
            throw new NoSuchEntityException(
                __('Entity with email "%1" does not exist.', $email)
            );
        }
        
        return $entity;
    }
}
```

### 2. **Service Layer Integration**

#### **Resource Model in Services**
```php
<?php
namespace Vendor\Module\Model\Service;

use Vendor\Module\Model\ResourceModel\Entity as EntityResource;

class EntityManagementService
{
    private EntityResource $entityResource;

    public function __construct(EntityResource $entityResource)
    {
        $this->entityResource = $entityResource;
    }

    public function bulkUpdateStatus(array $entityIds, $status)
    {
        return $this->entityResource->updateWhere(
            ['status' => $status, 'updated_at' => date('Y-m-d H:i:s')],
            ['entity_id' => ['in' => $entityIds]]
        );
    }

    public function getStatistics($dateFrom = null, $dateTo = null)
    {
        $conditions = [];
        if ($dateFrom) {
            $conditions['created_at']['from'] = $dateFrom;
        }
        if ($dateTo) {
            $conditions['created_at']['to'] = $dateTo;
        }

        return $this->entityResource->getAggregatedData('status', [
            'total_count' => 'COUNT(*)',
            'avg_value' => 'AVG(value)',
            'sum_value' => 'SUM(value)'
        ]);
    }

    public function cleanupOldRecords($daysBefore = 365)
    {
        $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$daysBefore} days"));
        
        return $this->entityResource->deleteWhere([
            'created_at' => ['lt' => $cutoffDate],
            'status' => ['in' => [0, 2]] // Only inactive records
        ]);
    }
}
```

## Performance Optimization

### 1. **Query Optimization**

#### **Index-Aware Queries**
```php
/**
 * Optimized queries using proper indexes
 */
public function getOptimizedRecords(array $filters)
{
    $connection = $this->getConnection();
    $select = $connection->select()->from($this->getMainTable());
    
    // Order filters by selectivity (most selective first)
    $orderedFilters = $this->orderFiltersBySelectivity($filters);
    
    foreach ($orderedFilters as $field => $value) {
        $select->where($field . ' = ?', $value);
    }
    
    // Use covering index when possible
    $select->forceIndex('idx_covering_status_created_name');
    
    return $connection->fetchAll($select);
}

/**
 * Order filters by selectivity for optimal performance
 */
private function orderFiltersBySelectivity(array $filters)
{
    // Define selectivity order (most selective first)
    $selectivityOrder = [
        'id' => 1,
        'email' => 2,
        'user_id' => 3,
        'status' => 4,
        'type' => 5,
        'created_at' => 6
    ];
    
    uksort($filters, function($a, $b) use ($selectivityOrder) {
        $aOrder = $selectivityOrder[$a] ?? 999;
        $bOrder = $selectivityOrder[$b] ?? 999;
        return $aOrder - $bOrder;
    });
    
    return $filters;
}

/**
 * Use prepared statements for repeated queries
 */
public function getPreparedQuery($sql)
{
    static $preparedStatements = [];
    
    $key = md5($sql);
    
    if (!isset($preparedStatements[$key])) {
        $preparedStatements[$key] = $this->getConnection()->prepare($sql);
    }
    
    return $preparedStatements[$key];
}
```

### 2. **Connection Management**

#### **Read/Write Splitting**
```php
/**
 * Use read replica for read operations
 */
public function getReadConnection()
{
    return $this->resource->getConnection('read');
}

/**
 * Use master for write operations
 */
public function getWriteConnection()
{
    return $this->resource->getConnection('write');
}

/**
 * Load with read connection
 */
public function loadReadOnly($object, $value, $field = null)
{
    $field = $field ?: $this->getIdFieldName();
    
    $connection = $this->getReadConnection();
    $select = $connection->select()
        ->from($this->getMainTable())
        ->where($field . ' = ?', $value);
    
    $data = $connection->fetchRow($select);
    
    if ($data) {
        $object->setData($data);
        $this->_afterLoad($object);
    }
    
    return $this;
}
```

### 3. **Memory Management**

#### **Memory-Efficient Operations**
```php
/**
 * Process large datasets with minimal memory usage
 */
public function processLargeDataset(callable $processor, $batchSize = 1000)
{
    $connection = $this->getConnection();
    $offset = 0;
    $totalProcessed = 0;
    
    // Use unbuffered query for large results
    $connection->getConnection()->setAttribute(\PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, false);
    
    try {
        do {
            $select = $connection->select()
                ->from($this->getMainTable())
                ->limit($batchSize, $offset);
            
            $stmt = $connection->query($select);
            $processedInBatch = 0;
            
            while ($row = $stmt->fetch()) {
                $processor($row);
                $processedInBatch++;
                $totalProcessed++;
                
                // Free memory periodically
                if ($totalProcessed % 10000 === 0) {
                    gc_collect_cycles();
                }
            }
            
            $offset += $batchSize;
            
        } while ($processedInBatch === $batchSize);
        
    } finally {
        // Restore buffered queries
        $connection->getConnection()->setAttribute(\PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, true);
    }
    
    return $totalProcessed;
}
```

## Best Practices

### 1. **Code Organization**

#### **Separation of Concerns**
```php
// Good - Specific methods for different operations
class EntityResource extends AbstractDb
{
    // Basic CRUD
    public function save($object) { /* ... */ }
    public function load($object, $value, $field = null) { /* ... */ }
    public function delete($object) { /* ... */ }
    
    // Specialized queries
    public function loadByEmail($object, $email) { /* ... */ }
    public function getActiveEntities() { /* ... */ }
    public function getStatisticsByPeriod($from, $to) { /* ... */ }
    
    // Bulk operations
    public function bulkInsert(array $data) { /* ... */ }
    public function bulkUpdateStatus(array $ids, $status) { /* ... */ }
}

// Avoid - Mixing concerns in single methods
class BadEntityResource extends AbstractDb
{
    public function doEverything($object, $type) {
        // Don't mix save, load, validation, and business logic
    }
}
```

#### **Configuration-Driven Behavior**
```php
/**
 * Configurable resource model
 */
class ConfigurableEntityResource extends AbstractDb
{
    private $config;

    public function __construct(
        Context $context,
        ConfigInterface $config,
        $connectionName = null
    ) {
        parent::__construct($context, $connectionName);
        $this->config = $config;
    }

    protected function _beforeSave($object)
    {
        // Configurable validation
        if ($this->config->isValidationEnabled()) {
            $this->validateObject($object);
        }
        
        // Configurable timestamps
        if ($this->config->isAutoTimestampEnabled()) {
            $this->setTimestamps($object);
        }
        
        return parent::_beforeSave($object);
    }
}
```

### 2. **Error Handling**

#### **Comprehensive Error Management**
```php
/**
 * Error-aware save operation
 */
public function save(\Magento\Framework\Model\AbstractModel $object)
{
    $connection = $this->getConnection();
    $connection->beginTransaction();
    
    try {
        // Pre-save validation
        $this->validateObject($object);
        
        // Save operation
        $result = parent::save($object);
        
        // Post-save operations
        $this->afterSaveOperations($object);
        
        $connection->commit();
        
        // Log successful save
        $this->logger->info('Entity saved successfully', [
            'entity_id' => $object->getId(),
            'entity_type' => get_class($object)
        ]);
        
        return $result;
        
    } catch (\Magento\Framework\Exception\ValidatorException $e) {
        $connection->rollBack();
        $this->logger->warning('Validation error during save', [
            'error' => $e->getMessage(),
            'entity_data' => $object->getData()
        ]);
        throw $e;
        
    } catch (\Magento\Framework\DB\Exception $e) {
        $connection->rollBack();
        $this->logger->error('Database error during save', [
            'error' => $e->getMessage(),
            'entity_id' => $object->getId()
        ]);
        throw new CouldNotSaveException(__('Unable to save entity due to database error.'));
        
    } catch (\Exception $e) {
        $connection->rollBack();
        $this->logger->critical('Unexpected error during save', [
            'error' => $e->getMessage(),
            'trace' => $e->getTraceAsString()
        ]);
        throw new CouldNotSaveException(__('An unexpected error occurred while saving.'));
    }
}
```

### 3. **Security**

#### **SQL Injection Prevention**
```php
/**
 * Secure query building
 */
public function searchEntities($searchTerm, $filters = [])
{
    $connection = $this->getConnection();
    $select = $connection->select()->from($this->getMainTable());
    
    // Always use parameter binding
    if ($searchTerm) {
        $select->where(
            'name LIKE ? OR description LIKE ?',
            '%' . $searchTerm . '%'
        );
    }
    
    // Validate and sanitize filters
    foreach ($filters as $field => $value) {
        if (!$this->isValidFilterField($field)) {
            throw new \InvalidArgumentException("Invalid filter field: {$field}");
        }
        
        $select->where($field . ' = ?', $value);
    }
    
    return $connection->fetchAll($select);
}

/**
 * Validate filter fields against whitelist
 */
private function isValidFilterField($field)
{
    $allowedFields = ['status', 'type', 'category_id', 'created_at'];
    return in_array($field, $allowedFields);
}

/**
 * Escape user input for LIKE queries
 */
private function escapeLikeValue($value)
{
    return str_replace(['%', '_', '\\'], ['\\%', '\\_', '\\\\'], $value);
}
```

## Real-World Examples

### 1. **E-commerce Product Resource Model**

```php
<?php
namespace Vendor\Catalog\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Product extends AbstractDb
{
    const MAIN_TABLE = 'vendor_catalog_product';
    const ID_FIELD = 'entity_id';

    protected function _construct()
    {
        $this->_init(self::MAIN_TABLE, self::ID_FIELD);
    }

    /**
     * Load product with all related data
     */
    public function loadComplete($object, $productId, $storeId = null)
    {
        // Load basic product data
        $this->load($object, $productId);
        
        if (!$object->getId()) {
            return $this;
        }
        
        // Load pricing data
        $this->loadPricingData($object, $storeId);
        
        // Load inventory data
        $this->loadInventoryData($object);
        
        // Load category associations
        $this->loadCategoryData($object);
        
        // Load media gallery
        $this->loadMediaGallery($object);
        
        return $this;
    }

    /**
     * Load pricing data including tier prices and special prices
     */
    private function loadPricingData($object, $storeId)
    {
        $connection = $this->getConnection();
        
        // Base price
        $priceSelect = $connection->select()
            ->from($this->getTable('vendor_catalog_product_price'))
            ->where('product_id = ?', $object->getId())
            ->where('store_id = ? OR store_id = 0', $storeId)
            ->order('store_id DESC')
            ->limit(1);
        
        $priceData = $connection->fetchRow($priceSelect);
        if ($priceData) {
            $object->setPrice($priceData['price']);
            $object->setSpecialPrice($priceData['special_price']);
            $object->setSpecialFromDate($priceData['special_from_date']);
            $object->setSpecialToDate($priceData['special_to_date']);
        }
        
        // Tier prices
        $tierPriceSelect = $connection->select()
            ->from($this->getTable('vendor_catalog_product_tier_price'))
            ->where('product_id = ?', $object->getId())
            ->order(['customer_group_id ASC', 'qty ASC']);
        
        $tierPrices = $connection->fetchAll($tierPriceSelect);
        $object->setTierPrices($tierPrices);
    }

    /**
     * Load inventory data
     */
    private function loadInventoryData($object)
    {
        $connection = $this->getConnection();
        
        $inventorySelect = $connection->select()
            ->from($this->getTable('vendor_catalog_product_inventory'))
            ->where('product_id = ?', $object->getId());
        
        $inventoryData = $connection->fetchRow($inventorySelect);
        if ($inventoryData) {
            $object->setQty($inventoryData['qty']);
            $object->setIsInStock($inventoryData['is_in_stock']);
            $object->setMinQty($inventoryData['min_qty']);
            $object->setManageStock($inventoryData['manage_stock']);
        }
    }

    /**
     * Load category associations
     */
    private function loadCategoryData($object)
    {
        $connection = $this->getConnection();
        
        $categorySelect = $connection->select()
            ->from(['cp' => $this->getTable('vendor_catalog_category_product')])
            ->joinLeft(
                ['ce' => $this->getTable('vendor_catalog_category')],
                'cp.category_id = ce.entity_id',
                ['category_name' => 'ce.name']
            )
            ->where('cp.product_id = ?', $object->getId())
            ->order('cp.position ASC');
        
        $categories = $connection->fetchAll($categorySelect);
        $object->setCategoryIds(array_column($categories, 'category_id'));
        $object->setCategories($categories);
    }

    /**
     * Save product with all related data
     */
    public function saveComplete($object)
    {
        $connection = $this->getConnection();
        $connection->beginTransaction();
        
        try {
            // Save main product data
            parent::save($object);
            
            // Save pricing data
            $this->savePricingData($object);
            
            // Save inventory data
            $this->saveInventoryData($object);
            
            // Save category associations
            $this->saveCategoryData($object);
            
            // Save media gallery
            $this->saveMediaGallery($object);
            
            $connection->commit();
            
        } catch (\Exception $e) {
            $connection->rollBack();
            throw $e;
        }
        
        return $this;
    }

    /**
     * Get products by category with filters
     */
    public function getProductsByCategory($categoryId, $filters = [], $sortOrder = 'position ASC')
    {
        $connection = $this->getConnection();
        
        $select = $connection->select()
            ->from(['p' => $this->getMainTable()])
            ->joinInner(
                ['cp' => $this->getTable('vendor_catalog_category_product')],
                'p.entity_id = cp.product_id',
                ['position']
            )
            ->where('cp.category_id = ?', $categoryId)
            ->where('p.status = ?', 1);
        
        // Apply filters
        foreach ($filters as $field => $value) {
            if (is_array($value)) {
                $select->where($field . ' IN (?)', $value);
            } else {
                $select->where($field . ' = ?', $value);
            }
        }
        
        // Apply sorting
        $select->order($sortOrder);
        
        return $connection->fetchAll($select);
    }

    /**
     * Update product stock status based on quantity
     */
    public function updateStockStatus()
    {
        $connection = $this->getConnection();
        
        // Update in_stock status based on quantity
        $updateSql = "
            UPDATE {$this->getTable('vendor_catalog_product_inventory')} 
            SET is_in_stock = CASE 
                WHEN manage_stock = 1 AND qty <= min_qty THEN 0 
                ELSE 1 
            END
        ";
        
        return $connection->query($updateSql);
    }

    /**
     * Get low stock products
     */
    public function getLowStockProducts($threshold = 10)
    {
        $connection = $this->getConnection();
        
        $select = $connection->select()
            ->from(['p' => $this->getMainTable()])
            ->joinInner(
                ['i' => $this->getTable('vendor_catalog_product_inventory')],
                'p.entity_id = i.product_id',
                ['qty', 'min_qty', 'is_in_stock']
            )
            ->where('i.manage_stock = ?', 1)
            ->where('i.qty <= ?', $threshold)
            ->where('p.status = ?', 1)
            ->order('i.qty ASC');
        
        return $connection->fetchAll($select);
    }

    /**
     * Bulk update product status
     */
    public function bulkUpdateStatus(array $productIds, $status)
    {
        if (empty($productIds)) {
            return 0;
        }
        
        $connection = $this->getConnection();
        
        return $connection->update(
            $this->getMainTable(),
            [
                'status' => $status,
                'updated_at' => date('Y-m-d H:i:s')
            ],
            ['entity_id IN (?)' => $productIds]
        );
    }
}
```

### 2. **Customer Management Resource Model**

```php
<?php
namespace Vendor\Customer\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Customer extends AbstractDb
{
    protected function _construct()
    {
        $this->_init('vendor_customer_entity', 'entity_id');
    }

    /**
     * Load customer with profile data
     */
    public function loadWithProfile($object, $customerId)
    {
        $this->load($object, $customerId);
        
        if (!$object->getId()) {
            return $this;
        }
        
        // Load addresses
        $this->loadAddresses($object);
        
        // Load order statistics
        $this->loadOrderStatistics($object);
        
        // Load loyalty points
        $this->loadLoyaltyData($object);
        
        return $this;
    }

    /**
     * Load customer addresses
     */
    private function loadAddresses($object)
    {
        $connection = $this->getConnection();
        
        $select = $connection->select()
            ->from($this->getTable('vendor_customer_address'))
            ->where('customer_id = ?', $object->getId())
            ->order(['is_default_billing DESC', 'is_default_shipping DESC']);
        
        $addresses = $connection->fetchAll($select);
        $object->setAddresses($addresses);
    }

    /**
     * Load order statistics
     */
    private function loadOrderStatistics($object)
    {
        $connection = $this->getConnection();
        
        $select = $connection->select()
            ->from($this->getTable('sales_order'), [
                'order_count' => 'COUNT(*)',
                'total_spent' => 'SUM(grand_total)',
                'avg_order_value' => 'AVG(grand_total)',
                'first_order_date' => 'MIN(created_at)',
                'last_order_date' => 'MAX(created_at)'
            ])
            ->where('customer_id = ?', $object->getId())
            ->where('state IN (?)', ['complete', 'processing']);
        
        $stats = $connection->fetchRow($select);
        
        if ($stats && $stats['order_count'] > 0) {
            $object->setOrderCount($stats['order_count']);
            $object->setTotalSpent($stats['total_spent']);
            $object->setAvgOrderValue($stats['avg_order_value']);
            $object->setFirstOrderDate($stats['first_order_date']);
            $object->setLastOrderDate($stats['last_order_date']);
            
            // Calculate days since last order
            if ($stats['last_order_date']) {
                $daysSinceLastOrder = floor(
                    (time() - strtotime($stats['last_order_date'])) / 86400
                );
                $object->setDaysSinceLastOrder($daysSinceLastOrder);
            }
        }
    }

    /**
     * Get customer segmentation data
     */
    public function getCustomerSegmentation()
    {
        $connection = $this->getConnection();
        
        $select = $connection->select()
            ->from(['c' => $this->getMainTable()])
            ->joinLeft(
                ['o' => $this->getTable('sales_order')],
                'c.entity_id = o.customer_id AND o.state IN ("complete", "processing")',
                [
                    'order_count' => 'COUNT(o.entity_id)',
                    'total_spent' => 'SUM(o.grand_total)',
                    'avg_order_value' => 'AVG(o.grand_total)',
                    'last_order_date' => 'MAX(o.created_at)'
                ]
            )
            ->group('c.entity_id')
            ->columns([
                'customer_segment' => new \Zend_Db_Expr(
                    'CASE 
                        WHEN SUM(o.grand_total) > 1000 AND DATEDIFF(NOW(), MAX(o.created_at)) < 90 THEN "VIP Active"
                        WHEN SUM(o.grand_total) > 1000 AND DATEDIFF(NOW(), MAX(o.created_at)) >= 90 THEN "VIP Inactive"
                        WHEN SUM(o.grand_total) > 500 AND DATEDIFF(NOW(), MAX(o.created_at)) < 180 THEN "High Value"
                        WHEN COUNT(o.entity_id) > 5 AND DATEDIFF(NOW(), MAX(o.created_at)) < 180 THEN "Loyal"
                        WHEN DATEDIFF(NOW(), MAX(o.created_at)) > 365 THEN "Churned"
                        WHEN COUNT(o.entity_id) = 0 THEN "Never Purchased"
                        ELSE "Regular"
                    END'
                )
            ]);
        
        return $connection->fetchAll($select);
    }

    /**
     * Merge customer accounts
     */
    public function mergeCustomers($primaryCustomerId, $secondaryCustomerId)
    {
        $connection = $this->getConnection();
        $connection->beginTransaction();
        
        try {
            // Update orders
            $connection->update(
                $this->getTable('sales_order'),
                ['customer_id' => $primaryCustomerId],
                ['customer_id = ?' => $secondaryCustomerId]
            );
            
            // Update addresses (avoid duplicates)
            $connection->update(
                $this->getTable('vendor_customer_address'),
                ['customer_id' => $primaryCustomerId],
                ['customer_id = ?' => $secondaryCustomerId]
            );
            
            // Update other related tables
            $relatedTables = [
                'customer_log',
                'loyalty_points',
                'wishlist',
                'customer_reviews'
            ];
            
            foreach ($relatedTables as $table) {
                if ($connection->isTableExists($this->getTable($table))) {
                    $connection->update(
                        $this->getTable($table),
                        ['customer_id' => $primaryCustomerId],
                        ['customer_id = ?' => $secondaryCustomerId]
                    );
                }
            }
            
            // Delete secondary customer
            $connection->delete(
                $this->getMainTable(),
                ['entity_id = ?' => $secondaryCustomerId]
            );
            
            $connection->commit();
            return true;
            
        } catch (\Exception $e) {
            $connection->rollBack();
            throw $e;
        }
    }
}
```

## Testing Resource Models

### 1. **Unit Testing**

#### **Basic Resource Model Test**
```php
<?php
namespace Vendor\Module\Test\Unit\Model\ResourceModel;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Model\ResourceModel\Entity;

class EntityTest extends TestCase
{
    private $resource;
    private $connectionMock;
    private $adapterMock;

    protected function setUp(): void
    {
        $this->connectionMock = $this->createMock(\Magento\Framework\DB\Adapter\AdapterInterface::class);
        $this->adapterMock = $this->createMock(\Magento\Framework\Model\ResourceModel\Db\Context::class);
        
        $this->resource = new Entity($this->adapterMock);
    }

    public function testSaveEntity()
    {
        $entityMock = $this->createMock(\Vendor\Module\Model\Entity::class);
        
        $entityMock->expects($this->once())
            ->method('getId')
            ->willReturn(null);
        
        $entityMock->expects($this->once())
            ->method('getData')
            ->willReturn(['name' => 'Test Entity', 'status' => 1]);
        
        $this->connectionMock->expects($this->once())
            ->method('insert')
            ->with('vendor_module_entity', ['name' => 'Test Entity', 'status' => 1])
            ->willReturn(1);
        
        $result = $this->resource->save($entityMock);
        $this->assertInstanceOf(Entity::class, $result);
    }

    public function testLoadEntity()
    {
        $entityMock = $this->createMock(\Vendor\Module\Model\Entity::class);
        
        $this->connectionMock->expects($this->once())
            ->method('fetchRow')
            ->willReturn(['entity_id' => 1, 'name' => 'Test Entity']);
        
        $entityMock->expects($this->once())
            ->method('setData')
            ->with(['entity_id' => 1, 'name' => 'Test Entity']);
        
        $result = $this->resource->load($entityMock, 1);
        $this->assertInstanceOf(Entity::class, $result);
    }
}
```

### 2. **Integration Testing**

#### **Database Integration Test**
```php
<?php
namespace Vendor\Module\Test\Integration\Model\ResourceModel;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class EntityIntegrationTest extends TestCase
{
    private $resource;
    private $entityFactory;

    protected function setUp(): void
    {
        $objectManager = Bootstrap::getObjectManager();
        $this->resource = $objectManager->get(\Vendor\Module\Model\ResourceModel\Entity::class);
        $this->entityFactory = $objectManager->get(\Vendor\Module\Model\EntityFactory::class);
    }

    /**
     * @magentoDataFixture loadEntityFixtures
     */
    public function testSaveAndLoadEntity()
    {
        $entity = $this->entityFactory->create();
        $entity->setName('Integration Test Entity');
        $entity->setStatus(1);
        
        // Save entity
        $this->resource->save($entity);
        $this->assertNotNull($entity->getId());
        
        // Load entity
        $loadedEntity = $this->entityFactory->create();
        $this->resource->load($loadedEntity, $entity->getId());
        
        $this->assertEquals($entity->getName(), $loadedEntity->getName());
        $this->assertEquals($entity->getStatus(), $loadedEntity->getStatus());
    }

    public function testBulkOperations()
    {
        $data = [
            ['name' => 'Bulk Entity 1', 'status' => 1],
            ['name' => 'Bulk Entity 2', 'status' => 1],
            ['name' => 'Bulk Entity 3', 'status' => 0]
        ];
        
        $insertedRows = $this->resource->bulkInsert($data);
        $this->assertEquals(3, $insertedRows);
        
        // Verify data was inserted
        $connection = $this->resource->getConnection();
        $select = $connection->select()
            ->from($this->resource->getMainTable())
            ->where('name LIKE ?', 'Bulk Entity%');
        
        $results = $connection->fetchAll($select);
        $this->assertCount(3, $results);
    }

    public static function loadEntityFixtures()
    {
        // Load test fixtures
    }
}
```

## Troubleshooting

### Common Issues and Solutions

#### 1. **Performance Issues**
```php
// Problem: Slow queries
// Solution: Add proper indexes and optimize queries

public function optimizeQuery()
{
    $connection = $this->getConnection();
    
    // Use EXPLAIN to analyze query performance
    $select = $connection->select()
        ->from($this->getMainTable())
        ->where('status = ?', 1)
        ->where('created_at > ?', '2024-01-01');
    
    $explainResult = $connection->query('EXPLAIN ' . $select->__toString());
    
    // Log slow queries
    $startTime = microtime(true);
    $result = $connection->fetchAll($select);
    $executionTime = microtime(true) - $startTime;
    
    if ($executionTime > 1.0) { // Log queries taking more than 1 second
        $this->logger->warning('Slow query detected', [
            'query' => $select->__toString(),
            'execution_time' => $executionTime
        ]);
    }
    
    return $result;
}
```

#### 2. **Memory Issues**
```php
// Problem: Memory exhaustion with large datasets
// Solution: Use batch processing

public function processLargeDatasetSafely()
{
    $batchSize = 1000;
    $offset = 0;
    
    do {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable())
            ->limit($batchSize, $offset);
        
        $batch = $connection->fetchAll($select);
        
        foreach ($batch as $row) {
            // Process row
            $this->processRow($row);
        }
        
        $offset += $batchSize;
        
        // Force garbage collection
        if ($offset % 10000 === 0) {
            gc_collect_cycles();
        }
        
    } while (count($batch) === $batchSize);
}
```

#### 3. **Transaction Issues**
```php
// Problem: Nested transactions not properly handled
// Solution: Check transaction state

public function saveWithNestedTransaction($object)
{
    $connection = $this->getConnection();
    $isNestedTransaction = $connection->getTransactionLevel() > 0;
    
    if (!$isNestedTransaction) {
        $connection->beginTransaction();
    }
    
    try {
        $result = parent::save($object);
        
        if (!$isNestedTransaction) {
            $connection->commit();
        }
        
        return $result;
        
    } catch (\Exception $e) {
        if (!$isNestedTransaction) {
            $connection->rollBack();
        }
        throw $e;
    }
}
```

#### 4. **Connection Issues**
```php
// Problem: Database connection lost
// Solution: Implement connection retry logic

public function executeWithRetry($operation, $maxRetries = 3)
{
    $attempts = 0;
    
    while ($attempts < $maxRetries) {
        try {
            return $operation();
            
        } catch (\Magento\Framework\DB\Exception $e) {
            $attempts++;
            
            if ($attempts >= $maxRetries) {
                throw $e;
            }
            
            // Wait before retry
            sleep(pow(2, $attempts)); // Exponential backoff
            
            // Try to reconnect
            $this->getConnection()->closeConnection();
        }
    }
}
```

## Conclusion

Resource Models are the **backbone of data persistence** in Magento 2, providing:

### **Core Functions:**
1. **Database Abstraction** - Consistent interface for database operations
2. **Data Integrity** - Validation, constraints, and transaction management
3. **Performance Optimization** - Query optimization and caching
4. **Security** - SQL injection prevention and data sanitization
5. **Extensibility** - Event dispatching and plugin support

### **Key Benefits:**
- **Consistency** - Standardized data access patterns
- **Security** - Built-in protection against common vulnerabilities
- **Performance** - Optimized query building and execution
- **Maintainability** - Clean separation of concerns
- **Scalability** - Efficient handling of large datasets

### **Essential Patterns:**
- Use proper validation in `_beforeSave()` hooks
- Implement transaction management for complex operations
- Design for performance with proper indexing
- Follow security best practices with parameter binding
- Use bulk operations for better performance
- Implement proper error handling and logging

### **Best Practices Summary:**
1. **Always validate data** before persistence
2. **Use transactions** for multi-step operations
3. **Implement proper error handling** with meaningful messages
4. **Optimize queries** with proper indexes and query structure
5. **Use bulk operations** for better performance
6. **Follow security practices** to prevent SQL injection
7. **Test thoroughly** with both unit and integration tests

Understanding Resource Models deeply enables you to build robust, secure, and performant data access layers that can handle enterprise-scale requirements while maintaining data integrity and optimal performance in Magento 2 applications.