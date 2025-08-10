# The Complete Guide to Magento 2 Models

## Table of Contents
1. [Learning Objectives](#learning-objectives)
2. [Introduction to Models](#introduction-to-models)
3. [Model Architecture Overview](#model-architecture-overview)
4. [AbstractModel Deep Dive](#abstractmodel-deep-dive)
5. [Model Lifecycle and Hooks](#model-lifecycle-and-hooks)
6. [Data Validation Framework](#data-validation-framework)
7. [Change Tracking System](#change-tracking-system)
8. [Caching Integration](#caching-integration)
9. [Resource Models](#resource-models)
10. [Collection Models](#collection-models)
11. [Model Events and Observers](#model-events-and-observers)
12. [Custom Model Implementation](#custom-model-implementation)
13. [Advanced Model Patterns](#advanced-model-patterns)
14. [Performance Optimization](#performance-optimization)
15. [Testing Models](#testing-models)
16. [Common Patterns and Anti-Patterns](#common-patterns-and-anti-patterns)
17. [Troubleshooting Guide](#troubleshooting-guide)
18. [Hands-On Exercises](#hands-on-exercises)
19. [Summary and Best Practices](#summary-and-best-practices)

## Learning Objectives

By the end of this guide, you will be able to:

- Understand the Model-View-Controller (MVC) pattern in Magento 2
- Master AbstractModel and its lifecycle hooks
- Implement custom models with proper validation
- Use change tracking for optimized database operations
- Integrate caching mechanisms effectively
- Create and work with Resource Models and Collections
- Handle model events and implement observers
- Apply advanced patterns for complex business logic
- Optimize model performance for large datasets
- Test models thoroughly and debug common issues
- Follow best practices for maintainable code

## Introduction to Models

Models in Magento 2 represent the business logic and data layer of the application. They are responsible for:

- **Data Management**: Storing, retrieving, and manipulating business data
- **Business Logic**: Implementing domain-specific rules and workflows
- **Database Abstraction**: Providing an object-oriented interface to database operations
- **Validation**: Ensuring data integrity and business rule compliance
- **Event Dispatching**: Enabling extensibility through event-driven architecture

### Model Types in Magento 2

**Entity Models**: Represent business entities (Product, Customer, Order)
- Extend AbstractModel
- Have corresponding Resource Models
- Support collections for bulk operations

**Resource Models**: Handle database operations
- Extend AbstractResource
- Manage CRUD operations
- Handle database schema information

**Collection Models**: Manage sets of entities
- Extend AbstractCollection
- Provide filtering, sorting, and pagination
- Optimize database queries

**Data Models**: Simple data containers
- Implement interfaces for type safety
- Used in Service Layer contracts
- Focus on data transfer rather than behavior

## Model Architecture Overview

### Traditional MVC in Magento 2

```
Controller → Model → Resource Model → Database
     ↓         ↓           ↓
   View    Collection   Cache Layer
```

### Model Relationships

```php
<?php
// Entity Model
class Product extends \Magento\Framework\Model\AbstractModel
{
    protected function _construct()
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Product::class);
    }
}

// Resource Model
class ProductResource extends \Magento\Framework\Model\ResourceModel\Db\AbstractDb
{
    protected function _construct()
    {
        $this->_init('custom_product', 'entity_id');
    }
}

// Collection Model
class ProductCollection extends \Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection
{
    protected function _construct()
    {
        $this->_init(
            \Vendor\Module\Model\Product::class,
            \Vendor\Module\Model\ResourceModel\Product::class
        );
    }
}
```

### Dependency Injection Configuration

```xml
<!-- etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    
    <!-- Resource Model Factory -->
    <type name="Vendor\Module\Model\Product">
        <arguments>
            <argument name="resource" xsi:type="object">Vendor\Module\Model\ResourceModel\Product</argument>
            <argument name="resourceCollection" xsi:type="string">Vendor\Module\Model\ResourceModel\Product\Collection</argument>
        </arguments>
    </type>
    
</config>
```

## AbstractModel Deep Dive

AbstractModel is the foundation class for all entity models in Magento 2. It provides essential functionality for data management, validation, and lifecycle management.

### Core Features of AbstractModel

#### 1. Data Storage and Access

```php
<?php
namespace Magento\Framework\Model;

abstract class AbstractModel extends \Magento\Framework\DataObject
{
    /**
     * Model data storage
     */
    protected $_data = [];

    /**
     * Original data snapshot
     */
    protected $_origData = [];

    /**
     * Changed fields flag
     */
    protected $_hasDataChanges = false;

    /**
     * Deleted flag
     */
    protected $_isDeleted = false;

    /**
     * Get data by key
     */
    public function getData($key = '', $index = null)
    {
        if ('' === $key) {
            return $this->_data;
        }

        // Handle nested array access
        if (strpos($key, '/') !== false) {
            $data = $this->_data;
            foreach (explode('/', $key) as $step) {
                if (!isset($data[$step])) {
                    return null;
                }
                $data = $data[$step];
            }
            return $data;
        }

        if (isset($this->_data[$key])) {
            if ($index !== null) {
                if (is_array($this->_data[$key])) {
                    return isset($this->_data[$key][$index]) ? $this->_data[$key][$index] : null;
                }
                return null;
            }
            return $this->_data[$key];
        }
        return null;
    }

    /**
     * Set data with change tracking
     */
    public function setData($key, $value = null)
    {
        if (is_array($key)) {
            foreach ($key as $k => $v) {
                $this->setData($k, $v);
            }
            return $this;
        }

        // Track changes
        if (array_key_exists($key, $this->_data) && $this->_data[$key] !== $value) {
            $this->_hasDataChanges = true;
        } elseif (!array_key_exists($key, $this->_data)) {
            $this->_hasDataChanges = true;
        }

        $this->_data[$key] = $value;
        return $this;
    }
}
```

#### 2. Magic Methods for Property Access

```php
<?php
abstract class AbstractModel extends \Magento\Framework\DataObject
{
    /**
     * Magic getter for model properties
     */
    public function __call($method, $args)
    {
        switch (substr($method, 0, 3)) {
            case 'get':
                $key = $this->_underscore(substr($method, 3));
                return $this->getData($key, isset($args[0]) ? $args[0] : null);

            case 'set':
                $key = $this->_underscore(substr($method, 3));
                $value = isset($args[0]) ? $args[0] : null;
                return $this->setData($key, $value);

            case 'uns':
                $key = $this->_underscore(substr($method, 3));
                return $this->unsetData($key);

            case 'has':
                $key = $this->_underscore(substr($method, 3));
                return isset($this->_data[$key]);
        }

        throw new \Exception("Invalid method " . get_class($this) . "::" . $method);
    }

    /**
     * Convert CamelCase to snake_case
     */
    protected function _underscore($name)
    {
        return strtolower(preg_replace('/(.)([A-Z])/', "$1_$2", $name));
    }
}
```

### Example: Custom Product Model

```php
<?php
namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractModel;
use Magento\Framework\DataObject\IdentityInterface;

/**
 * Custom Product Model
 * 
 * @method int getId()
 * @method $this setId(int $id)
 * @method string getSku()
 * @method $this setSku(string $sku)
 * @method string getName()
 * @method $this setName(string $name)
 * @method float getPrice()
 * @method $this setPrice(float $price)
 * @method int getStatus()
 * @method $this setStatus(int $status)
 * @method string getCreatedAt()
 * @method $this setCreatedAt(string $createdAt)
 * @method string getUpdatedAt()
 * @method $this setUpdatedAt(string $updatedAt)
 */
class Product extends AbstractModel implements IdentityInterface
{
    /**
     * Product statuses
     */
    const STATUS_ENABLED = 1;
    const STATUS_DISABLED = 2;

    /**
     * Cache tag for this model
     */
    const CACHE_TAG = 'vendor_module_product';

    /**
     * Cache tag
     */
    protected $_cacheTag = self::CACHE_TAG;

    /**
     * Event prefix
     */
    protected $_eventPrefix = 'vendor_module_product';

    /**
     * Event object name
     */
    protected $_eventObject = 'product';

    /**
     * Model construct that should be used for object initialization
     */
    protected function _construct()
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Product::class);
    }

    /**
     * Get identity for cache
     */
    public function getIdentities()
    {
        return [self::CACHE_TAG . '_' . $this->getId()];
    }

    /**
     * Get default values for new product
     */
    public function getDefaultValues()
    {
        return [
            'status' => self::STATUS_ENABLED,
            'created_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Check if product is enabled
     */
    public function isEnabled()
    {
        return $this->getStatus() == self::STATUS_ENABLED;
    }

    /**
     * Get formatted price
     */
    public function getFormattedPrice()
    {
        return '$' . number_format($this->getPrice(), 2);
    }

    /**
     * Validate product data before save
     */
    public function validateBeforeSave()
    {
        parent::validateBeforeSave();

        // Validate SKU
        if (!$this->getSku()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('SKU is required.')
            );
        }

        // Validate name
        if (!$this->getName()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Product name is required.')
            );
        }

        // Validate price
        if ($this->getPrice() < 0) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Price cannot be negative.')
            );
        }

        return $this;
    }
}
```

## Model Lifecycle and Hooks

Models in Magento 2 have a well-defined lifecycle with hooks that allow you to inject custom logic at specific points.

### Model Lifecycle Overview

```
1. Object Creation → _construct()
2. Data Loading → beforeLoad() → load() → afterLoad()
3. Data Validation → validateBeforeSave()
4. Data Saving → beforeSave() → save() → afterSave()
5. Data Deletion → beforeDelete() → delete() → afterDelete()
```

### Lifecycle Hooks in Detail

#### 1. Construction Hook

```php
<?php
class Product extends AbstractModel
{
    /**
     * Initialize model
     * Called when object is created
     */
    protected function _construct()
    {
        // Set resource model
        $this->_init(\Vendor\Module\Model\ResourceModel\Product::class);
        
        // Set default values
        $this->setData($this->getDefaultValues());
        
        // Initialize custom properties
        $this->_initializeCustomProperties();
    }

    /**
     * Initialize custom properties
     */
    private function _initializeCustomProperties()
    {
        $this->_eventPrefix = 'vendor_module_product';
        $this->_eventObject = 'product';
        $this->_cacheTag = 'vendor_module_product';
    }
}
```

#### 2. Loading Hooks

```php
<?php
class Product extends AbstractModel
{
    /**
     * Processing object before load data
     */
    protected function _beforeLoad($id, $field = null)
    {
        // Dispatch before load event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_load_before',
            [$this->_eventObject => $this, 'field' => $field, 'value' => $id]
        );

        // Custom logic before loading
        $this->_preprocessLoadData($id, $field);

        return parent::_beforeLoad($id, $field);
    }

    /**
     * Processing object after load data
     */
    protected function _afterLoad()
    {
        // Custom logic after loading
        $this->_postprocessLoadData();

        // Dispatch after load event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_load_after',
            [$this->_eventObject => $this]
        );

        return parent::_afterLoad();
    }

    /**
     * Custom preprocessing before load
     */
    private function _preprocessLoadData($id, $field)
    {
        // Log loading activity
        $this->_logger->debug('Loading product', ['id' => $id, 'field' => $field]);

        // Validate load parameters
        if ($field && !in_array($field, $this->getAllowedLoadFields())) {
            throw new \InvalidArgumentException("Invalid load field: $field");
        }
    }

    /**
     * Custom postprocessing after load
     */
    private function _postprocessLoadData()
    {
        // Decrypt sensitive data
        if ($this->getEncryptedField()) {
            $this->setDecryptedField($this->decrypt($this->getEncryptedField()));
        }

        // Load related data
        $this->_loadRelatedData();

        // Set original data snapshot
        $this->_origData = $this->_data;
    }

    /**
     * Load related data
     */
    private function _loadRelatedData()
    {
        // Load categories, images, etc.
        if ($this->getId()) {
            $categories = $this->getCategoryCollection()
                ->addFieldToFilter('product_id', $this->getId())
                ->load();
            $this->setData('categories', $categories->getItems());
        }
    }
}
```

#### 3. Validation Hooks

```php
<?php
class Product extends AbstractModel
{
    /**
     * Validate before save
     */
    public function validateBeforeSave()
    {
        // Call parent validation
        parent::validateBeforeSave();

        // Dispatch validation event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_validate_before',
            [$this->_eventObject => $this]
        );

        // Required field validation
        $this->_validateRequiredFields();

        // Business rule validation
        $this->_validateBusinessRules();

        // Custom validation
        $this->_validateCustomRules();

        return $this;
    }

    /**
     * Validate required fields
     */
    private function _validateRequiredFields()
    {
        $requiredFields = ['sku', 'name', 'price', 'status'];
        
        foreach ($requiredFields as $field) {
            if (!$this->getData($field) && $this->getData($field) !== '0' && $this->getData($field) !== 0) {
                throw new \Magento\Framework\Exception\LocalizedException(
                    __('Field "%1" is required.', $field)
                );
            }
        }
    }

    /**
     * Validate business rules
     */
    private function _validateBusinessRules()
    {
        // Price validation
        if ($this->getPrice() < 0) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Price cannot be negative.')
            );
        }

        // SKU uniqueness (simplified - would use resource model)
        if ($this->_isDuplicateSku()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('SKU "%1" already exists.', $this->getSku())
            );
        }

        // Status validation
        if (!in_array($this->getStatus(), [self::STATUS_ENABLED, self::STATUS_DISABLED])) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Invalid status value.')
            );
        }
    }

    /**
     * Custom validation rules
     */
    private function _validateCustomRules()
    {
        // Custom business logic validation
        if ($this->getType() === 'subscription' && !$this->getSubscriptionPeriod()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Subscription period is required for subscription products.')
            );
        }
    }
}
```

#### 4. Save Hooks

```php
<?php
class Product extends AbstractModel
{
    /**
     * Processing object before save data
     */
    public function beforeSave()
    {
        // Dispatch before save event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_save_before',
            [$this->_eventObject => $this]
        );

        // Set timestamps
        $this->_updateTimestamps();

        // Process data transformations
        $this->_processDataTransformations();

        // Validate data
        $this->validateBeforeSave();

        return parent::beforeSave();
    }

    /**
     * Processing object after save data
     */
    public function afterSave()
    {
        // Call parent after save
        parent::afterSave();

        // Save related data
        $this->_saveRelatedData();

        // Update search index
        $this->_updateSearchIndex();

        // Clear cache
        $this->_clearRelatedCache();

        // Dispatch after save event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_save_after',
            [$this->_eventObject => $this]
        );

        return $this;
    }

    /**
     * Update timestamps
     */
    private function _updateTimestamps()
    {
        $now = date('Y-m-d H:i:s');
        
        if (!$this->getId()) {
            // New record
            $this->setCreatedAt($now);
        }
        
        $this->setUpdatedAt($now);
    }

    /**
     * Process data transformations
     */
    private function _processDataTransformations()
    {
        // Normalize SKU
        if ($this->getSku()) {
            $this->setSku(strtoupper(trim($this->getSku())));
        }

        // Format price
        if ($this->getPrice()) {
            $this->setPrice(round($this->getPrice(), 2));
        }

        // Encrypt sensitive data
        if ($this->getSensitiveData()) {
            $this->setEncryptedData($this->encrypt($this->getSensitiveData()));
            $this->unsSensitiveData(); // Remove plain text
        }
    }

    /**
     * Save related data
     */
    private function _saveRelatedData()
    {
        // Save categories
        if ($this->hasData('category_ids')) {
            $this->_saveCategoryRelations();
        }

        // Save custom attributes
        if ($this->hasData('custom_attributes')) {
            $this->_saveCustomAttributes();
        }
    }

    /**
     * Update search index
     */
    private function _updateSearchIndex()
    {
        if ($this->isEnabled()) {
            $this->_indexer->reindexRow($this->getId());
        } else {
            $this->_indexer->deleteRow($this->getId());
        }
    }
}
```

#### 5. Delete Hooks

```php
<?php
class Product extends AbstractModel
{
    /**
     * Processing object before delete data
     */
    public function beforeDelete()
    {
        // Dispatch before delete event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_delete_before',
            [$this->_eventObject => $this]
        );

        // Check if deletion is allowed
        $this->_validateDeletion();

        // Backup data for rollback if needed
        $this->_backupData();

        return parent::beforeDelete();
    }

    /**
     * Processing object after delete data
     */
    public function afterDelete()
    {
        // Call parent after delete
        parent::afterDelete();

        // Delete related data
        $this->_deleteRelatedData();

        // Update search index
        $this->_removeFromSearchIndex();

        // Clear cache
        $this->_clearAllRelatedCache();

        // Dispatch after delete event
        $this->_eventManager->dispatch(
            $this->_eventPrefix . '_delete_after',
            [$this->_eventObject => $this]
        );

        return $this;
    }

    /**
     * Validate if deletion is allowed
     */
    private function _validateDeletion()
    {
        // Check if product is used in orders
        if ($this->_hasActiveOrders()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Cannot delete product that has been ordered.')
            );
        }

        // Check business rules
        if ($this->getType() === 'system' && !$this->_allowSystemProductDeletion()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('System products cannot be deleted.')
            );
        }
    }

    /**
     * Delete related data
     */
    private function _deleteRelatedData()
    {
        // Delete category relations
        $this->_deleteCategoryRelations();

        // Delete custom attributes
        $this->_deleteCustomAttributes();

        // Delete images
        $this->_deleteProductImages();
    }
}
```

## Data Validation Framework

Magento 2 models include a comprehensive validation framework that ensures data integrity at multiple levels.

### Built-in Validation Methods

#### 1. Basic Field Validation

```php
<?php
class Product extends AbstractModel
{
    /**
     * Validation rules array
     */
    protected $_validationRules = [
        'sku' => [
            'required' => true,
            'max_length' => 64,
            'pattern' => '/^[A-Z0-9_-]+$/',
            'unique' => true
        ],
        'name' => [
            'required' => true,
            'max_length' => 255,
            'min_length' => 2
        ],
        'price' => [
            'required' => true,
            'type' => 'decimal',
            'min' => 0,
            'max' => 99999.99
        ],
        'weight' => [
            'type' => 'decimal',
            'min' => 0
        ],
        'status' => [
            'required' => true,
            'in' => [self::STATUS_ENABLED, self::STATUS_DISABLED]
        ]
    ];

    /**
     * Validate model data
     */
    public function validate()
    {
        $errors = [];

        foreach ($this->_validationRules as $field => $rules) {
            $value = $this->getData($field);
            $fieldErrors = $this->_validateField($field, $value, $rules);
            
            if (!empty($fieldErrors)) {
                $errors[$field] = $fieldErrors;
            }
        }

        return empty($errors) ? true : $errors;
    }

    /**
     * Validate individual field
     */
    private function _validateField($field, $value, array $rules)
    {
        $errors = [];

        // Required validation
        if (isset($rules['required']) && $rules['required']) {
            if ($this->_isEmpty($value)) {
                $errors[] = __('Field "%1" is required.', $field);
            }
        }

        // Skip other validations if field is empty and not required
        if ($this->_isEmpty($value) && !isset($rules['required'])) {
            return $errors;
        }

        // Type validation
        if (isset($rules['type'])) {
            if (!$this->_validateType($value, $rules['type'])) {
                $errors[] = __('Field "%1" must be of type %2.', $field, $rules['type']);
            }
        }

        // Length validation
        if (isset($rules['max_length'])) {
            if (strlen($value) > $rules['max_length']) {
                $errors[] = __('Field "%1" cannot exceed %2 characters.', $field, $rules['max_length']);
            }
        }

        if (isset($rules['min_length'])) {
            if (strlen($value) < $rules['min_length']) {
                $errors[] = __('Field "%1" must be at least %2 characters.', $field, $rules['min_length']);
            }
        }

        // Range validation
        if (isset($rules['min'])) {
            if ($value < $rules['min']) {
                $errors[] = __('Field "%1" must be at least %2.', $field, $rules['min']);
            }
        }

        if (isset($rules['max'])) {
            if ($value > $rules['max']) {
                $errors[] = __('Field "%1" cannot exceed %2.', $field, $rules['max']);
            }
        }

        // Pattern validation
        if (isset($rules['pattern'])) {
            if (!preg_match($rules['pattern'], $value)) {
                $errors[] = __('Field "%1" format is invalid.', $field);
            }
        }

        // In array validation
        if (isset($rules['in'])) {
            if (!in_array($value, $rules['in'])) {
                $errors[] = __('Field "%1" must be one of: %2.', $field, implode(', ', $rules['in']));
            }
        }

        // Unique validation
        if (isset($rules['unique']) && $rules['unique']) {
            if (!$this->_validateUnique($field, $value)) {
                $errors[] = __('Field "%1" must be unique.', $field);
            }
        }

        return $errors;
    }

    /**
     * Check if value is empty
     */
    private function _isEmpty($value)
    {
        return $value === null || $value === '' || $value === [];
    }

    /**
     * Validate data type
     */
    private function _validateType($value, $type)
    {
        switch ($type) {
            case 'string':
                return is_string($value);
            case 'integer':
                return is_int($value) || (is_string($value) && ctype_digit($value));
            case 'decimal':
                return is_numeric($value);
            case 'boolean':
                return is_bool($value) || in_array($value, [0, 1, '0', '1']);
            case 'date':
                return $this->_validateDate($value);
            case 'email':
                return filter_var($value, FILTER_VALIDATE_EMAIL) !== false;
            case 'url':
                return filter_var($value, FILTER_VALIDATE_URL) !== false;
            default:
                return true;
        }
    }

    /**
     * Validate date format
     */
    private function _validateDate($value)
    {
        return (bool) \DateTime::createFromFormat('Y-m-d H:i:s', $value);
    }

    /**
     * Validate field uniqueness
     */
    private function _validateUnique($field, $value)
    {
        $collection = $this->getCollection()
            ->addFieldToFilter($field, $value);

        if ($this->getId()) {
            $collection->addFieldToFilter('entity_id', ['neq' => $this->getId()]);
        }

        return $collection->getSize() === 0;
    }
}
```

#### 2. Custom Validator Classes

```php
<?php
namespace Vendor\Module\Model\Validator;

class ProductValidator
{
    private $storeManager;
    private $productRepository;

    public function __construct(
        \Magento\Store\Model\StoreManagerInterface $storeManager,
        \Vendor\Module\Api\ProductRepositoryInterface $productRepository
    ) {
        $this->storeManager = $storeManager;
        $this->productRepository = $productRepository;
    }

    /**
     * Validate product business rules
     */
    public function validate(\Vendor\Module\Model\Product $product)
    {
        $errors = [];

        // SKU validation
        $skuErrors = $this->validateSku($product);
        if (!empty($skuErrors)) {
            $errors['sku'] = $skuErrors;
        }

        // Price validation
        $priceErrors = $this->validatePrice($product);
        if (!empty($priceErrors)) {
            $errors['price'] = $priceErrors;
        }

        // Category validation
        $categoryErrors = $this->validateCategories($product);
        if (!empty($categoryErrors)) {
            $errors['categories'] = $categoryErrors;
        }

        // Store-specific validation
        $storeErrors = $this->validateStoreSpecificData($product);
        if (!empty($storeErrors)) {
            $errors = array_merge($errors, $storeErrors);
        }

        return $errors;
    }

    /**
     * Validate SKU
     */
    private function validateSku(\Vendor\Module\Model\Product $product)
    {
        $errors = [];
        $sku = $product->getSku();

        // Format validation
        if (!preg_match('/^[A-Z0-9_-]+$/', $sku)) {
            $errors[] = __('SKU can only contain uppercase letters, numbers, underscores, and hyphens.');
        }

        // Length validation
        if (strlen($sku) < 3) {
            $errors[] = __('SKU must be at least 3 characters long.');
        }

        if (strlen($sku) > 64) {
            $errors[] = __('SKU cannot exceed 64 characters.');
        }

        // Uniqueness validation
        try {
            $existingProduct = $this->productRepository->getBySku($sku);
            if ($existingProduct->getId() !== $product->getId()) {
                $errors[] = __('SKU "%1" already exists.', $sku);
            }
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            // SKU is unique, no error
        }

        return $errors;
    }

    /**
     * Validate price
     */
    private function validatePrice(\Vendor\Module\Model\Product $product)
    {
        $errors = [];
        $price = $product->getPrice();

        // Minimum price validation
        if ($price < 0.01) {
            $errors[] = __('Price must be at least $0.01.');
        }

        // Maximum price validation
        if ($price > 99999.99) {
            $errors[] = __('Price cannot exceed $99,999.99.');
        }

        // Business rule: Luxury tax validation
        if ($price > 10000 && !$product->hasLuxuryTaxExemption()) {
            $product->setData('requires_luxury_tax', true);
        }

        return $errors;
    }

    /**
     * Validate categories
     */
    private function validateCategories(\Vendor\Module\Model\Product $product)
    {
        $errors = [];
        $categoryIds = $product->getCategoryIds();

        if (empty($categoryIds)) {
            $errors[] = __('Product must be assigned to at least one category.');
        }

        // Validate category exists and is active
        foreach ($categoryIds as $categoryId) {
            if (!$this->_isCategoryValid($categoryId)) {
                $errors[] = __('Category ID %1 is invalid or inactive.', $categoryId);
            }
        }

        return $errors;
    }

    /**
     * Validate store-specific data
     */
    private function validateStoreSpecificData(\Vendor\Module\Model\Product $product)
    {
        $errors = [];

        foreach ($this->storeManager->getStores() as $store) {
            $storeId = $store->getId();
            
            // Validate store-specific pricing
            $storePrice = $product->getData("price_store_{$storeId}");
            if ($storePrice !== null && $storePrice < 0) {
                $errors["price_store_{$storeId}"] = [
                    __('Store price for %1 cannot be negative.', $store->getName())
                ];
            }

            // Validate store-specific availability
            $storeStatus = $product->getData("status_store_{$storeId}");
            if ($storeStatus !== null && !in_array($storeStatus, [0, 1])) {
                $errors["status_store_{$storeId}"] = [
                    __('Invalid status for store %1.', $store->getName())
                ];
            }
        }

        return $errors;
    }
}
```

#### 3. Validation Integration

```php
<?php
class Product extends AbstractModel
{
    private $validator;

    public function __construct(
        \Magento\Framework\Model\Context $context,
        \Magento\Framework\Registry $registry,
        \Vendor\Module\Model\Validator\ProductValidator $validator,
        \Magento\Framework\Model\ResourceModel\AbstractResource $resource = null,
        \Magento\Framework\Data\Collection\AbstractDb $resourceCollection = null,
        array $data = []
    ) {
        $this->validator = $validator;
        parent::__construct($context, $registry, $resource, $resourceCollection, $data);
    }

    /**
     * Enhanced validation with custom validator
     */
    public function validateBeforeSave()
    {
        // Basic validation first
        parent::validateBeforeSave();

        // Built-in model validation
        $modelErrors = $this->validate();
        if ($modelErrors !== true) {
            $this->_throwValidationException($modelErrors);
        }

        // Custom business validation
        $businessErrors = $this->validator->validate($this);
        if (!empty($businessErrors)) {
            $this->_throwValidationException($businessErrors);
        }

        return $this;
    }

    /**
     * Throw validation exception with formatted errors
     */
    private function _throwValidationException(array $errors)
    {
        $messages = [];
        foreach ($errors as $field => $fieldErrors) {
            foreach ($fieldErrors as $error) {
                $messages[] = $error;
            }
        }

        throw new \Magento\Framework\Exception\LocalizedException(
            __('Validation failed: %1', implode('; ', $messages))
        );
    }

    /**
     * Get validation errors without throwing exception
     */
    public function getValidationErrors()
    {
        $errors = [];

        // Model validation errors
        $modelErrors = $this->validate();
        if ($modelErrors !== true) {
            $errors = array_merge($errors, $modelErrors);
        }

        // Business validation errors
        $businessErrors = $this->validator->validate($this);
        $errors = array_merge($errors, $businessErrors);

        return $errors;
    }

    /**
     * Check if model is valid
     */
    public function isValid()
    {
        return empty($this->getValidationErrors());
    }
}
```

## Change Tracking System

Magento 2 models include sophisticated change tracking that optimizes database operations and enables audit trails.

### Built-in Change Tracking

#### 1. Basic Change Detection

```php
<?php
abstract class AbstractModel extends \Magento\Framework\DataObject
{
    /**
     * Original data before changes
     */
    protected $_origData = [];

    /**
     * Flag indicating if object has data changes
     */
    protected $_hasDataChanges = false;

    /**
     * Check if model has changes
     */
    public function hasDataChanges()
    {
        return $this->_hasDataChanges;
    }

    /**
     * Get original data
     */
    public function getOrigData($key = null)
    {
        if ($key === null) {
            return $this->_origData;
        }
        return isset($this->_origData[$key]) ? $this->_origData[$key] : null;
    }

    /**
     * Get data changes
     */
    public function getDataChanges()
    {
        $changes = [];
        
        foreach ($this->_data as $key => $value) {
            $origValue = isset($this->_origData[$key]) ? $this->_origData[$key] : null;
            
            if ($origValue !== $value) {
                $changes[$key] = [
                    'from' => $origValue,
                    'to' => $value
                ];
            }
        }

        return $changes;
    }

    /**
     * Check if specific field has changed
     */
    public function dataHasChangedFor($field)
    {
        return array_key_exists($field, $this->_data) && 
               (!array_key_exists($field, $this->_origData) || 
                $this->_data[$field] !== $this->_origData[$field]);
    }

    /**
     * Set original data snapshot
     */
    public function setOrigData($key = null, $data = null)
    {
        if ($key === null) {
            $this->_origData = $this->_data;
        } else {
            $this->_origData[$key] = $data;
        }
        return $this;
    }
}
```

#### 2. Advanced Change Tracking

```php
<?php
class Product extends AbstractModel
{
    /**
     * Fields that should trigger specific actions when changed
     */
    protected $_changeTrackedFields = [
        'status' => 'onStatusChange',
        'price' => 'onPriceChange',
        'name' => 'onNameChange',
        'sku' => 'onSkuChange'
    ];

    /**
     * Fields that should not be tracked for changes
     */
    protected $_excludeFromChangeTracking = [
        'updated_at',
        'view_count',
        'last_accessed'
    ];

    /**
     * Enhanced data setter with detailed change tracking
     */
    public function setData($key, $value = null)
    {
        if (is_array($key)) {
            foreach ($key as $k => $v) {
                $this->setData($k, $v);
            }
            return $this;
        }

        // Skip change tracking for excluded fields
        if (in_array($key, $this->_excludeFromChangeTracking)) {
            $this->_data[$key] = $value;
            return $this;
        }

        // Track changes
        $hasChanged = $this->_trackFieldChange($key, $value);

        // Call field-specific change handlers
        if ($hasChanged && isset($this->_changeTrackedFields[$key])) {
            $method = $this->_changeTrackedFields[$key];
            if (method_exists($this, $method)) {
                $this->$method($this->getOrigData($key), $value);
            }
        }

        return parent::setData($key, $value);
    }

    /**
     * Track field change with detailed logging
     */
    private function _trackFieldChange($field, $newValue)
    {
        $oldValue = $this->getData($field);
        
        if ($oldValue === $newValue) {
            return false; // No change
        }

        // Log the change
        $this->_logFieldChange($field, $oldValue, $newValue);

        // Add to change log
        $this->_addToChangeLog($field, $oldValue, $newValue);

        return true;
    }

    /**
     * Log field changes
     */
    private function _logFieldChange($field, $oldValue, $newValue)
    {
        $this->_logger->info('Product field changed', [
            'product_id' => $this->getId(),
            'field' => $field,
            'old_value' => $oldValue,
            'new_value' => $newValue,
            'user_id' => $this->_getCurrentUserId(),
            'timestamp' => time()
        ]);
    }

    /**
     * Add change to internal change log
     */
    private function _addToChangeLog($field, $oldValue, $newValue)
    {
        if (!isset($this->_data['_change_log'])) {
            $this->_data['_change_log'] = [];
        }

        $this->_data['_change_log'][] = [
            'field' => $field,
            'old_value' => $oldValue,
            'new_value' => $newValue,
            'timestamp' => microtime(true),
            'user_id' => $this->_getCurrentUserId()
        ];
    }

    /**
     * Get change log
     */
    public function getChangeLog()
    {
        return $this->getData('_change_log') ?: [];
    }

    /**
     * Get changes for specific field
     */
    public function getFieldChanges($field)
    {
        $changes = [];
        $changeLog = $this->getChangeLog();

        foreach ($changeLog as $change) {
            if ($change['field'] === $field) {
                $changes[] = $change;
            }
        }

        return $changes;
    }

    /**
     * Handle status change
     */
    protected function onStatusChange($oldStatus, $newStatus)
    {
        // Clear cache when status changes
        if ($oldStatus != $newStatus) {
            $this->_clearProductCache();
        }

        // Update search index
        if ($newStatus == self::STATUS_DISABLED) {
            $this->setData('_should_remove_from_index', true);
        } elseif ($oldStatus == self::STATUS_DISABLED) {
            $this->setData('_should_add_to_index', true);
        }

        // Notify administrators of status changes
        if ($this->getId()) {
            $this->_notifyStatusChange($oldStatus, $newStatus);
        }
    }

    /**
     * Handle price change
     */
    protected function onPriceChange($oldPrice, $newPrice)
    {
        // Calculate price change percentage
        if ($oldPrice > 0) {
            $changePercent = (($newPrice - $oldPrice) / $oldPrice) * 100;
            $this->setData('_price_change_percent', $changePercent);

            // Flag significant price changes
            if (abs($changePercent) > 20) {
                $this->setData('_significant_price_change', true);
            }
        }

        // Update related product prices if necessary
        $this->_updateRelatedProductPrices();

        // Clear price cache
        $this->_clearPriceCache();
    }

    /**
     * Handle name change
     */
    protected function onNameChange($oldName, $newName)
    {
        // Update URL key if not manually set
        if (!$this->dataHasChangedFor('url_key')) {
            $urlKey = $this->_generateUrlKey($newName);
            $this->setData('url_key', $urlKey);
        }

        // Update search keywords
        $this->_updateSearchKeywords();
    }

    /**
     * Handle SKU change
     */
    protected function onSkuChange($oldSku, $newSku)
    {
        // Validate SKU change is allowed
        if ($this->getId() && !$this->_isSkuChangeAllowed()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('SKU cannot be changed for existing products with orders.')
            );
        }

        // Update related data
        $this->_updateSkuRelatedData($oldSku, $newSku);
    }

    /**
     * Get summary of all changes
     */
    public function getChangesSummary()
    {
        $changes = $this->getDataChanges();
        $summary = [
            'total_changes' => count($changes),
            'significant_changes' => 0,
            'fields_changed' => array_keys($changes),
            'change_details' => []
        ];

        $significantFields = ['sku', 'name', 'price', 'status'];

        foreach ($changes as $field => $change) {
            if (in_array($field, $significantFields)) {
                $summary['significant_changes']++;
            }

            $summary['change_details'][$field] = [
                'old' => $change['from'],
                'new' => $change['to'],
                'significant' => in_array($field, $significantFields)
            ];
        }

        return $summary;
    }
}
```

#### 3. Change Tracking for Audit Trail

```php
<?php
namespace Vendor\Module\Model;

class ProductChangeTracker
{
    private $auditRepository;
    private $userContext;
    private $dateTime;

    public function __construct(
        \Vendor\Module\Api\AuditRepositoryInterface $auditRepository,
        \Magento\Authorization\Model\UserContextInterface $userContext,
        \Magento\Framework\Stdlib\DateTime\DateTime $dateTime
    ) {
        $this->auditRepository = $auditRepository;
        $this->userContext = $userContext;
        $this->dateTime = $dateTime;
    }

    /**
     * Track product changes for audit
     */
    public function trackChanges(\Vendor\Module\Model\Product $product)
    {
        if (!$product->hasDataChanges()) {
            return;
        }

        $changes = $product->getDataChanges();
        
        foreach ($changes as $field => $change) {
            $this->_createAuditRecord($product, $field, $change);
        }
    }

    /**
     * Create audit record
     */
    private function _createAuditRecord($product, $field, $change)
    {
        $auditData = [
            'entity_type' => 'product',
            'entity_id' => $product->getId(),
            'field_name' => $field,
            'old_value' => $this->_serializeValue($change['from']),
            'new_value' => $this->_serializeValue($change['to']),
            'user_id' => $this->userContext->getUserId(),
            'user_type' => $this->userContext->getUserType(),
            'ip_address' => $this->_getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'action' => $product->isObjectNew() ? 'create' : 'update',
            'timestamp' => $this->dateTime->gmtDate()
        ];

        $audit = $this->auditRepository->create($auditData);
        $this->auditRepository->save($audit);
    }

    /**
     * Serialize value for storage
     */
    private function _serializeValue($value)
    {
        if (is_array($value) || is_object($value)) {
            return json_encode($value);
        }
        return (string) $value;
    }

    /**
     * Get client IP address
     */
    private function _getClientIp()
    {
        $ipKeys = ['HTTP_X_FORWARDED_FOR', 'HTTP_X_REAL_IP', 'HTTP_CLIENT_IP', 'REMOTE_ADDR'];
        
        foreach ($ipKeys as $key) {
            if (!empty($_SERVER[$key])) {
                $ips = explode(',', $_SERVER[$key]);
                return trim($ips[0]);
            }
        }
        
        return 'unknown';
    }

    /**
     * Get change history for product
     */
    public function getChangeHistory($productId, $limit = 50)
    {
        return $this->auditRepository->getByEntityId('product', $productId, $limit);
    }

    /**
     * Get changes by user
     */
    public function getChangesByUser($userId, $dateFrom = null, $dateTo = null)
    {
        return $this->auditRepository->getByUser($userId, $dateFrom, $dateTo);
    }
}
```

## Caching Integration

Models in Magento 2 integrate seamlessly with the caching system to improve performance and reduce database load.

### Model-Level Caching

#### 1. Basic Cache Integration

```php
<?php
class Product extends AbstractModel implements IdentityInterface
{
    /**
     * Cache tag for this model
     */
    const CACHE_TAG = 'vendor_module_product';

    /**
     * Cache lifetime (seconds)
     */
    const CACHE_LIFETIME = 3600;

    /**
     * Cache tag
     */
    protected $_cacheTag = self::CACHE_TAG;

    /**
     * Cache instance
     */
    private $cache;

    /**
     * Serializer for cache data
     */
    private $serializer;

    public function __construct(
        \Magento\Framework\Model\Context $context,
        \Magento\Framework\Registry $registry,
        \Magento\Framework\App\CacheInterface $cache,
        \Magento\Framework\Serialize\SerializerInterface $serializer,
        \Magento\Framework\Model\ResourceModel\AbstractResource $resource = null,
        \Magento\Framework\Data\Collection\AbstractDb $resourceCollection = null,
        array $data = []
    ) {
        $this->cache = $cache;
        $this->serializer = $serializer;
        parent::__construct($context, $registry, $resource, $resourceCollection, $data);
    }

    /**
     * Get cache identities for this model
     */
    public function getIdentities()
    {
        $identities = [self::CACHE_TAG];
        
        if ($this->getId()) {
            $identities[] = self::CACHE_TAG . '_' . $this->getId();
        }
        
        if ($this->getSku()) {
            $identities[] = self::CACHE_TAG . '_sku_' . $this->getSku();
        }

        // Add category-based cache tags
        $categoryIds = $this->getCategoryIds();
        if ($categoryIds) {
            foreach ($categoryIds as $categoryId) {
                $identities[] = 'category_products_' . $categoryId;
            }
        }

        return $identities;
    }

    /**
     * Load model with caching
     */
    public function load($id, $field = null)
    {
        if ($field === null) {
            $field = $this->getIdFieldName();
        }

        // Generate cache key
        $cacheKey = $this->_getCacheKey($field, $id);

        // Try to load from cache
        $cachedData = $this->cache->load($cacheKey);
        if ($cachedData !== false) {
            $data = $this->serializer->unserialize($cachedData);
            $this->setData($data);
            $this->setOrigData();
            $this->_afterLoad();
            return $this;
        }

        // Load from database
        parent::load($id, $field);

        // Cache the loaded data
        if ($this->getId()) {
            $this->_cacheModelData($cacheKey);
        }

        return $this;
    }

    /**
     * Save model with cache invalidation
     */
    public function save()
    {
        // Get old identities before saving
        $oldIdentities = $this->getIdentities();

        // Save the model
        $result = parent::save();

        // Get new identities after saving
        $newIdentities = $this->getIdentities();

        // Invalidate relevant cache
        $identities = array_unique(array_merge($oldIdentities, $newIdentities));
        $this->_invalidateCache($identities);

        // Cache the updated model
        $cacheKey = $this->_getCacheKey($this->getIdFieldName(), $this->getId());
        $this->_cacheModelData($cacheKey);

        return $result;
    }

    /**
     * Delete model with cache invalidation
     */
    public function delete()
    {
        $identities = $this->getIdentities();
        
        $result = parent::delete();
        
        // Invalidate cache
        $this->_invalidateCache($identities);
        
        return $result;
    }

    /**
     * Generate cache key
     */
    private function _getCacheKey($field, $value)
    {
        return sprintf(
            '%s_%s_%s',
            self::CACHE_TAG,
            $field,
            md5($value)
        );
    }

    /**
     * Cache model data
     */
    private function _cacheModelData($cacheKey)
    {
        $data = $this->getData();
        
        // Remove sensitive data from cache
        unset($data['password'], $data['token'], $data['api_key']);
        
        $this->cache->save(
            $this->serializer->serialize($data),
            $cacheKey,
            $this->getIdentities(),
            self::CACHE_LIFETIME
        );
    }

    /**
     * Invalidate cache by identities
     */
    private function _invalidateCache(array $identities)
    {
        foreach ($identities as $identity) {
            $this->cache->clean([\Zend_Cache::CLEANING_MODE_MATCHING_TAG], [$identity]);
        }
    }
}
```

#### 2. Advanced Caching Strategies

```php
<?php
class Product extends AbstractModel
{
    /**
     * Cache for expensive calculations
     */
    private $_calculationCache = [];

    /**
     * Cache configuration
     */
    private $_cacheConfig = [
        'related_products' => ['lifetime' => 1800, 'tag' => 'related_products'],
        'category_path' => ['lifetime' => 3600, 'tag' => 'category_data'],
        'price_data' => ['lifetime' => 900, 'tag' => 'price_data'],
        'stock_data' => ['lifetime' => 300, 'tag' => 'stock_data']
    ];

    /**
     * Get related products with caching
     */
    public function getRelatedProducts()
    {
        $cacheKey = 'related_products_' . $this->getId();
        
        return $this->_getCachedData($cacheKey, function() {
            return $this->_loadRelatedProducts();
        }, 'related_products');
    }

    /**
     * Get category path with caching
     */
    public function getCategoryPath()
    {
        $cacheKey = 'category_path_' . $this->getId();
        
        return $this->_getCachedData($cacheKey, function() {
            return $this->_buildCategoryPath();
        }, 'category_path');
    }

    /**
     * Get price data with caching
     */
    public function getPriceData()
    {
        $cacheKey = 'price_data_' . $this->getId();
        
        return $this->_getCachedData($cacheKey, function() {
            return $this->_calculatePriceData();
        }, 'price_data');
    }

    /**
     * Generic cached data getter
     */
    private function _getCachedData($cacheKey, callable $dataProvider, $configKey)
    {
        // Check memory cache first
        if (isset($this->_calculationCache[$cacheKey])) {
            return $this->_calculationCache[$cacheKey];
        }

        // Check persistent cache
        $cachedData = $this->cache->load($cacheKey);
        if ($cachedData !== false) {
            $data = $this->serializer->unserialize($cachedData);
            $this->_calculationCache[$cacheKey] = $data;
            return $data;
        }

        // Calculate data
        $data = $dataProvider();

        // Cache in memory
        $this->_calculationCache[$cacheKey] = $data;

        // Cache persistently
        $config = $this->_cacheConfig[$configKey];
        $tags = array_merge($this->getIdentities(), [$config['tag']]);
        
        $this->cache->save(
            $this->serializer->serialize($data),
            $cacheKey,
            $tags,
            $config['lifetime']
        );

        return $data;
    }

    /**
     * Clear specific cache
     */
    public function clearCache($type = null)
    {
        if ($type === null) {
            // Clear all cache for this model
            $this->_invalidateCache($this->getIdentities());
            $this->_calculationCache = [];
        } else {
            // Clear specific cache type
            if (isset($this->_cacheConfig[$type])) {
                $tag = $this->_cacheConfig[$type]['tag'];
                $this->cache->clean([\Zend_Cache::CLEANING_MODE_MATCHING_TAG], [$tag]);
            }
            
            // Clear from memory cache
            foreach ($this->_calculationCache as $key => $value) {
                if (strpos($key, $type) !== false) {
                    unset($this->_calculationCache[$key]);
                }
            }
        }
    }

    /**
     * Warm up cache for commonly accessed data
     */
    public function warmUpCache()
    {
        // Pre-load commonly accessed data
        $this->getRelatedProducts();
        $this->getCategoryPath();
        $this->getPriceData();
        
        // Pre-calculate expensive operations
        $this->_preCalculateData();
    }

    /**
     * Pre-calculate data that might be needed
     */
    private function _preCalculateData()
    {
        // Calculate and cache search keywords
        $keywords = $this->_generateSearchKeywords();
        $cacheKey = 'search_keywords_' . $this->getId();
        $this->cache->save(
            $this->serializer->serialize($keywords),
            $cacheKey,
            $this->getIdentities(),
            3600
        );

        // Calculate and cache related SKUs
        $relatedSkus = $this->_getRelatedSkus();
        $cacheKey = 'related_skus_' . $this->getId();
        $this->cache->save(
            $this->serializer->serialize($relatedSkus),
            $cacheKey,
            $this->getIdentities(),
            1800
        );
    }
}
```

#### 3. Cache Tags and Invalidation

```php
<?php
class ProductCacheManager
{
    private $cache;
    private $eventManager;

    public function __construct(
        \Magento\Framework\App\CacheInterface $cache,
        \Magento\Framework\Event\ManagerInterface $eventManager
    ) {
        $this->cache = $cache;
        $this->eventManager = $eventManager;
    }

    /**
     * Invalidate product cache
     */
    public function invalidateProduct($productId)
    {
        $tags = [
            Product::CACHE_TAG . '_' . $productId,
            'product_data',
            'category_products'
        ];

        $this->cache->clean([\Zend_Cache::CLEANING_MODE_MATCHING_TAG], $tags);
        
        // Dispatch event for additional cache invalidation
        $this->eventManager->dispatch('product_cache_invalidate', [
            'product_id' => $productId,
            'cache_tags' => $tags
        ]);
    }

    /**
     * Invalidate category-related cache
     */
    public function invalidateCategory($categoryId)
    {
        $tags = [
            'category_' . $categoryId,
            'category_products_' . $categoryId,
            'navigation_menu',
            'catalog_category'
        ];

        $this->cache->clean([\Zend_Cache::CLEANING_MODE_MATCHING_TAG], $tags);
    }

    /**
     * Invalidate price-related cache
     */
    public function invalidatePriceData()
    {
        $tags = [
            'price_data',
            'catalog_product_price',
            'tier_price',
            'special_price'
        ];

        $this->cache->clean([\Zend_Cache::CLEANING_MODE_MATCHING_TAG], $tags);
    }

    /**
     * Batch invalidate multiple products
     */
    public function batchInvalidateProducts(array $productIds)
    {
        $tags = ['product_data', 'category_products'];
        
        foreach ($productIds as $productId) {
            $tags[] = Product::CACHE_TAG . '_' . $productId;
        }

        $this->cache->clean([\Zend_Cache::CLEANING_MODE_MATCHING_TAG], $tags);
    }

    /**
     * Schedule cache warm-up
     */
    public function scheduleWarmUp($productId)
    {
        // Add to warm-up queue
        $this->eventManager->dispatch('cache_warmup_schedule', [
            'entity_type' => 'product',
            'entity_id' => $productId
        ]);
    }
}
```

## Resource Models

Resource Models handle all database operations for entity models. They provide an abstraction layer between models and the database.

### Basic Resource Model Implementation

```php
<?php
namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;
use Magento\Framework\Model\ResourceModel\Db\Context;
use Magento\Framework\Stdlib\DateTime\DateTime;
use Magento\Framework\Model\AbstractModel;

class Product extends AbstractDb
{
    /**
     * Date model
     */
    private $date;

    /**
     * Constructor
     */
    public function __construct(
        Context $context,
        DateTime $date,
        $connectionName = null
    ) {
        $this->date = $date;
        parent::__construct($context, $connectionName);
    }

    /**
     * Initialize resource model
     */
    protected function _construct()
    {
        // Table name and primary key
        $this->_init('vendor_module_product', 'entity_id');
    }

    /**
     * Process model data before save
     */
    protected function _beforeSave(AbstractModel $object)
    {
        // Set timestamps
        if (!$object->getId()) {
            $object->setCreatedAt($this->date->gmtDate());
        }
        $object->setUpdatedAt($this->date->gmtDate());

        // Validate unique fields
        $this->_validateUniqueFields($object);

        return parent::_beforeSave($object);
    }

    /**
     * Process model data after save
     */
    protected function _afterSave(AbstractModel $object)
    {
        // Save related data
        $this->_saveRelatedData($object);

        return parent::_afterSave($object);
    }

    /**
     * Process model data after load
     */
    protected function _afterLoad(AbstractModel $object)
    {
        // Load related data
        $this->_loadRelatedData($object);

        return parent::_afterLoad($object);
    }

    /**
     * Process model data before delete
     */
    protected function _beforeDelete(AbstractModel $object)
    {
        // Check if deletion is allowed
        $this->_validateDeletion($object);

        return parent::_beforeDelete($object);
    }

    /**
     * Process model data after delete
     */
    protected function _afterDelete(AbstractModel $object)
    {
        // Delete related data
        $this->_deleteRelatedData($object);

        return parent::_afterDelete($object);
    }

    /**
     * Validate unique fields
     */
    private function _validateUniqueFields(AbstractModel $object)
    {
        // Check SKU uniqueness
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable(), 'entity_id')
            ->where('sku = ?', $object->getSku());

        if ($object->getId()) {
            $select->where('entity_id != ?', $object->getId());
        }

        $duplicateId = $connection->fetchOne($select);
        if ($duplicateId) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('A product with SKU "%1" already exists.', $object->getSku())
            );
        }
    }

    /**
     * Save related data
     */
    private function _saveRelatedData(AbstractModel $object)
    {
        // Save category relations
        if ($object->hasData('category_ids')) {
            $this->_saveCategoryRelations($object);
        }

        // Save custom attributes
        if ($object->hasData('custom_attributes')) {
            $this->_saveCustomAttributes($object);
        }
    }

    /**
     * Save category relations
     */
    private function _saveCategoryRelations(AbstractModel $object)
    {
        $connection = $this->getConnection();
        $categoryTable = $this->getTable('vendor_module_product_category');

        // Delete existing relations
        $connection->delete($categoryTable, ['product_id = ?' => $object->getId()]);

        // Insert new relations
        $categoryIds = $object->getData('category_ids');
        if (!empty($categoryIds)) {
            $data = [];
            foreach ($categoryIds as $categoryId) {
                $data[] = [
                    'product_id' => $object->getId(),
                    'category_id' => $categoryId
                ];
            }
            $connection->insertMultiple($categoryTable, $data);
        }
    }

    /**
     * Load related data
     */
    private function _loadRelatedData(AbstractModel $object)
    {
        if ($object->getId()) {
            // Load category IDs
            $categoryIds = $this->_getCategoryIds($object->getId());
            $object->setData('category_ids', $categoryIds);
        }
    }

    /**
     * Get category IDs for product
     */
    private function _getCategoryIds($productId)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getTable('vendor_module_product_category'), 'category_id')
            ->where('product_id = ?', $productId);

        return $connection->fetchCol($select);
    }

    /**
     * Delete related data
     */
    private function _deleteRelatedData(AbstractModel $object)
    {
        $connection = $this->getConnection();

        // Delete category relations
        $connection->delete(
            $this->getTable('vendor_module_product_category'),
            ['product_id = ?' => $object->getId()]
        );

        // Delete custom attributes
        $connection->delete(
            $this->getTable('vendor_module_product_attributes'),
            ['product_id = ?' => $object->getId()]
        );
    }

    /**
     * Load model by SKU
     */
    public function loadBySku(AbstractModel $object, $sku)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable())
            ->where('sku = ?', $sku);

        $data = $connection->fetchRow($select);
        if ($data) {
            $object->setData($data);
            $this->_afterLoad($object);
        }

        return $this;
    }

    /**
     * Check if SKU exists
     */
    public function isSkuExists($sku, $excludeId = null)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable(), 'entity_id')
            ->where('sku = ?', $sku);

        if ($excludeId) {
            $select->where('entity_id != ?', $excludeId);
        }

        return (bool) $connection->fetchOne($select);
    }

    /**
     * Get products by category
     */
    public function getProductsByCategory($categoryId)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from(['p' => $this->getMainTable()])
            ->join(
                ['pc' => $this->getTable('vendor_module_product_category')],
                'p.entity_id = pc.product_id',
                []
            )
            ->where('pc.category_id = ?', $categoryId)
            ->where('p.status = ?', \Vendor\Module\Model\Product::STATUS_ENABLED);

        return $connection->fetchAll($select);
    }

    /**
     * Update product status in bulk
     */
    public function updateStatus(array $productIds, $status)
    {
        $connection = $this->getConnection();
        $connection->update(
            $this->getMainTable(),
            ['status' => $status, 'updated_at' => $this->date->gmtDate()],
            ['entity_id IN (?)' => $productIds]
        );
    }

    /**
     * Get product statistics
     */
    public function getStatistics()
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable(), [
                'total_products' => 'COUNT(*)',
                'enabled_products' => 'SUM(CASE WHEN status = 1 THEN 1 ELSE 0 END)',
                'disabled_products' => 'SUM(CASE WHEN status = 2 THEN 1 ELSE 0 END)',
                'avg_price' => 'AVG(price)',
                'min_price' => 'MIN(price)',
                'max_price' => 'MAX(price)'
            ]);

        return $connection->fetchRow($select);
    }
}
```

### Advanced Resource Model Features

```php
<?php
namespace Vendor\Module\Model\ResourceModel;

class AdvancedProduct extends Product
{
    /**
     * Custom select modifiers
     */
    private $selectModifiers = [];

    /**
     * Add custom select modifier
     */
    public function addSelectModifier($name, callable $modifier)
    {
        $this->selectModifiers[$name] = $modifier;
        return $this;
    }

    /**
     * Get connection with read/write separation
     */
    public function getConnection($type = 'read')
    {
        if ($type === 'write') {
            return $this->_resources->getConnection('core_write');
        }
        return $this->_resources->getConnection('core_read');
    }

    /**
     * Advanced product search
     */
    public function searchProducts($searchCriteria)
    {
        $connection = $this->getConnection('read');
        $select = $connection->select()
            ->from(['main_table' => $this->getMainTable()]);

        // Apply search filters
        $this->_applySearchFilters($select, $searchCriteria);

        // Apply custom modifiers
        foreach ($this->selectModifiers as $modifier) {
            $modifier($select, $searchCriteria);
        }

        // Apply sorting
        $this->_applySorting($select, $searchCriteria);

        // Apply pagination
        $this->_applyPagination($select, $searchCriteria);

        return $connection->fetchAll($select);
    }

    /**
     * Apply search filters
     */
    private function _applySearchFilters($select, $searchCriteria)
    {
        // Text search
        if ($searchCriteria->getSearchTerm()) {
            $searchTerm = '%' . $searchCriteria->getSearchTerm() . '%';
            $select->where(
                'main_table.name LIKE ? OR main_table.sku LIKE ? OR main_table.description LIKE ?',
                $searchTerm
            );
        }

        // Category filter
        if ($searchCriteria->getCategoryIds()) {
            $select->join(
                ['pc' => $this->getTable('vendor_module_product_category')],
                'main_table.entity_id = pc.product_id',
                []
            )->where('pc.category_id IN (?)', $searchCriteria->getCategoryIds());
        }

        // Price range filter
        if ($searchCriteria->getMinPrice()) {
            $select->where('main_table.price >= ?', $searchCriteria->getMinPrice());
        }
        if ($searchCriteria->getMaxPrice()) {
            $select->where('main_table.price <= ?', $searchCriteria->getMaxPrice());
        }

        // Status filter
        if ($searchCriteria->getStatus()) {
            $select->where('main_table.status = ?', $searchCriteria->getStatus());
        }

        // Date range filter
        if ($searchCriteria->getCreatedFrom()) {
            $select->where('main_table.created_at >= ?', $searchCriteria->getCreatedFrom());
        }
        if ($searchCriteria->getCreatedTo()) {
            $select->where('main_table.created_at <= ?', $searchCriteria->getCreatedTo());
        }
    }

    /**
     * Apply sorting
     */
    private function _applySorting($select, $searchCriteria)
    {
        $sortBy = $searchCriteria->getSortBy() ?: 'entity_id';
        $sortDir = $searchCriteria->getSortDirection() ?: 'ASC';

        // Handle special sort cases
        switch ($sortBy) {
            case 'relevance':
                if ($searchCriteria->getSearchTerm()) {
                    $this->_addRelevanceSort($select, $searchCriteria->getSearchTerm());
                }
                break;
            case 'popularity':
                $this->_addPopularitySort($select);
                break;
            case 'newest':
                $select->order('main_table.created_at DESC');
                break;
            default:
                $select->order("main_table.{$sortBy} {$sortDir}");
        }
    }

    /**
     * Add relevance sorting for search
     */
    private function _addRelevanceSort($select, $searchTerm)
    {
        $relevanceScore = sprintf(
            '(CASE 
                WHEN main_table.name LIKE %s THEN 3
                WHEN main_table.sku LIKE %s THEN 2
                WHEN main_table.description LIKE %s THEN 1
                ELSE 0
            END)',
            $this->getConnection()->quote('%' . $searchTerm . '%'),
            $this->getConnection()->quote('%' . $searchTerm . '%'),
            $this->getConnection()->quote('%' . $searchTerm . '%')
        );

        $select->columns(['relevance' => new \Zend_Db_Expr($relevanceScore)])
               ->order('relevance DESC');
    }

    /**
     * Add popularity sorting
     */
    private function _addPopularitySort($select)
    {
        $select->joinLeft(
            ['stats' => $this->getTable('vendor_module_product_stats')],
            'main_table.entity_id = stats.product_id',
            ['popularity' => 'COALESCE(stats.view_count, 0) + COALESCE(stats.order_count, 0) * 10']
        )->order('popularity DESC');
    }

    /**
     * Apply pagination
     */
    private function _applyPagination($select, $searchCriteria)
    {
        $pageSize = $searchCriteria->getPageSize() ?: 20;
        $currentPage = $searchCriteria->getCurrentPage() ?: 1;
        
        $select->limitPage($currentPage, $pageSize);
    }

    /**
     * Bulk operations
     */
    public function bulkUpdate(array $productIds, array $data)
    {
        $connection = $this->getConnection('write');
        
        $data['updated_at'] = $this->date->gmtDate();
        
        return $connection->update(
            $this->getMainTable(),
            $data,
            ['entity_id IN (?)' => $productIds]
        );
    }

    /**
     * Batch insert
     */
    public function batchInsert(array $products)
    {
        $connection = $this->getConnection('write');
        
        $data = [];
        foreach ($products as $product) {
            $productData = $product;
            $productData['created_at'] = $this->date->gmtDate();
            $productData['updated_at'] = $this->date->gmtDate();
            $data[] = $productData;
        }
        
        return $connection->insertMultiple($this->getMainTable(), $data);
    }

    /**
     * Execute raw query with logging
     */
    public function executeQuery($sql, array $bind = [])
    {
        $connection = $this->getConnection('write');
        
        // Log query for debugging
        $this->_logger->debug('Executing custom query', [
            'sql' => $sql,
            'bind' => $bind
        ]);
        
        return $connection->query($sql, $bind);
    }

    /**
     * Get table statistics
     */
    public function getTableStats()
    {
        $connection = $this->getConnection('read');
        $select = $connection->select()
            ->from('INFORMATION_SCHEMA.TABLES', [
                'table_rows' => 'TABLE_ROWS',
                'data_length' => 'DATA_LENGTH',
                'index_length' => 'INDEX_LENGTH',
                'auto_increment' => 'AUTO_INCREMENT'
            ])
            ->where('TABLE_SCHEMA = ?', $connection->getConfig()['dbname'])
            ->where('TABLE_NAME = ?', $this->getMainTable());

        return $connection->fetchRow($select);
    }
}
```

## Collection Models

Collection Models provide powerful tools for working with sets of entities, including filtering, sorting, pagination, and aggregation.

### Basic Collection Implementation

```php
<?php
namespace Vendor\Module\Model\ResourceModel\Product;

use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;

class Collection extends AbstractCollection
{
    /**
     * ID field name
     */
    protected $_idFieldName = 'entity_id';

    /**
     * Event prefix
     */
    protected $_eventPrefix = 'vendor_module_product_collection';

    /**
     * Event object
     */
    protected $_eventObject = 'product_collection';

    /**
     * Define resource model
     */
    protected function _construct()
    {
        $this->_init(
            \Vendor\Module\Model\Product::class,
            \Vendor\Module\Model\ResourceModel\Product::class
        );
    }

    /**
     * Initialize collection
     */
    protected function _initSelect()
    {
        parent::_initSelect();

        // Add default filters
        $this->addFieldToFilter('status', \Vendor\Module\Model\Product::STATUS_ENABLED);

        return $this;
    }

    /**
     * Add enabled filter
     */
    public function addEnabledFilter()
    {
        return $this->addFieldToFilter('status', \Vendor\Module\Model\Product::STATUS_ENABLED);
    }

    /**
     * Add disabled filter
     */
    public function addDisabledFilter()
    {
        return $this->addFieldToFilter('status', \Vendor\Module\Model\Product::STATUS_DISABLED);
    }

    /**
     * Add SKU filter
     */
    public function addSkuFilter($sku)
    {
        if (is_array($sku)) {
            return $this->addFieldToFilter('sku', ['in' => $sku]);
        }
        return $this->addFieldToFilter('sku', $sku);
    }

    /**
     * Add category filter
     */
    public function addCategoryFilter($categoryId)
    {
        $this->getSelect()->join(
            ['cat' => $this->getTable('vendor_module_product_category')],
            'main_table.entity_id = cat.product_id',
            []
        )->where('cat.category_id = ?', $categoryId);

        return $this;
    }

    /**
     * Add price range filter
     */
    public function addPriceFilter($minPrice = null, $maxPrice = null)
    {
        if ($minPrice !== null) {
            $this->addFieldToFilter('price', ['gteq' => $minPrice]);
        }
        
        if ($maxPrice !== null) {
            $this->addFieldToFilter('price', ['lteq' => $maxPrice]);
        }

        return $this;
    }

    /**
     * Add search filter
     */
    public function addSearchFilter($searchTerm)
    {
        $searchTerm = '%' . $searchTerm . '%';
        $this->getSelect()->where(
            'main_table.name LIKE ? OR main_table.sku LIKE ? OR main_table.description LIKE ?',
            $searchTerm
        );

        return $this;
    }

    /**
     * Add date range filter
     */
    public function addDateRangeFilter($fromDate = null, $toDate = null)
    {
        if ($fromDate) {
            $this->addFieldToFilter('created_at', ['gteq' => $fromDate]);
        }
        
        if ($toDate) {
            $this->addFieldToFilter('created_at', ['lteq' => $toDate]);
        }

        return $this;
    }

    /**
     * Add order by popularity
     */
    public function addPopularityOrder()
    {
        $this->getSelect()->joinLeft(
            ['stats' => $this->getTable('vendor_module_product_stats')],
            'main_table.entity_id = stats.product_id',
            []
        )->order('COALESCE(stats.view_count, 0) + COALESCE(stats.order_count, 0) * 10 DESC');

        return $this;
    }

    /**
     * Add order by newest
     */
    public function addNewestOrder()
    {
        return $this->setOrder('created_at', 'DESC');
    }

    /**
     * Get products grouped by category
     */
    public function getGroupedByCategory()
    {
        $this->getSelect()->join(
            ['cat' => $this->getTable('vendor_module_product_category')],
            'main_table.entity_id = cat.product_id',
            ['category_id']
        )->join(
            ['category' => $this->getTable('vendor_module_category')],
            'cat.category_id = category.entity_id',
            ['category_name' => 'name']
        )->order('category.name ASC');

        $grouped = [];
        foreach ($this as $product) {
            $categoryName = $product->getData('category_name');
            if (!isset($grouped[$categoryName])) {
                $grouped[$categoryName] = [];
            }
            $grouped[$categoryName][] = $product;
        }

        return $grouped;
    }

    /**
     * Get products with statistics
     */
    public function addStatistics()
    {
        $this->getSelect()->joinLeft(
            ['stats' => $this->getTable('vendor_module_product_stats')],
            'main_table.entity_id = stats.product_id',
            [
                'view_count' => 'COALESCE(stats.view_count, 0)',
                'order_count' => 'COALESCE(stats.order_count, 0)',
                'rating' => 'COALESCE(stats.avg_rating, 0)'
            ]
        );

        return $this;
    }

    /**
     * Get collection size with cache
     */
    public function getSize()
    {
        if ($this->_totalRecords === null) {
            $this->_totalRecords = parent::getSize();
        }
        return $this->_totalRecords;
    }

    /**
     * Get items with eager loading
     */
    public function getItems()
    {
        $items = parent::getItems();
        
        // Eager load related data
        $this->_eagerLoadRelatedData($items);
        
        return $items;
    }

    /**
     * Eager load related data for performance
     */
    private function _eagerLoadRelatedData($items)
    {
        if (empty($items)) {
            return;
        }

        $productIds = array_keys($items);

        // Load categories for all products
        $categories = $this->_loadCategoriesForProducts($productIds);
        
        // Load custom attributes
        $customAttributes = $this->_loadCustomAttributesForProducts($productIds);

        // Assign to products
        foreach ($items as $productId => $product) {
            if (isset($categories[$productId])) {
                $product->setData('category_ids', $categories[$productId]);
            }
            
            if (isset($customAttributes[$productId])) {
                $product->setData('custom_attributes', $customAttributes[$productId]);
            }
        }
    }

    /**
     * Load categories for multiple products
     */
    private function _loadCategoriesForProducts($productIds)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getTable('vendor_module_product_category'))
            ->where('product_id IN (?)', $productIds);

        $result = $connection->fetchAll($select);
        
        $categories = [];
        foreach ($result as $row) {
            $categories[$row['product_id']][] = $row['category_id'];
        }

        return $categories;
    }

    /**
     * Clone collection
     */
    public function cloneCollection()
    {
        $cloned = clone $this;
        $cloned->_reset();
        return $cloned;
    }
}
```

### Advanced Collection Features

```php
<?php
namespace Vendor\Module\Model\ResourceModel\Product;

class AdvancedCollection extends Collection
{
    /**
     * Aggregation cache
     */
    private $_aggregationCache = [];

    /**
     * Custom filters
     */
    private $_customFilters = [];

    /**
     * Add custom filter
     */
    public function addCustomFilter($name, callable $filter)
    {
        $this->_customFilters[$name] = $filter;
        return $this;
    }

    /**
     * Apply custom filter
     */
    public function applyCustomFilter($name, ...$args)
    {
        if (isset($this->_customFilters[$name])) {
            $this->_customFilters[$name]($this, ...$args);
        }
        return $this;
    }

    /**
     * Add faceted search capabilities
     */
    public function addFacets(array $fields)
    {
        $facets = [];
        
        foreach ($fields as $field) {
            $facets[$field] = $this->_getFacetData($field);
        }
        
        $this->setData('facets', $facets);
        return $this;
    }

    /**
     * Get facet data for field
     */
    private function _getFacetData($field)
    {
        $connection = $this->getConnection();
        $select = clone $this->getSelect();
        
        // Reset select parts that might interfere
        $select->reset(\Zend_Db_Select::COLUMNS)
               ->reset(\Zend_Db_Select::LIMIT_COUNT)
               ->reset(\Zend_Db_Select::LIMIT_OFFSET)
               ->reset(\Zend_Db_Select::ORDER);

        $select->columns([
            'value' => "main_table.{$field}",
            'count' => 'COUNT(*)'
        ])->group("main_table.{$field}");

        return $connection->fetchPairs($select);
    }

    /**
     * Add aggregation functions
     */
    public function getAggregation()
    {
        $cacheKey = 'aggregation_' . md5($this->getSelect()->__toString());
        
        if (!isset($this->_aggregationCache[$cacheKey])) {
            $connection = $this->getConnection();
            $select = clone $this->getSelect();
            
            $select->reset(\Zend_Db_Select::COLUMNS)
                   ->reset(\Zend_Db_Select::LIMIT_COUNT)
                   ->reset(\Zend_Db_Select::LIMIT_OFFSET)
                   ->reset(\Zend_Db_Select::ORDER);

            $select->columns([
                'total_count' => 'COUNT(*)',
                'avg_price' => 'AVG(main_table.price)',
                'min_price' => 'MIN(main_table.price)',
                'max_price' => 'MAX(main_table.price)',
                'sum_price' => 'SUM(main_table.price)',
                'enabled_count' => 'SUM(CASE WHEN main_table.status = 1 THEN 1 ELSE 0 END)',
                'disabled_count' => 'SUM(CASE WHEN main_table.status = 2 THEN 1 ELSE 0 END)'
            ]);

            $this->_aggregationCache[$cacheKey] = $connection->fetchRow($select);
        }

        return $this->_aggregationCache[$cacheKey];
    }

    /**
     * Add price ranges
     */
    public function getPriceRanges($numberOfRanges = 5)
    {
        $aggregation = $this->getAggregation();
        $minPrice = $aggregation['min_price'];
        $maxPrice = $aggregation['max_price'];
        
        if ($minPrice === null || $maxPrice === null) {
            return [];
        }

        $rangeSize = ($maxPrice - $minPrice) / $numberOfRanges;
        $ranges = [];

        for ($i = 0; $i < $numberOfRanges; $i++) {
            $rangeMin = $minPrice + ($i * $rangeSize);
            $rangeMax = $minPrice + (($i + 1) * $rangeSize);
            
            if ($i === $numberOfRanges - 1) {
                $rangeMax = $maxPrice; // Ensure last range includes max price
            }

            $count = $this->_getPriceRangeCount($rangeMin, $rangeMax);
            
            $ranges[] = [
                'min' => round($rangeMin, 2),
                'max' => round($rangeMax, 2),
                'count' => $count,
                'label' => '$' . round($rangeMin, 2) . ' - $' . round($rangeMax, 2)
            ];
        }

        return $ranges;
    }

    /**
     * Get count for price range
     */
    private function _getPriceRangeCount($minPrice, $maxPrice)
    {
        $connection = $this->getConnection();
        $select = clone $this->getSelect();
        
        $select->reset(\Zend_Db_Select::COLUMNS)
               ->reset(\Zend_Db_Select::LIMIT_COUNT)
               ->reset(\Zend_Db_Select::LIMIT_OFFSET)
               ->reset(\Zend_Db_Select::ORDER);

        $select->columns(['count' => 'COUNT(*)'])
               ->where('main_table.price >= ?', $minPrice)
               ->where('main_table.price <= ?', $maxPrice);

        return (int) $connection->fetchOne($select);
    }

    /**
     * Add random order
     */
    public function addRandomOrder()
    {
        $this->getSelect()->order('RAND()');
        return $this;
    }

    /**
     * Add weighted random order (based on popularity)
     */
    public function addWeightedRandomOrder()
    {
        $this->getSelect()->joinLeft(
            ['stats' => $this->getTable('vendor_module_product_stats')],
            'main_table.entity_id = stats.product_id',
            []
        )->order('RAND() * (COALESCE(stats.view_count, 1) + COALESCE(stats.order_count, 1) * 10) DESC');

        return $this;
    }

    /**
     * Get products by similarity to given product
     */
    public function addSimilarityFilter($productId, $limit = 10)
    {
        // Get the target product's categories and attributes
        $targetProduct = $this->_productRepository->getById($productId);
        $targetCategories = $targetProduct->getCategoryIds();
        $targetPrice = $targetProduct->getPrice();

        // Calculate similarity score
        $priceRange = $targetPrice * 0.2; // 20% price range
        
        $this->getSelect()->joinLeft(
            ['cat' => $this->getTable('vendor_module_product_category')],
            'main_table.entity_id = cat.product_id',
            []
        )->where('main_table.entity_id != ?', $productId)
         ->where('cat.category_id IN (?)', $targetCategories)
         ->where('main_table.price BETWEEN ? AND ?', 
             $targetPrice - $priceRange, 
             $targetPrice + $priceRange)
         ->group('main_table.entity_id')
         ->order('COUNT(cat.category_id) DESC')
         ->limit($limit);

        return $this;
    }

    /**
     * Export collection to CSV
     */
    public function exportToCsv($filename = null)
    {
        if (!$filename) {
            $filename = 'products_' . date('Y-m-d_H-i-s') . '.csv';
        }

        $handle = fopen($filename, 'w');
        
        // Write headers
        $headers = ['ID', 'SKU', 'Name', 'Price', 'Status', 'Created At'];
        fputcsv($handle, $headers);

        // Write data
        foreach ($this as $product) {
            $row = [
                $product->getId(),
                $product->getSku(),
                $product->getName(),
                $product->getPrice(),
                $product->getStatus(),
                $product->getCreatedAt()
            ];
            fputcsv($handle, $row);
        }

        fclose($handle);
        return $filename;
    }

    /**
     * Convert collection to array with pagination info
     */
    public function toArray($fields = [])
    {
        $result = [
            'items' => [],
            'total_count' => $this->getSize(),
            'page_size' => $this->getPageSize(),
            'current_page' => $this->getCurPage()
        ];

        foreach ($this as $item) {
            if (empty($fields)) {
                $result['items'][] = $item->getData();
            } else {
                $itemData = [];
                foreach ($fields as $field) {
                    $itemData[$field] = $item->getData($field);
                }
                $result['items'][] = $itemData;
            }
        }

        return $result;
    }

    /**
     * Apply search criteria from search criteria interface
     */
    public function applySearchCriteria($searchCriteria)
    {
        // Apply filters
        foreach ($searchCriteria->getFilterGroups() as $filterGroup) {
            $filters = $filterGroup->getFilters();
            $conditions = [];
            
            foreach ($filters as $filter) {
                $condition = $filter->getConditionType() ?: 'eq';
                $field = $filter->getField();
                $value = $filter->getValue();
                
                switch ($condition) {
                    case 'eq':
                        $conditions[] = ['eq' => $value];
                        break;
                    case 'neq':
                        $conditions[] = ['neq' => $value];
                        break;
                    case 'like':
                        $conditions[] = ['like' => '%' . $value . '%'];
                        break;
                    case 'in':
                        $conditions[] = ['in' => $value];
                        break;
                    case 'nin':
                        $conditions[] = ['nin' => $value];
                        break;
                    case 'gt':
                        $conditions[] = ['gt' => $value];
                        break;
                    case 'lt':
                        $conditions[] = ['lt' => $value];
                        break;
                    case 'gteq':
                        $conditions[] = ['gteq' => $value];
                        break;
                    case 'lteq':
                        $conditions[] = ['lteq' => $value];
                        break;
                }
            }
            
            if (!empty($conditions)) {
                $this->addFieldToFilter($field, $conditions);
            }
        }

        // Apply sorting
        $sortOrders = $searchCriteria->getSortOrders();
        if ($sortOrders) {
            foreach ($sortOrders as $sortOrder) {
                $this->addOrder(
                    $sortOrder->getField(),
                    $sortOrder->getDirection()
                );
            }
        }

        // Apply pagination
        $this->setCurPage($searchCriteria->getCurrentPage());
        $this->setPageSize($searchCriteria->getPageSize());

        return $this;
    }
}
```

## Model Events and Observers

Magento 2 models dispatch events throughout their lifecycle, enabling loose coupling and extensibility through the observer pattern.

### Model Events

#### 1. Built-in Model Events

```php
<?php
class Product extends AbstractModel
{
    /**
     * Model events are automatically dispatched
     * Event naming pattern: {event_prefix}_{action}_{when}
     */
    
    // Before/after load events
    // vendor_module_product_load_before
    // vendor_module_product_load_after
    
    // Before/after save events  
    // vendor_module_product_save_before
    // vendor_module_product_save_after
    
    // Before/after delete events
    // vendor_module_product_delete_before
    // vendor_module_product_delete_after
    
    /**
     * Dispatch custom events
     */
    public function processSpecialAction($data)
    {
        // Dispatch before event
        $this->_eventManager->dispatch(
            'vendor_module_product_special_action_before',
            [
                'product' => $this,
                'data' => $data
            ]
        );

        // Perform action
        $result = $this->_performSpecialAction($data);

        // Dispatch after event
        $this->_eventManager->dispatch(
            'vendor_module_product_special_action_after',
            [
                'product' => $this,
                'data' => $data,
                'result' => $result
            ]
        );

        return $result;
    }

    /**
     * Enhanced save with custom events
     */
    public function save()
    {
        $isNew = $this->isObjectNew();
        $changes = $this->getDataChanges();

        // Dispatch specific events based on changes
        if ($this->dataHasChangedFor('status')) {
            $this->_eventManager->dispatch(
                'vendor_module_product_status_change',
                [
                    'product' => $this,
                    'old_status' => $this->getOrigData('status'),
                    'new_status' => $this->getData('status')
                ]
            );
        }

        if ($this->dataHasChangedFor('price')) {
            $this->_eventManager->dispatch(
                'vendor_module_product_price_change',
                [
                    'product' => $this,
                    'old_price' => $this->getOrigData('price'),
                    'new_price' => $this->getData('price')
                ]
            );
        }

        // Call parent save
        $result = parent::save();

        // Dispatch completion event
        $this->_eventManager->dispatch(
            'vendor_module_product_save_complete',
            [
                'product' => $this,
                'is_new' => $isNew,
                'changes' => $changes
            ]
        );

        return $result;
    }
}
```

#### 2. Event Observer Implementation

```php
<?php
namespace Vendor\Module\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;

class ProductSaveObserver implements ObserverInterface
{
    private $logger;
    private $cacheManager;
    private $indexer;

    public function __construct(
        \Psr\Log\LoggerInterface $logger,
        \Magento\Framework\App\Cache\Manager $cacheManager,
        \Magento\Indexer\Model\IndexerInterface $indexer
    ) {
        $this->logger = $logger;
        $this->cacheManager = $cacheManager;
        $this->indexer = $indexer;
    }

    /**
     * Handle product save after event
     */
    public function execute(Observer $observer)
    {
        /** @var \Vendor\Module\Model\Product $product */
        $product = $observer->getData('product');
        
        try {
            // Log the save operation
            $this->_logProductSave($product);
            
            // Update search index
            $this->_updateSearchIndex($product);
            
            // Clear related cache
            $this->_clearRelatedCache($product);
            
            // Send notifications if needed
            $this->_sendNotifications($product);
            
        } catch (\Exception $e) {
            $this->logger->error('Error in ProductSaveObserver: ' . $e->getMessage(), [
                'product_id' => $product->getId(),
                'exception' => $e
            ]);
        }
    }

    /**
     * Log product save operation
     */
    private function _logProductSave($product)
    {
        $this->logger->info('Product saved', [
            'product_id' => $product->getId(),
            'sku' => $product->getSku(),
            'name' => $product->getName(),
            'is_new' => $product->isObjectNew(),
            'changes' => $product->getDataChanges()
        ]);
    }

    /**
     * Update search index
     */
    private function _updateSearchIndex($product)
    {
        if ($product->isEnabled()) {
            $this->indexer->reindexRow($product->getId());
        } else {
            $this->indexer->deleteRow($product->getId());
        }
    }

    /**
     * Clear related cache
     */
    private function _clearRelatedCache($product)
    {
        $tags = $product->getIdentities();
        $this->cacheManager->clean($tags);
    }

    /**
     * Send notifications
     */
    private function _sendNotifications($product)
    {
        // Send notifications for significant changes
        if ($product->dataHasChangedFor('status') || $product->dataHasChangedFor('price')) {
            // Implementation would depend on notification system
        }
    }
}
```

#### 3. Observer Configuration

```xml
<!-- etc/events.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    
    <!-- Model lifecycle events -->
    <event name="vendor_module_product_save_after">
        <observer name="product_save_logger" instance="Vendor\Module\Observer\ProductSaveObserver" />
        <observer name="product_save_cache" instance="Vendor\Module\Observer\ProductCacheObserver" />
        <observer name="product_save_index" instance="Vendor\Module\Observer\ProductIndexObserver" />
    </event>
    
    <event name="vendor_module_product_delete_after">
        <observer name="product_delete_cleanup" instance="Vendor\Module\Observer\ProductDeleteObserver" />
    </event>
    
    <!-- Custom business events -->
    <event name="vendor_module_product_status_change">
        <observer name="product_status_notification" instance="Vendor\Module\Observer\ProductStatusObserver" />
    </event>
    
    <event name="vendor_module_product_price_change">
        <observer name="product_price_alert" instance="Vendor\Module\Observer\ProductPriceObserver" />
    </event>
    
    <!-- Collection events -->
    <event name="vendor_module_product_collection_load_before">
        <observer name="collection_cache_check" instance="Vendor\Module\Observer\CollectionCacheObserver" />
    </event>
    
</config>
```

#### 4. Advanced Observer Patterns

```php
<?php
namespace Vendor\Module\Observer;

class ProductStatusObserver implements ObserverInterface
{
    private $notificationService;
    private $emailSender;
    private $stockUpdater;

    public function __construct(
        \Vendor\Module\Service\NotificationService $notificationService,
        \Vendor\Module\Service\EmailSender $emailSender,
        \Vendor\Module\Service\StockUpdater $stockUpdater
    ) {
        $this->notificationService = $notificationService;
        $this->emailSender = $emailSender;
        $this->stockUpdater = $stockUpdater;
    }

    /**
     * Handle product status changes
     */
    public function execute(Observer $observer)
    {
        $product = $observer->getData('product');
        $oldStatus = $observer->getData('old_status');
        $newStatus = $observer->getData('new_status');

        // Handle status change scenarios
        if ($this->_isStatusChangeSignificant($oldStatus, $newStatus)) {
            $this->_handleSignificantStatusChange($product, $oldStatus, $newStatus);
        }

        // Update related systems
        $this->_updateRelatedSystems($product, $newStatus);

        // Send notifications
        $this->_sendStatusChangeNotifications($product, $oldStatus, $newStatus);
    }

    /**
     * Check if status change is significant
     */
    private function _isStatusChangeSignificant($oldStatus, $newStatus)
    {
        // Enabled to disabled or vice versa
        return ($oldStatus == Product::STATUS_ENABLED && $newStatus == Product::STATUS_DISABLED) ||
               ($oldStatus == Product::STATUS_DISABLED && $newStatus == Product::STATUS_ENABLED);
    }

    /**
     * Handle significant status changes
     */
    private function _handleSignificantStatusChange($product, $oldStatus, $newStatus)
    {
        if ($newStatus == Product::STATUS_DISABLED) {
            // Product disabled
            $this->_handleProductDisabled($product);
        } elseif ($newStatus == Product::STATUS_ENABLED) {
            // Product enabled
            $this->_handleProductEnabled($product);
        }
    }

    /**
     * Handle product disabled
     */
    private function _handleProductDisabled($product)
    {
        // Remove from stock
        $this->stockUpdater->setOutOfStock($product->getId());
        
        // Remove from search index
        $this->_removeFromSearchIndex($product);
        
        // Cancel pending orders (if applicable)
        $this->_cancelPendingOrders($product);
        
        // Notify customers with wishlists
        $this->_notifyWishlistCustomers($product);
    }

    /**
     * Handle product enabled
     */
    private function _handleProductEnabled($product)
    {
        // Update stock status
        $this->stockUpdater->updateStockStatus($product->getId());
        
        // Add to search index
        $this->_addToSearchIndex($product);
        
        // Notify customers waiting for availability
        $this->_notifyWaitingCustomers($product);
    }

    /**
     * Update related systems
     */
    private function _updateRelatedSystems($product, $newStatus)
    {
        // Update recommendation engine
        $this->_updateRecommendations($product, $newStatus);
        
        // Update analytics
        $this->_updateAnalytics($product, $newStatus);
        
        // Update third-party integrations
        $this->_updateThirdPartyIntegrations($product, $newStatus);
    }

    /**
     * Send status change notifications
     */
    private function _sendStatusChangeNotifications($product, $oldStatus, $newStatus)
    {
        // Admin notifications
        $this->notificationService->sendAdminNotification(
            'Product Status Changed',
            sprintf(
                'Product "%s" (SKU: %s) status changed from %s to %s',
                $product->getName(),
                $product->getSku(),
                $this->_getStatusLabel($oldStatus),
                $this->_getStatusLabel($newStatus)
            )
        );

        // Customer notifications (if applicable)
        if ($newStatus == Product::STATUS_ENABLED) {
            $this->_sendCustomerNotifications($product);
        }
    }
}
```

## Custom Model Implementation

### Complete Custom Model Example

```php
<?php
namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractModel;
use Magento\Framework\DataObject\IdentityInterface;
use Vendor\Module\Api\Data\ReviewInterface;

/**
 * Product Review Model
 * 
 * @method int getReviewId()
 * @method $this setReviewId(int $reviewId)
 * @method int getProductId()
 * @method $this setProductId(int $productId)
 * @method int getCustomerId()
 * @method $this setCustomerId(int $customerId)
 * @method string getTitle()
 * @method $this setTitle(string $title)
 * @method string getDetail()
 * @method $this setDetail(string $detail)
 * @method string getNickname()
 * @method $this setNickname(string $nickname)
 * @method int getRating()
 * @method $this setRating(int $rating)
 * @method int getStatus()
 * @method $this setStatus(int $status)
 * @method string getCreatedAt()
 * @method $this setCreatedAt(string $createdAt)
 * @method string getUpdatedAt()
 * @method $this setUpdatedAt(string $updatedAt)
 */
class Review extends AbstractModel implements ReviewInterface, IdentityInterface
{
    /**
     * Review statuses
     */
    const STATUS_PENDING = 1;
    const STATUS_APPROVED = 2;
    const STATUS_REJECTED = 3;

    /**
     * Cache tag
     */
    const CACHE_TAG = 'vendor_module_review';

    /**
     * Cache tag
     */
    protected $_cacheTag = self::CACHE_TAG;

    /**
     * Event prefix
     */
    protected $_eventPrefix = 'vendor_module_review';

    /**
     * Event object
     */
    protected $_eventObject = 'review';

    /**
     * Validation rules
     */
    protected $_validationRules = [
        'product_id' => ['required' => true, 'type' => 'integer'],
        'title' => ['required' => true, 'max_length' => 255],
        'detail' => ['required' => true, 'max_length' => 2000],
        'rating' => ['required' => true, 'min' => 1, 'max' => 5],
        'nickname' => ['required' => true, 'max_length' => 50]
    ];

    /**
     * Dependencies
     */
    private $productRepository;
    private $customerRepository;
    private $datetime;

    /**
     * Constructor
     */
    public function __construct(
        \Magento\Framework\Model\Context $context,
        \Magento\Framework\Registry $registry,
        \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        \Magento\Customer\Api\CustomerRepositoryInterface $customerRepository,
        \Magento\Framework\Stdlib\DateTime\DateTime $datetime,
        \Magento\Framework\Model\ResourceModel\AbstractResource $resource = null,
        \Magento\Framework\Data\Collection\AbstractDb $resourceCollection = null,
        array $data = []
    ) {
        $this->productRepository = $productRepository;
        $this->customerRepository = $customerRepository;
        $this->datetime = $datetime;
        parent::__construct($context, $registry, $resource, $resourceCollection, $data);
    }

    /**
     * Initialize model
     */
    protected function _construct()
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Review::class);
    }

    /**
     * Get cache identities
     */
    public function getIdentities()
    {
        $identities = [self::CACHE_TAG];
        
        if ($this->getId()) {
            $identities[] = self::CACHE_TAG . '_' . $this->getId();
        }
        
        if ($this->getProductId()) {
            $identities[] = 'product_reviews_' . $this->getProductId();
        }
        
        if ($this->getCustomerId()) {
            $identities[] = 'customer_reviews_' . $this->getCustomerId();
        }

        return $identities;
    }

    /**
     * Before save processing
     */
    public function beforeSave()
    {
        // Set timestamps
        if (!$this->getId()) {
            $this->setCreatedAt($this->datetime->gmtDate());
            $this->setStatus(self::STATUS_PENDING);
        }
        $this->setUpdatedAt($this->datetime->gmtDate());

        // Validate data
        $this->validateBeforeSave();

        // Auto-approve for verified customers
        if ($this->_isVerifiedCustomer()) {
            $this->setStatus(self::STATUS_APPROVED);
        }

        return parent::beforeSave();
    }

    /**
     * After save processing
     */
    public function afterSave()
    {
        parent::afterSave();

        // Update product rating
        $this->_updateProductRating();

        // Send notifications
        $this->_sendNotifications();

        return $this;
    }

    /**
     * Validate model data
     */
    public function validateBeforeSave()
    {
        parent::validateBeforeSave();

        // Validate product exists
        if (!$this->_validateProductExists()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Product with ID %1 does not exist.', $this->getProductId())
            );
        }

        // Validate customer if provided
        if ($this->getCustomerId() && !$this->_validateCustomerExists()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Customer with ID %1 does not exist.', $this->getCustomerId())
            );
        }

        // Validate rating range
        if ($this->getRating() < 1 || $this->getRating() > 5) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Rating must be between 1 and 5.')
            );
        }

        // Check for duplicate reviews
        if ($this->_isDuplicateReview()) {
            throw new \Magento\Framework\Exception\LocalizedException(
                __('You have already reviewed this product.')
            );
        }

        return $this;
    }

    /**
     * Approve review
     */
    public function approve()
    {
        $this->setStatus(self::STATUS_APPROVED);
        $this->save();
        
        // Dispatch approval event
        $this->_eventManager->dispatch(
            'vendor_module_review_approved',
            ['review' => $this]
        );
        
        return $this;
    }

    /**
     * Reject review
     */
    public function reject($reason = null)
    {
        $this->setStatus(self::STATUS_REJECTED);
        if ($reason) {
            $this->setData('rejection_reason', $reason);
        }
        $this->save();
        
        // Dispatch rejection event
        $this->_eventManager->dispatch(
            'vendor_module_review_rejected',
            ['review' => $this, 'reason' => $reason]
        );
        
        return $this;
    }

    /**
     * Check if review is approved
     */
    public function isApproved()
    {
        return $this->getStatus() == self::STATUS_APPROVED;
    }

    /**
     * Check if review is pending
     */
    public function isPending()
    {
        return $this->getStatus() == self::STATUS_PENDING;
    }

    /**
     * Get review age in days
     */
    public function getAgeInDays()
    {
        $created = new \DateTime($this->getCreatedAt());
        $now = new \DateTime();
        return $now->diff($created)->days;
    }

    /**
     * Get helpfulness score (could be calculated from votes)
     */
    public function getHelpfulnessScore()
    {
        // This would typically be calculated from helpful votes
        return $this->getData('helpful_votes') ?: 0;
    }

    /**
     * Get product associated with review
     */
    public function getProduct()
    {
        if (!$this->hasData('product') && $this->getProductId()) {
            try {
                $product = $this->productRepository->getById($this->getProductId());
                $this->setData('product', $product);
            } catch (\Exception $e) {
                $this->setData('product', null);
            }
        }
        return $this->getData('product');
    }

    /**
     * Get customer associated with review
     */
    public function getCustomer()
    {
        if (!$this->hasData('customer') && $this->getCustomerId()) {
            try {
                $customer = $this->customerRepository->getById($this->getCustomerId());
                $this->setData('customer', $customer);
            } catch (\Exception $e) {
                $this->setData('customer', null);
            }
        }
        return $this->getData('customer');
    }

    /**
     * Validate product exists
     */
    private function _validateProductExists()
    {
        try {
            $this->productRepository->getById($this->getProductId());
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }

    /**
     * Validate customer exists
     */
    private function _validateCustomerExists()
    {
        try {
            $this->customerRepository->getById($this->getCustomerId());
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }

    /**
     * Check if this is a duplicate review
     */
    private function _isDuplicateReview()
    {
        if (!$this->getCustomerId()) {
            return false; // Guest reviews are allowed
        }

        $collection = $this->getCollection()
            ->addFieldToFilter('product_id', $this->getProductId())
            ->addFieldToFilter('customer_id', $this->getCustomerId());

        if ($this->getId()) {
            $collection->addFieldToFilter('review_id', ['neq' => $this->getId()]);
        }

        return $collection->getSize() > 0;
    }

    /**
     * Check if customer is verified
     */
    private function _isVerifiedCustomer()
    {
        if (!$this->getCustomerId()) {
            return false;
        }

        // Check if customer has purchased this product
        return $this->_hasCustomerPurchasedProduct();
    }

    /**
     * Check if customer has purchased the product
     */
    private function _hasCustomerPurchasedProduct()
    {
        // This would check order history
        // Implementation depends on your order/sales module structure
        return false; // Simplified for example
    }

    /**
     * Update product rating
     */
    private function _updateProductRating()
    {
        if ($this->isApproved()) {
            // This would update aggregate product rating
            // Implementation depends on your rating calculation logic
        }
    }

    /**
     * Send notifications
     */
    private function _sendNotifications()
    {
        if ($this->isApproved()) {
            // Notify customer of approval
            $this->_sendApprovalNotification();
        }
        
        // Notify admin of new review
        $this->_sendAdminNotification();
    }

    /**
     * Send approval notification
     */
    private function _sendApprovalNotification()
    {
        // Implementation would send email to customer
    }

    /**
     * Send admin notification
     */
    private function _sendAdminNotification()
    {
        // Implementation would notify admin of new review
    }
}
```

### Resource Model for Custom Model

```php
<?php
namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;
use Magento\Framework\Model\ResourceModel\Db\Context;
use Magento\Framework\Stdlib\DateTime\DateTime;
use Magento\Framework\Model\AbstractModel;

class Review extends AbstractDb
{
    /**
     * Date model
     */
    private $date;

    /**
     * Constructor
     */
    public function __construct(
        Context $context,
        DateTime $date,
        $connectionName = null
    ) {
        $this->date = $date;
        parent::__construct($context, $connectionName);
    }

    /**
     * Initialize resource model
     */
    protected function _construct()
    {
        $this->_init('vendor_module_review', 'review_id');
    }

    /**
     * Load review by product and customer
     */
    public function loadByProductAndCustomer(AbstractModel $object, $productId, $customerId)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable())
            ->where('product_id = ?', $productId)
            ->where('customer_id = ?', $customerId)
            ->order('created_at DESC')
            ->limit(1);

        $data = $connection->fetchRow($select);
        if ($data) {
            $object->setData($data);
            $this->_afterLoad($object);
        }

        return $this;
    }

    /**
     * Get reviews by status
     */
    public function getReviewsByStatus($status, $limit = null)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable())
            ->where('status = ?', $status)
            ->order('created_at DESC');

        if ($limit) {
            $select->limit($limit);
        }

        return $connection->fetchAll($select);
    }

    /**
     * Get product rating statistics
     */
    public function getProductRatingStats($productId)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable(), [
                'avg_rating' => 'AVG(rating)',
                'total_reviews' => 'COUNT(*)',
                'rating_1' => 'SUM(CASE WHEN rating = 1 THEN 1 ELSE 0 END)',
                'rating_2' => 'SUM(CASE WHEN rating = 2 THEN 1 ELSE 0 END)',
                'rating_3' => 'SUM(CASE WHEN rating = 3 THEN 1 ELSE 0 END)',
                'rating_4' => 'SUM(CASE WHEN rating = 4 THEN 1 ELSE 0 END)',
                'rating_5' => 'SUM(CASE WHEN rating = 5 THEN 1 ELSE 0 END)'
            ])
            ->where('product_id = ?', $productId)
            ->where('status = ?', \Vendor\Module\Model\Review::STATUS_APPROVED);

        return $connection->fetchRow($select);
    }

    /**
     * Update product rating summary
     */
    public function updateProductRatingSummary($productId)
    {
        $stats = $this->getProductRatingStats($productId);
        
        if ($stats['total_reviews'] > 0) {
            $connection = $this->getConnection();
            $summaryTable = $this->getTable('vendor_module_product_rating_summary');
            
            $data = [
                'product_id' => $productId,
                'avg_rating' => round($stats['avg_rating'], 2),
                'total_reviews' => $stats['total_reviews'],
                'rating_distribution' => json_encode([
                    '1' => $stats['rating_1'],
                    '2' => $stats['rating_2'],
                    '3' => $stats['rating_3'],
                    '4' => $stats['rating_4'],
                    '5' => $stats['rating_5']
                ]),
                'updated_at' => $this->date->gmtDate()
            ];
            
            $connection->insertOnDuplicate($summaryTable, $data);
        }
    }

    /**
     * After save processing
     */
    protected function _afterSave(AbstractModel $object)
    {
        parent::_afterSave($object);
        
        // Update product rating summary
        $this->updateProductRatingSummary($object->getProductId());
        
        return $this;
    }

    /**
     * After delete processing
     */
    protected function _afterDelete(AbstractModel $object)
    {
        parent::_afterDelete($object);
        
        // Update product rating summary
        $this->updateProductRatingSummary($object->getProductId());
        
        return $this;
    }
}
```

## Advanced Model Patterns

### 1. State Machine Pattern

```php
<?php
namespace Vendor\Module\Model;

class OrderStateMachine extends AbstractModel
{
    /**
     * Order states
     */
    const STATE_PENDING = 'pending';
    const STATE_PROCESSING = 'processing';
    const STATE_SHIPPED = 'shipped';
    const STATE_DELIVERED = 'delivered';
    const STATE_CANCELLED = 'cancelled';
    const STATE_REFUNDED = 'refunded';

    /**
     * Allowed state transitions
     */
    private $_allowedTransitions = [
        self::STATE_PENDING => [self::STATE_PROCESSING, self::STATE_CANCELLED],
        self::STATE_PROCESSING => [self::STATE_SHIPPED, self::STATE_CANCELLED],
        self::STATE_SHIPPED => [self::STATE_DELIVERED, self::STATE_REFUNDED],
        self::STATE_DELIVERED => [self::STATE_REFUNDED],
        self::STATE_CANCELLED => [],
        self::STATE_REFUNDED => []
    ];

    /**
     * State handlers
     */
    private $_stateHandlers = [];

    /**
     * Add state handler
     */
    public function addStateHandler($state, callable $handler)
    {
        $this->_stateHandlers[$state] = $handler;
        return $this;
    }

    /**
     * Transition to new state
     */
    public function transitionTo($newState, array $data = [])
    {
        $currentState = $this->getState();
        
        // Validate transition
        if (!$this->_canTransitionTo($newState)) {
            throw new \InvalidArgumentException(
                sprintf('Cannot transition from %s to %s', $currentState, $newState)
            );
        }

        // Execute pre-transition logic
        $this->_beforeStateTransition($currentState, $newState, $data);

        // Update state
        $oldState = $this->getState();
        $this->setState($newState);
        $this->save();

        // Execute post-transition logic
        $this->_afterStateTransition($oldState, $newState, $data);

        // Execute state handler
        if (isset($this->_stateHandlers[$newState])) {
            $this->_stateHandlers[$newState]($this, $oldState, $data);
        }

        return $this;
    }

    /**
     * Check if transition is allowed
     */
    private function _canTransitionTo($newState)
    {
        $currentState = $this->getState();
        
        if (!isset($this->_allowedTransitions[$currentState])) {
            return false;
        }
        
        return in_array($newState, $this->_allowedTransitions[$currentState]);
    }

    /**
     * Get allowed transitions from current state
     */
    public function getAllowedTransitions()
    {
        $currentState = $this->getState();
        return $this->_allowedTransitions[$currentState] ?? [];
    }

    /**
     * Before state transition
     */
    protected function _beforeStateTransition($fromState, $toState, array $data)
    {
        $this->_eventManager->dispatch(
            'order_state_transition_before',
            [
                'order' => $this,
                'from_state' => $fromState,
                'to_state' => $toState,
                'data' => $data
            ]
        );
    }

    /**
     * After state transition
     */
    protected function _afterStateTransition($fromState, $toState, array $data)
    {
        $this->_eventManager->dispatch(
            'order_state_transition_after',
            [
                'order' => $this,
                'from_state' => $fromState,
                'to_state' => $toState,
                'data' => $data
            ]
        );
    }

    /**
     * Convenience methods for state transitions
     */
    public function process($data = [])
    {
        return $this->transitionTo(self::STATE_PROCESSING, $data);
    }

    public function ship($trackingNumber = null)
    {
        $data = [];
        if ($trackingNumber) {
            $data['tracking_number'] = $trackingNumber;
        }
        return $this->transitionTo(self::STATE_SHIPPED, $data);
    }

    public function deliver($deliveryDate = null)
    {
        $data = [];
        if ($deliveryDate) {
            $data['delivery_date'] = $deliveryDate;
        }
        return $this->transitionTo(self::STATE_DELIVERED, $data);
    }

    public function cancel($reason = null)
    {
        $data = [];
        if ($reason) {
            $data['cancellation_reason'] = $reason;
        }
        return $this->transitionTo(self::STATE_CANCELLED, $data);
    }

    /**
     * State query methods
     */
    public function isPending()
    {
        return $this->getState() === self::STATE_PENDING;
    }

    public function isProcessing()
    {
        return $this->getState() === self::STATE_PROCESSING;
    }

    public function isShipped()
    {
        return $this->getState() === self::STATE_SHIPPED;
    }

    public function isDelivered()
    {
        return $this->getState() === self::STATE_DELIVERED;
    }

    public function isCancelled()
    {
        return $this->getState() === self::STATE_CANCELLED;
    }

    public function isRefunded()
    {
        return $this->getState() === self::STATE_REFUNDED;
    }

    public function isFinal()
    {
        return in_array($this->getState(), [
            self::STATE_DELIVERED,
            self::STATE_CANCELLED,
            self::STATE_REFUNDED
        ]);
    }
}
```

### 2. Composite Pattern for Hierarchical Data

```php
<?php
namespace Vendor\Module\Model;

abstract class CategoryComposite extends AbstractModel
{
    /**
     * Child categories
     */
    protected $_children = [];

    /**
     * Parent category
     */
    protected $_parent;

    /**
     * Load children
     */
    public function getChildren()
    {
        if (empty($this->_children) && $this->getId()) {
            $this->_loadChildren();
        }
        return $this->_children;
    }

    /**
     * Get parent category
     */
    public function getParent()
    {
        if (!$this->_parent && $this->getParentId()) {
            $this->_parent = $this->_loadParent();
        }
        return $this->_parent;
    }

    /**
     * Add child category
     */
    public function addChild(CategoryComposite $child)
    {
        $child->setParentId($this->getId());
        $child->_parent = $this;
        $this->_children[$child->getId()] = $child;
        return $this;
    }

    /**
     * Remove child category
     */
    public function removeChild($childId)
    {
        if (isset($this->_children[$childId])) {
            $this->_children[$childId]->_parent = null;
            unset($this->_children[$childId]);
        }
        return $this;
    }

    /**
     * Check if category has children
     */
    public function hasChildren()
    {
        return !empty($this->getChildren());
    }

    /**
     * Get all descendants
     */
    public function getAllDescendants()
    {
        $descendants = [];
        
        foreach ($this->getChildren() as $child) {
            $descendants[$child->getId()] = $child;
            $descendants = array_merge($descendants, $child->getAllDescendants());
        }
        
        return $descendants;
    }

    /**
     * Get all ancestors
     */
    public function getAllAncestors()
    {
        $ancestors = [];
        $parent = $this->getParent();
        
        while ($parent) {
            $ancestors[$parent->getId()] = $parent;
            $parent = $parent->getParent();
        }
        
        return array_reverse($ancestors, true);
    }

    /**
     * Get category path
     */
    public function getPath()
    {
        $path = [];
        $ancestors = $this->getAllAncestors();
        
        foreach ($ancestors as $ancestor) {
            $path[] = $ancestor->getName();
        }
        
        $path[] = $this->getName();
        return implode(' > ', $path);
    }

    /**
     * Get category level (depth)
     */
    public function getLevel()
    {
        return count($this->getAllAncestors());
    }

    /**
     * Check if category is root
     */
    public function isRoot()
    {
        return !$this->getParentId();
    }

    /**
     * Check if category is leaf (no children)
     */
    public function isLeaf()
    {
        return !$this->hasChildren();
    }

    /**
     * Move category to new parent
     */
    public function moveTo(CategoryComposite $newParent)
    {
        // Remove from current parent
        if ($this->_parent) {
            $this->_parent->removeChild($this->getId());
        }
        
        // Add to new parent
        $newParent->addChild($this);
        
        // Update database
        $this->setParentId($newParent->getId());
        $this->save();
        
        return $this;
    }

    /**
     * Calculate total products in category and subcategories
     */
    public function getTotalProductCount()
    {
        $count = $this->getProductCount();
        
        foreach ($this->getChildren() as $child) {
            $count += $child->getTotalProductCount();
        }
        
        return $count;
    }

    /**
     * Load children from database
     */
    protected function _loadChildren()
    {
        $collection = $this->getCollection()
            ->addFieldToFilter('parent_id', $this->getId())
            ->addOrder('position', 'ASC');
            
        foreach ($collection as $child) {
            $child->_parent = $this;
            $this->_children[$child->getId()] = $child;
        }
    }

    /**
     * Load parent from database
     */
    protected function _loadParent()
    {
        if (!$this->getParentId()) {
            return null;
        }
        
        $parent = $this->getCollection()
            ->addFieldToFilter('entity_id', $this->getParentId())
            ->getFirstItem();
            
        return $parent->getId() ? $parent : null;
    }
}
```

### 3. Builder Pattern for Complex Models

```php
<?php
namespace Vendor\Module\Model\Builder;

class ProductBuilder
{
    private $productFactory;
    private $categoryLinkManagement;
    private $stockItem;
    private $data = [];

    public function __construct(
        \Vendor\Module\Model\ProductFactory $productFactory,
        \Magento\Catalog\Api\CategoryLinkManagementInterface $categoryLinkManagement,
        \Magento\CatalogInventory\Api\StockItemRepositoryInterface $stockItem
    ) {
        $this->productFactory = $productFactory;
        $this->categoryLinkManagement = $categoryLinkManagement;
        $this->stockItem = $stockItem;
    }

    /**
     * Set basic product data
     */
    public function setBasicData($sku, $name, $price)
    {
        $this->data = array_merge($this->data, [
            'sku' => $sku,
            'name' => $name,
            'price' => $price,
            'type_id' => 'simple',
            'attribute_set_id' => 4, // Default attribute set
            'status' => 1,
            'visibility' => 4
        ]);
        return $this;
    }

    /**
     * Set product type
     */
    public function setType($type)
    {
        $this->data['type_id'] = $type;
        return $this;
    }

    /**
     * Set product status
     */
    public function setStatus($status)
    {
        $this->data['status'] = $status;
        return $this;
    }

    /**
     * Set product description
     */
    public function setDescription($description, $shortDescription = null)
    {
        $this->data['description'] = $description;
        if ($shortDescription) {
            $this->data['short_description'] = $shortDescription;
        }
        return $this;
    }

    /**
     * Set product weight
     */
    public function setWeight($weight)
    {
        $this->data['weight'] = $weight;
        return $this;
    }

    /**
     * Set product categories
     */
    public function setCategories(array $categoryIds)
    {
        $this->data['category_ids'] = $categoryIds;
        return $this;
    }

    /**
     * Set custom attributes
     */
    public function setCustomAttributes(array $attributes)
    {
        foreach ($attributes as $code => $value) {
            $this->data[$code] = $value;
        }
        return $this;
    }

    /**
     * Set stock data
     */
    public function setStock($qty, $isInStock = true, $manageStock = true)
    {
        $this->data['stock_data'] = [
            'qty' => $qty,
            'is_in_stock' => $isInStock,
            'manage_stock' => $manageStock
        ];
        return $this;
    }

    /**
     * Set pricing data
     */
    public function setPricing($price, $specialPrice = null, $specialFromDate = null, $specialToDate = null)
    {
        $this->data['price'] = $price;
        
        if ($specialPrice) {
            $this->data['special_price'] = $specialPrice;
            $this->data['special_from_date'] = $specialFromDate;
            $this->data['special_to_date'] = $specialToDate;
        }
        
        return $this;
    }

    /**
     * Set SEO data
     */
    public function setSeoData($urlKey, $metaTitle = null, $metaDescription = null, $metaKeywords = null)
    {
        $this->data['url_key'] = $urlKey;
        
        if ($metaTitle) {
            $this->data['meta_title'] = $metaTitle;
        }
        
        if ($metaDescription) {
            $this->data['meta_description'] = $metaDescription;
        }
        
        if ($metaKeywords) {
            $this->data['meta_keyword'] = $metaKeywords;
        }
        
        return $this;
    }

    /**
     * Add product images
     */
    public function setImages(array $images)
    {
        $this->data['media_gallery'] = [
            'images' => $images
        ];
        return $this;
    }

    /**
     * Build and return the product
     */
    public function build()
    {
        $product = $this->productFactory->create();
        
        // Set basic data
        foreach ($this->data as $key => $value) {
            if ($key !== 'category_ids' && $key !== 'stock_data' && $key !== 'media_gallery') {
                $product->setData($key, $value);
            }
        }
        
        // Save the product first
        $product->save();
        
        // Handle categories
        if (isset($this->data['category_ids'])) {
            $this->_assignCategories($product, $this->data['category_ids']);
        }
        
        // Handle stock
        if (isset($this->data['stock_data'])) {
            $this->_setStockData($product, $this->data['stock_data']);
        }
        
        // Handle images
        if (isset($this->data['media_gallery'])) {
            $this->_addImages($product, $this->data['media_gallery']['images']);
        }
        
        return $product;
    }

    /**
     * Reset builder for reuse
     */
    public function reset()
    {
        $this->data = [];
        return $this;
    }

    /**
     * Create a copy of current builder
     */
    public function copy()
    {
        $copy = clone $this;
        $copy->data = $this->data;
        return $copy;
    }

    /**
     * Assign categories to product
     */
    private function _assignCategories($product, array $categoryIds)
    {
        $this->categoryLinkManagement->assignProductToCategories(
            $product->getSku(),
            $categoryIds
        );
    }

    /**
     * Set stock data
     */
    private function _setStockData($product, array $stockData)
    {
        $stockItem = $this->stockItem->get($product->getId());
        
        foreach ($stockData as $key => $value) {
            $stockItem->setData($key, $value);
        }
        
        $this->stockItem->save($stockItem);
    }

    /**
     * Add images to product
     */
    private function _addImages($product, array $images)
    {
        // Implementation would handle image upload and assignment
        // This is simplified for the example
        foreach ($images as $image) {
            // Add image logic here
        }
    }
}

/**
 * Usage example
 */
class ProductBuilderExample
{
    private $productBuilder;

    public function __construct(ProductBuilder $productBuilder)
    {
        $this->productBuilder = $productBuilder;
    }

    /**
     * Create a simple product
     */
    public function createSimpleProduct()
    {
        return $this->productBuilder
            ->setBasicData('SIMPLE-001', 'Simple Product', 29.99)
            ->setDescription('This is a simple product', 'Simple product short description')
            ->setWeight(1.5)
            ->setCategories([3, 4, 5])
            ->setStock(100, true, true)
            ->setSeoData('simple-product', 'Simple Product Meta Title')
            ->build();
    }

    /**
     * Create a configurable product
     */
    public function createConfigurableProduct()
    {
        return $this->productBuilder
            ->setBasicData('CONFIG-001', 'Configurable Product', 0)
            ->setType('configurable')
            ->setDescription('This is a configurable product')
            ->setCategories([3, 4])
            ->setCustomAttributes([
                'color' => 'Blue',
                'size' => 'Medium'
            ])
            ->build();
    }

    /**
     * Create products in bulk
     */
    public function createBulkProducts(array $productsData)
    {
        $products = [];
        
        foreach ($productsData as $productData) {
            $product = $this->productBuilder
                ->reset()
                ->setBasicData($productData['sku'], $productData['name'], $productData['price'])
                ->setDescription($productData['description'])
                ->setCategories($productData['categories'])
                ->setStock($productData['qty'])
                ->build();
                
            $products[] = $product;
        }
        
        return $products;
    }
}
```

## Performance Optimization

### 1. Lazy Loading and Eager Loading

```php
<?php
namespace Vendor\Module\Model;

class OptimizedProduct extends AbstractModel
{
    /**
     * Lazy loading cache
     */
    private $_lazyCache = [];

    /**
     * Eager loading flags
     */
    private $_eagerLoad = [];

    /**
     * Enable eager loading for specific data
     */
    public function setEagerLoad(array $types)
    {
        $this->_eagerLoad = array_merge($this->_eagerLoad, $types);
        return $this;
    }

    /**
     * Get categories with lazy loading
     */
    public function getCategories()
    {
        $cacheKey = 'categories_' . $this->getId();
        
        if (!isset($this->_lazyCache[$cacheKey])) {
            $this->_lazyCache[$cacheKey] = $this->_loadCategories();
        }
        
        return $this->_lazyCache[$cacheKey];
    }

    /**
     * Get related products with lazy loading
     */
    public function getRelatedProducts()
    {
        $cacheKey = 'related_products_' . $this->getId();
        
        if (!isset($this->_lazyCache[$cacheKey])) {
            $this->_lazyCache[$cacheKey] = $this->_loadRelatedProducts();
        }
        
        return $this->_lazyCache[$cacheKey];
    }

    /**
     * Get stock data with lazy loading
     */
    public function getStockData()
    {
        $cacheKey = 'stock_data_' . $this->getId();
        
        if (!isset($this->_lazyCache[$cacheKey])) {
            $this->_lazyCache[$cacheKey] = $this->_loadStockData();
        }
        
        return $this->_lazyCache[$cacheKey];
    }

    /**
     * After load with eager loading
     */
    protected function _afterLoad()
    {
        parent::_afterLoad();
        
        // Eager load requested data
        if (in_array('categories', $this->_eagerLoad)) {
            $this->getCategories();
        }
        
        if (in_array('related_products', $this->_eagerLoad)) {
            $this->getRelatedProducts();
        }
        
        if (in_array('stock_data', $this->_eagerLoad)) {
            $this->getStockData();
        }
        
        return $this;
    }

    /**
     * Clear lazy cache
     */
    public function clearLazyCache($type = null)
    {
        if ($type === null) {
            $this->_lazyCache = [];
        } else {
            $cacheKey = $type . '_' . $this->getId();
            unset($this->_lazyCache[$cacheKey]);
        }
        
        return $this;
    }

    /**
     * Load categories efficiently
     */
    private function _loadCategories()
    {
        // Implementation would use optimized query
        return [];
    }

    /**
     * Load related products efficiently
     */
    private function _loadRelatedProducts()
    {
        // Implementation would use optimized query
        return [];
    }

    /**
     * Load stock data efficiently
     */
    private function _loadStockData()
    {
        // Implementation would use optimized query
        return [];
    }
}
```

### 2. Collection Optimization

```php
<?php
namespace Vendor\Module\Model\ResourceModel\Product;

class OptimizedCollection extends Collection
{
    /**
     * Batch size for bulk operations
     */
    const BATCH_SIZE = 1000;

    /**
     * Enable query optimization
     */
    private $_optimizeQueries = true;

    /**
     * Memory usage optimization
     */
    private $_useMemoryOptimization = false;

    /**
     * Enable memory optimization for large datasets
     */
    public function enableMemoryOptimization()
    {
        $this->_useMemoryOptimization = true;
        return $this;
    }

    /**
     * Optimized item loading
     */
    public function getItems()
    {
        if ($this->_items === null) {
            $this->_renderFilters();
            
            if ($this->_useMemoryOptimization) {
                $this->_items = $this->_getItemsWithMemoryOptimization();
            } else {
                $this->_items = $this->_getItemsOptimized();
            }
        }
        
        return $this->_items;
    }

    /**
     * Get items with memory optimization
     */
    private function _getItemsWithMemoryOptimization()
    {
        $items = [];
        
        // Process in batches to reduce memory usage
        $batchSize = self::BATCH_SIZE;
        $offset = 0;
        
        do {
            $select = clone $this->getSelect();
            $select->limit($batchSize, $offset);
            
            $data = $this->getConnection()->fetchAll($select);
            
            foreach ($data as $row) {
                $item = $this->getNewEmptyItem();
                $item->setData($row);
                $items[$item->getId()] = $item;
            }
            
            $offset += $batchSize;
            
        } while (count($data) === $batchSize);
        
        return $items;
    }

    /**
     * Get items with optimized queries
     */
    private function _getItemsOptimized()
    {
        $items = [];
        $data = $this->getData();
        
        if (!empty($data)) {
            // Bulk load related data
            $this->_bulkLoadRelatedData($data);
            
            foreach ($data as $row) {
                $item = $this->getNewEmptyItem();
                $item->setData($row);
                $this->_assignRelatedData($item);
                $items[$item->getId()] = $item;
            }
        }
        
        return $items;
    }

    /**
     * Bulk load related data for all items
     */
    private function _bulkLoadRelatedData($data)
    {
        $productIds = array_column($data, 'entity_id');
        
        // Bulk load categories
        $this->_bulkLoadCategories($productIds);
        
        // Bulk load stock data
        $this->_bulkLoadStockData($productIds);
        
        // Bulk load custom attributes
        $this->_bulkLoadCustomAttributes($productIds);
    }

    /**
     * Bulk load categories for products
     */
    private function _bulkLoadCategories($productIds)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from(['pc' => $this->getTable('vendor_module_product_category')])
            ->join(
                ['c' => $this->getTable('vendor_module_category')],
                'pc.category_id = c.entity_id',
                ['name', 'url_path']
            )
            ->where('pc.product_id IN (?)', $productIds);

        $categories = $connection->fetchAll($select);
        
        // Group by product ID
        $categoryData = [];
        foreach ($categories as $category) {
            $categoryData[$category['product_id']][] = $category;
        }
        
        $this->setData('_bulk_categories', $categoryData);
    }

    /**
     * Bulk load stock data
     */
    private function _bulkLoadStockData($productIds)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getTable('cataloginventory_stock_item'))
            ->where('product_id IN (?)', $productIds);

        $stockData = $connection->fetchAll($select);
        
        // Index by product ID
        $stockDataIndexed = [];
        foreach ($stockData as $stock) {
            $stockDataIndexed[$stock['product_id']] = $stock;
        }
        
        $this->setData('_bulk_stock', $stockDataIndexed);
    }

    /**
     * Assign bulk loaded data to item
     */
    private function _assignRelatedData($item)
    {
        $productId = $item->getId();
        
        // Assign categories
        $bulkCategories = $this->getData('_bulk_categories');
        if (isset($bulkCategories[$productId])) {
            $item->setData('categories', $bulkCategories[$productId]);
        }
        
        // Assign stock data
        $bulkStock = $this->getData('_bulk_stock');
        if (isset($bulkStock[$productId])) {
            $item->setData('stock_item', $bulkStock[$productId]);
        }
    }

    /**
     * Optimize select for better performance
     */
    public function addFieldToSelect($field, $alias = null)
    {
        if ($this->_optimizeQueries) {
            // Only add field if not already selected
            $columns = $this->getSelect()->getPart(\Zend_Db_Select::COLUMNS);
            foreach ($columns as $column) {
                if ($column[2] === $field || $column[2] === $alias) {
                    return $this; // Field already selected
                }
            }
        }
        
        return parent::addFieldToSelect($field, $alias);
    }

    /**
     * Optimized count query
     */
    public function getSize()
    {
        if ($this->_totalRecords === null) {
            $countSelect = $this->getSelectCountSql();
            
            // Optimize count query
            $countSelect->reset(\Zend_Db_Select::ORDER);
            $countSelect->reset(\Zend_Db_Select::LIMIT_COUNT);
            $countSelect->reset(\Zend_Db_Select::LIMIT_OFFSET);
            
            $this->_totalRecords = $this->getConnection()->fetchOne($countSelect);
        }
        
        return intval($this->_totalRecords);
    }

    /**
     * Add index hints for better performance
     */
    public function addIndexHint($index, $type = 'USE')
    {
        $this->getSelect()->from(
            [$this->getMainTable() . ' ' . $type . ' INDEX (' . $index . ')']
        );
        
        return $this;
    }

    /**
     * Optimize joins
     */
    public function joinOptimized($table, $condition, $columns = [])
    {
        // Use index hints for joins when possible
        $this->getSelect()->joinLeft(
            [$table['alias'] => $table['table']],
            $condition,
            $columns
        );
        
        return $this;
    }
}
```

### 3. Caching Strategies

```php
<?php
namespace Vendor\Module\Model;

class CachedModel extends AbstractModel
{
    /**
     * Cache layers
     */
    const CACHE_L1 = 'memory';
    const CACHE_L2 = 'redis';
    const CACHE_L3 = 'database';

    /**
     * Memory cache
     */
    private static $_memoryCache = [];

    /**
     * Cache backend
     */
    private $cacheBackend;

    /**
     * Multi-level cache get
     */
    public function getCachedData($key, callable $dataProvider, $lifetime = 3600)
    {
        // L1 Cache - Memory
        if (isset(self::$_memoryCache[$key])) {
            return self::$_memoryCache[$key];
        }

        // L2 Cache - Redis/File
        $cachedData = $this->cacheBackend->load($key);
        if ($cachedData !== false) {
            $data = unserialize($cachedData);
            self::$_memoryCache[$key] = $data;
            return $data;
        }

        // L3 Cache - Generate data
        $data = $dataProvider();
        
        // Store in all cache levels
        self::$_memoryCache[$key] = $data;
        $this->cacheBackend->save(
            serialize($data),
            $key,
            $this->getIdentities(),
            $lifetime
        );

        return $data;
    }

    /**
     * Invalidate multi-level cache
     */
    public function invalidateCache($key = null)
    {
        if ($key === null) {
            // Clear all cache for this model
            self::$_memoryCache = [];
            $this->cacheBackend->clean(
                \Zend_Cache::CLEANING_MODE_MATCHING_TAG,
                $this->getIdentities()
            );
        } else {
            // Clear specific key
            unset(self::$_memoryCache[$key]);
            $this->cacheBackend->remove($key);
        }
    }

    /**
     * Cache warming strategy
     */
    public function warmCache()
    {
        // Warm up commonly accessed data
        $this->getCachedData('basic_data', function() {
            return $this->getData();
        });

        $this->getCachedData('computed_data', function() {
            return $this->_computeExpensiveData();
        });
    }

    /**
     * Batch cache operations
     */
    public function batchCacheOperations(array $operations)
    {
        $results = [];
        
        foreach ($operations as $operation) {
            $key = $operation['key'];
            $provider = $operation['provider'];
            $lifetime = $operation['lifetime'] ?? 3600;
            
            $results[$key] = $this->getCachedData($key, $provider, $lifetime);
        }
        
        return $results;
    }
}
```

## Testing Models

### 1. Unit Testing

```php
<?php
namespace Vendor\Module\Test\Unit\Model;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Model\Product;
use Magento\Framework\TestFramework\Unit\Helper\ObjectManager;

class ProductTest extends TestCase
{
    /**
     * @var Product
     */
    private $product;

    /**
     * @var ObjectManager
     */
    private $objectManager;

    /**
     * @var \PHPUnit\Framework\MockObject\MockObject
     */
    private $resourceMock;

    /**
     * @var \PHPUnit\Framework\MockObject\MockObject
     */
    private $eventManagerMock;

    protected function setUp(): void
    {
        $this->objectManager = new ObjectManager($this);

        // Create mocks
        $this->resourceMock = $this->createMock(
            \Vendor\Module\Model\ResourceModel\Product::class
        );
        
        $this->eventManagerMock = $this->createMock(
            \Magento\Framework\Event\ManagerInterface::class
        );

        $contextMock = $this->createMock(\Magento\Framework\Model\Context::class);
        $contextMock->expects($this->any())
            ->method('getEventDispatcher')
            ->willReturn($this->eventManagerMock);

        $registryMock = $this->createMock(\Magento\Framework\Registry::class);

        // Create product instance
        $this->product = $this->objectManager->getObject(
            Product::class,
            [
                'context' => $contextMock,
                'registry' => $registryMock,
                'resource' => $this->resourceMock
            ]
        );
    }

    public function testSetAndGetSku()
    {
        $sku = 'TEST-SKU-123';
        $this->product->setSku($sku);
        
        $this->assertEquals($sku, $this->product->getSku());
    }

    public function testSetAndGetName()
    {
        $name = 'Test Product Name';
        $this->product->setName($name);
        
        $this->assertEquals($name, $this->product->getName());
    }

    public function testSetAndGetPrice()
    {
        $price = 29.99;
        $this->product->setPrice($price);
        
        $this->assertEquals($price, $this->product->getPrice());
    }

    public function testIsEnabled()
    {
        // Test enabled status
        $this->product->setStatus(Product::STATUS_ENABLED);
        $this->assertTrue($this->product->isEnabled());

        // Test disabled status
        $this->product->setStatus(Product::STATUS_DISABLED);
        $this->assertFalse($this->product->isEnabled());
    }

    public function testGetFormattedPrice()
    {
        $this->product->setPrice(29.99);
        $this->assertEquals('$29.99', $this->product->getFormattedPrice());

        $this->product->setPrice(100);
        $this->assertEquals('$100.00', $this->product->getFormattedPrice());
    }

    public function testValidationSuccess()
    {
        $this->product->setSku('VALID-SKU');
        $this->product->setName('Valid Product Name');
        $this->product->setPrice(29.99);
        $this->product->setStatus(Product::STATUS_ENABLED);

        $result = $this->product->validate();
        $this->assertTrue($result);
    }

    public function testValidationFailsForMissingSku()
    {
        $this->product->setName('Valid Product Name');
        $this->product->setPrice(29.99);

        $result = $this->product->validate();
        $this->assertIsArray($result);
        $this->assertArrayHasKey('sku', $result);
    }

    public function testValidationFailsForNegativePrice()
    {
        $this->product->setSku('VALID-SKU');
        $this->product->setName('Valid Product Name');
        $this->product->setPrice(-10);

        $result = $this->product->validate();
        $this->assertIsArray($result);
        $this->assertArrayHasKey('price', $result);
    }

    public function testBeforeSave()
    {
        $this->product->setSku('test-sku');
        $this->product->setName('Test Product');
        $this->product->setPrice(29.99);

        // Mock the resource save method
        $this->resourceMock->expects($this->once())
            ->method('save')
            ->with($this->product)
            ->willReturn($this->product);

        // Test that event is dispatched
        $this->eventManagerMock->expects($this->atLeastOnce())
            ->method('dispatch')
            ->with($this->stringContains('save_before'));

        $this->product->save();
    }

    public function testAfterSave()
    {
        $this->product->setId(1);
        $this->product->setSku('test-sku');

        $this->eventManagerMock->expects($this->atLeastOnce())
            ->method('dispatch')
            ->with($this->stringContains('save_after'));

        $this->product->afterSave();
    }

    /**
     * @dataProvider statusDataProvider
     */
    public function testStatusConstants($status, $expected)
    {
        $this->product->setStatus($status);
        $this->assertEquals($expected, $this->product->isEnabled());
    }

    public function statusDataProvider()
    {
        return [
            [Product::STATUS_ENABLED, true],
            [Product::STATUS_DISABLED, false],
            [1, true],
            [2, false]
        ];
    }

    public function testGetCacheIdentities()
    {
        $this->product->setId(123);
        $identities = $this->product->getIdentities();
        
        $this->assertContains(Product::CACHE_TAG, $identities);
        $this->assertContains(Product::CACHE_TAG . '_123', $identities);
    }

    public function testMagicMethods()
    {
        // Test magic setter
        $this->product->setCustomAttribute('test_value');
        $this->assertEquals('test_value', $this->product->getCustomAttribute());

        // Test magic getter for non-existent property
        $this->assertNull($this->product->getNonExistentProperty());

        // Test has method
        $this->assertTrue($this->product->hasCustomAttribute());
        $this->assertFalse($this->product->hasNonExistentProperty());

        // Test unset method
        $this->product->unsCustomAttribute();
        $this->assertFalse($this->product->hasCustomAttribute());
    }
}
```

### 2. Integration Testing

```php
<?php
namespace Vendor\Module\Test\Integration\Model;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class ProductIntegrationTest extends TestCase
{
    /**
     * @var \Magento\Framework\ObjectManagerInterface
     */
    private $objectManager;

    /**
     * @var \Vendor\Module\Model\ProductFactory
     */
    private $productFactory;

    /**
     * @var \Vendor\Module\Model\ResourceModel\Product
     */
    private $productResource;

    /**
     * @var \Vendor\Module\Api\ProductRepositoryInterface
     */
    private $productRepository;

    protected function setUp(): void
    {
        $this->objectManager = Bootstrap::getObjectManager();
        $this->productFactory = $this->objectManager->get(
            \Vendor\Module\Model\ProductFactory::class
        );
        $this->productResource = $this->objectManager->get(
            \Vendor\Module\Model\ResourceModel\Product::class
        );
        $this->productRepository = $this->objectManager->get(
            \Vendor\Module\Api\ProductRepositoryInterface::class
        );
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testCreateAndLoadProduct()
    {
        // Create product
        $product = $this->productFactory->create();
        $product->setSku('INTEGRATION-TEST-SKU');
        $product->setName('Integration Test Product');
        $product->setPrice(49.99);
        $product->setStatus(\Vendor\Module\Model\Product::STATUS_ENABLED);

        // Save product
        $this->productResource->save($product);
        $this->assertNotNull($product->getId());

        // Load product by ID
        $loadedProduct = $this->productFactory->create();
        $this->productResource->load($loadedProduct, $product->getId());
        
        $this->assertEquals($product->getSku(), $loadedProduct->getSku());
        $this->assertEquals($product->getName(), $loadedProduct->getName());
        $this->assertEquals($product->getPrice(), $loadedProduct->getPrice());
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testLoadBySku()
    {
        // Create and save product
        $product = $this->productFactory->create();
        $product->setSku('SKU-TEST-123');
        $product->setName('SKU Test Product');
        $product->setPrice(29.99);
        $this->productResource->save($product);

        // Load by SKU
        $loadedProduct = $this->productFactory->create();
        $this->productResource->loadBySku($loadedProduct, 'SKU-TEST-123');

        $this->assertEquals($product->getId(), $loadedProduct->getId());
        $this->assertEquals('SKU-TEST-123', $loadedProduct->getSku());
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testProductRepository()
    {
        // Create product using repository
        $productData = [
            'sku' => 'REPO-TEST-SKU',
            'name' => 'Repository Test Product',
            'price' => 39.99,
            'status' => \Vendor\Module\Model\Product::STATUS_ENABLED
        ];

        $product = $this->productFactory->create();
        $product->setData($productData);
        
        $savedProduct = $this->productRepository->save($product);
        $this->assertNotNull($savedProduct->getId());

        // Retrieve using repository
        $retrievedProduct = $this->productRepository->getById($savedProduct->getId());
        $this->assertEquals($savedProduct->getSku(), $retrievedProduct->getSku());

        // Delete using repository
        $this->productRepository->delete($retrievedProduct);
        
        $this->expectException(\Magento\Framework\Exception\NoSuchEntityException::class);
        $this->productRepository->getById($savedProduct->getId());
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testProductCollection()
    {
        // Create multiple products
        $products = [];
        for ($i = 1; $i <= 5; $i++) {
            $product = $this->productFactory->create();
            $product->setSku("COLLECTION-TEST-{$i}");
            $product->setName("Collection Test Product {$i}");
            $product->setPrice(10 * $i);
            $product->setStatus(\Vendor\Module\Model\Product::STATUS_ENABLED);
            $this->productResource->save($product);
            $products[] = $product;
        }

        // Test collection
        $collection = $this->objectManager->create(
            \Vendor\Module\Model\ResourceModel\Product\Collection::class
        );
        
        $collection->addFieldToFilter('sku', ['like' => 'COLLECTION-TEST-%']);
        $this->assertEquals(5, $collection->getSize());

        // Test filtering
        $collection->addFieldToFilter('price', ['gt' => 30]);
        $this->assertEquals(2, $collection->getSize());
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testProductEvents()
    {
        $eventTriggered = false;
        
        // Register event observer
        $this->objectManager->get(\Magento\Framework\Event\Manager::class)
            ->dispatch('vendor_module_product_save_after', [
                'product' => $this->productFactory->create()
            ]);

        // Create and save product to trigger events
        $product = $this->productFactory->create();
        $product->setSku('EVENT-TEST-SKU');
        $product->setName('Event Test Product');
        $product->setPrice(19.99);
        
        $this->productResource->save($product);
        
        // Events should be triggered during save
        $this->assertTrue(true); // Placeholder for actual event testing
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testProductValidation()
    {
        $product = $this->productFactory->create();
        
        // Test validation failure
        $this->expectException(\Magento\Framework\Exception\LocalizedException::class);
        $this->productResource->save($product);
    }

    /**
     * @magentoDbIsolation enabled
     * @magentoDataFixture Vendor_Module::Test/_files/products.php
     */
    public function testWithDataFixture()
    {
        // This test would use a data fixture
        // The fixture file would create test data
        $collection = $this->objectManager->create(
            \Vendor\Module\Model\ResourceModel\Product\Collection::class
        );
        
        $this->assertGreaterThan(0, $collection->getSize());
    }
}
```

### 3. Database Testing

```php
<?php
namespace Vendor\Module\Test\Integration\Model\ResourceModel;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class ProductResourceTest extends TestCase
{
    private $objectManager;
    private $resource;
    private $connection;

    protected function setUp(): void
    {
        $this->objectManager = Bootstrap::getObjectManager();
        $this->resource = $this->objectManager->get(
            \Vendor\Module\Model\ResourceModel\Product::class
        );
        $this->connection = $this->resource->getConnection();
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testDatabaseSchema()
    {
        $tableName = $this->resource->getMainTable();
        
        // Test table exists
        $this->assertTrue($this->connection->isTableExists($tableName));
        
        // Test required columns exist
        $requiredColumns = ['entity_id', 'sku', 'name', 'price', 'status', 'created_at', 'updated_at'];
        $tableColumns = $this->connection->describeTable($tableName);
        
        foreach ($requiredColumns as $column) {
            $this->assertArrayHasKey($column, $tableColumns);
        }
        
        // Test primary key
        $indexes = $this->connection->getIndexList($tableName);
        $this->assertArrayHasKey('PRIMARY', $indexes);
        $this->assertTrue($indexes['PRIMARY']['UNIQUE']);
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testBulkOperations()
    {
        // Test bulk insert
        $products = [
            ['sku' => 'BULK-1', 'name' => 'Bulk Product 1', 'price' => 10.00, 'status' => 1],
            ['sku' => 'BULK-2', 'name' => 'Bulk Product 2', 'price' => 20.00, 'status' => 1],
            ['sku' => 'BULK-3', 'name' => 'Bulk Product 3', 'price' => 30.00, 'status' => 1]
        ];
        
        $this->resource->batchInsert($products);
        
        // Verify products were inserted
        $select = $this->connection->select()
            ->from($this->resource->getMainTable())
            ->where('sku LIKE ?', 'BULK-%');
            
        $result = $this->connection->fetchAll($select);
        $this->assertCount(3, $result);
        
        // Test bulk update
        $productIds = array_column($result, 'entity_id');
        $this->resource->bulkUpdate($productIds, ['status' => 2]);
        
        // Verify update
        $select = $this->connection->select()
            ->from($this->resource->getMainTable(), 'status')
            ->where('entity_id IN (?)', $productIds);
            
        $statuses = $this->connection->fetchCol($select);
        foreach ($statuses as $status) {
            $this->assertEquals(2, $status);
        }
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testTransactions()
    {
        $this->connection->beginTransaction();
        
        try {
            // Insert product
            $data = [
                'sku' => 'TRANSACTION-TEST',
                'name' => 'Transaction Test Product',
                'price' => 25.00,
                'status' => 1,
                'created_at' => date('Y-m-d H:i:s'),
                'updated_at' => date('Y-m-d H:i:s')
            ];
            
            $this->connection->insert($this->resource->getMainTable(), $data);
            $productId = $this->connection->lastInsertId();
            
            // Simulate error condition
            if ($productId > 0) {
                $this->connection->commit();
                
                // Verify product exists
                $select = $this->connection->select()
                    ->from($this->resource->getMainTable())
                    ->where('entity_id = ?', $productId);
                    
                $product = $this->connection->fetchRow($select);
                $this->assertNotEmpty($product);
            } else {
                $this->connection->rollback();
            }
            
        } catch (\Exception $e) {
            $this->connection->rollback();
            throw $e;
        }
    }

    /**
     * @magentoDbIsolation enabled
     */
    public function testStatistics()
    {
        // Create test data
        $products = [
            ['sku' => 'STAT-1', 'name' => 'Stats Product 1', 'price' => 10.00, 'status' => 1],
            ['sku' => 'STAT-2', 'name' => 'Stats Product 2', 'price' => 20.00, 'status' => 1],
            ['sku' => 'STAT-3', 'name' => 'Stats Product 3', 'price' => 30.00, 'status' => 2]
        ];
        
        foreach ($products as $productData) {
            $productData['created_at'] = date('Y-m-d H:i:s');
            $productData['updated_at'] = date('Y-m-d H:i:s');
            $this->connection->insert($this->resource->getMainTable(), $productData);
        }
        
        // Test statistics
        $stats = $this->resource->getStatistics();
        
        $this->assertArrayHasKey('total_products', $stats);
        $this->assertArrayHasKey('enabled_products', $stats);
        $this->assertArrayHasKey('disabled_products', $stats);
        $this->assertArrayHasKey('avg_price', $stats);
        
        $this->assertGreaterThanOrEqual(3, $stats['total_products']);
        $this->assertGreaterThanOrEqual(2, $stats['enabled_products']);
        $this->assertGreaterThanOrEqual(1, $stats['disabled_products']);
        $this->assertEquals(20, $stats['avg_price']);
    }
}
```

## Common Patterns and Anti-Patterns

### ✅ Good Patterns

#### 1. Proper Model Initialization

```php
<?php
// Good: Proper model initialization
class Product extends AbstractModel
{
    protected function _construct()
    {
        // Initialize resource model
        $this->_init(\Vendor\Module\Model\ResourceModel\Product::class);
        
        // Set event configuration
        $this->_eventPrefix = 'vendor_module_product';
        $this->_eventObject = 'product';
        
        // Set cache configuration
        $this->_cacheTag = 'vendor_module_product';
        
        // Initialize default values
        $this->setData($this->getDefaultValues());
    }
    
    public function getDefaultValues()
    {
        return [
            'status' => self::STATUS_ENABLED,
            'visibility' => 4,
            'created_at' => date('Y-m-d H:i:s')
        ];
    }
}
```

#### 2. Proper Validation Implementation

```php
<?php
// Good: Comprehensive validation
public function validateBeforeSave()
{
    parent::validateBeforeSave();
    
    // Required field validation
    $requiredFields = ['sku', 'name', 'price'];
    foreach ($requiredFields as $field) {
        if (!$this->getData($field)) {
            throw new LocalizedException(__('%1 is required.', ucfirst($field)));
        }
    }
    
    // Business rule validation
    if ($this->getPrice() < 0) {
        throw new LocalizedException(__('Price cannot be negative.'));
    }
    
    // Custom validation
    if (!$this->_validateSku()) {
        throw new LocalizedException(__('SKU format is invalid.'));
    }
    
    return $this;
}
```

#### 3. Proper Event Usage

```php
<?php
// Good: Meaningful events with proper data
public function processOrder()
{
    $this->_eventManager->dispatch(
        'order_process_before',
        [
            'order' => $this,
            'payment_method' => $this->getPaymentMethod(),
            'total_amount' => $this->getTotalAmount()
        ]
    );
    
    $result = $this->_performProcessing();
    
    $this->_eventManager->dispatch(
        'order_process_after',
        [
            'order' => $this,
            'result' => $result,
            'processing_time' => $this->getProcessingTime()
        ]
    );
    
    return $result;
}
```

### ❌ Anti-Patterns to Avoid

#### 1. Fat Models

```php
<?php
// Bad: Model doing too many things
class Product extends AbstractModel
{
    public function save()
    {
        // Validation
        $this->validateData();
        
        // Image processing
        $this->processImages();
        
        // Email notifications
        $this->sendNotifications();
        
        // Update search index
        $this->updateSearchIndex();
        
        // Update recommendations
        $this->updateRecommendations();
        
        // Generate reports
        $this->generateReports();
        
        // Sync with third-party
        $this->syncWithThirdParty();
        
        return parent::save();
    }
}

// Good: Delegating responsibilities
class Product extends AbstractModel
{
    private $imageProcessor;
    private $notificationService;
    private $searchIndexer;
    
    public function save()
    {
        $result = parent::save();
        
        // Delegate to appropriate services
        $this->imageProcessor->processProductImages($this);
        $this->notificationService->sendProductUpdateNotification($this);
        $this->searchIndexer->reindexProduct($this);
        
        return $result;
    }
}
```

#### 2. Direct Database Access in Models

```php
<?php
// Bad: Direct database queries in models
class Product extends AbstractModel
{
    public function getRelatedProducts()
    {
        $connection = $this->getResourceConnection();
        $sql = "SELECT * FROM product_relations WHERE product_id = " . $this->getId();
        return $connection->fetchAll($sql);
    }
}

// Good: Using resource models and repositories
class Product extends AbstractModel
{
    private $relatedProductRepository;
    
    public function getRelatedProducts()
    {
        return $this->relatedProductRepository->getByProductId($this->getId());
    }
}
```

#### 3. Ignoring Change Tracking

```php
<?php
// Bad: Not utilizing change tracking
public function updatePrice($newPrice)
{
    $this->setPrice($newPrice);
    $this->save();
    
    // Missing: No check if price actually changed
    // Missing: No handling of price change implications
}

// Good: Proper change tracking usage
public function updatePrice($newPrice)
{
    $oldPrice = $this->getPrice();
    $this->setPrice($newPrice);
    
    if ($this->dataHasChangedFor('price')) {
        // Handle price change implications
        $this->_handlePriceChange($oldPrice, $newPrice);
        $this->save();
        
        // Dispatch price change event
        $this->_eventManager->dispatch('product_price_changed', [
            'product' => $this,
            'old_price' => $oldPrice,
            'new_price' => $newPrice
        ]);
    }
}
```

#### 4. Poor Error Handling

```php
<?php
// Bad: Silent failures and poor error handling
public function processPayment()
{
    try {
        $this->chargeCard();
        $this->setStatus('paid');
    } catch (Exception $e) {
        // Silent failure
        $this->setStatus('failed');
    }
    
    $this->save();
}

// Good: Proper error handling and logging
public function processPayment()
{
    try {
        $this->_validatePaymentData();
        $result = $this->paymentProcessor->charge($this->getPaymentData());
        
        $this->setStatus('paid');
        $this->setPaymentResult($result);
        
    } catch (PaymentException $e) {
        $this->setStatus('payment_failed');
        $this->setErrorMessage($e->getMessage());
        
        $this->logger->error('Payment processing failed', [
            'order_id' => $this->getId(),
            'error' => $e->getMessage(),
            'trace' => $e->getTraceAsString()
        ]);
        
        throw $e; // Re-throw for upstream handling
        
    } catch (Exception $e) {
        $this->setStatus('processing_error');
        
        $this->logger->critical('Unexpected error during payment processing', [
            'order_id' => $this->getId(),
            'error' => $e->getMessage()
        ]);
        
        throw new ProcessingException('Payment processing failed due to system error', 0, $e);
    }
    
    $this->save();
}
```

#### 5. Inefficient Data Loading

```php
<?php
// Bad: N+1 query problem
public function getProductsWithCategories()
{
    $products = $this->getCollection();
    $result = [];
    
    foreach ($products as $product) {
        $categories = $product->getCategories(); // Triggers separate query for each product
        $result[] = [
            'product' => $product,
            'categories' => $categories
        ];
    }
    
    return $result;
}

// Good: Bulk loading to avoid N+1 queries
public function getProductsWithCategories()
{
    $collection = $this->getCollection();
    
    // Bulk load categories for all products
    $productIds = $collection->getAllIds();
    $categoryData = $this->_bulkLoadCategories($productIds);
    
    $result = [];
    foreach ($collection as $product) {
        $productId = $product->getId();
        $categories = isset($categoryData[$productId]) ? $categoryData[$productId] : [];
        
        $result[] = [
            'product' => $product,
            'categories' => $categories
        ];
    }
    
    return $result;
}
```

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. "Call to undefined method" Errors

**Problem**: `Call to undefined method Vendor\Module\Model\Product::getCustomAttribute()`

**Cause**: Magic methods not working properly due to missing `__call` implementation or incorrect method naming.

**Solution**:
```php
<?php
// Ensure proper magic method implementation
public function __call($method, $args)
{
    switch (substr($method, 0, 3)) {
        case 'get':
            $key = $this->_underscore(substr($method, 3));
            return $this->getData($key, isset($args[0]) ? $args[0] : null);

        case 'set':
            $key = $this->_underscore(substr($method, 3));
            $value = isset($args[0]) ? $args[0] : null;
            return $this->setData($key, $value);

        case 'uns':
            $key = $this->_underscore(substr($method, 3));
            return $this->unsetData($key);

        case 'has':
            $key = $this->_underscore(substr($method, 3));
            return isset($this->_data[$key]);
    }

    throw new \Exception("Invalid method " . get_class($this) . "::" . $method);
}

// Check method naming - use camelCase
$product->getCustomAttribute(); // Correct
$product->getcustomattribute(); // Incorrect
```

#### 2. Resource Model Not Found

**Problem**: `Class Vendor\Module\Model\ResourceModel\Product does not exist`

**Cause**: Missing resource model class or incorrect `_construct()` method.

**Solution**:
```php
<?php
// 1. Ensure resource model exists
// File: Model/ResourceModel/Product.php
namespace Vendor\Module\Model\ResourceModel;

class Product extends \Magento\Framework\Model\ResourceModel\Db\AbstractDb
{
    protected function _construct()
    {
        $this->_init('vendor_module_product', 'entity_id');
    }
}

// 2. Ensure correct initialization in model
// File: Model/Product.php
protected function _construct()
{
    $this->_init(\Vendor\Module\Model\ResourceModel\Product::class);
}

// 3. Clear generated classes
// bin/magento setup:di:compile
```

#### 3. Events Not Firing

**Problem**: Model events are not being dispatched or observed.

**Cause**: Incorrect event configuration or missing event prefix.

**Solution**:
```php
<?php
// 1. Ensure event prefix is set
protected function _construct()
{
    $this->_eventPrefix = 'vendor_module_product';
    $this->_eventObject = 'product';
    parent::_construct();
}

// 2. Check events.xml configuration
// File: etc/events.xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="vendor_module_product_save_after">
        <observer name="my_observer" instance="Vendor\Module\Observer\ProductSaveObserver" />
    </event>
</config>

// 3. Verify observer class exists and implements ObserverInterface
namespace Vendor\Module\Observer;

use Magento\Framework\Event\ObserverInterface;

class ProductSaveObserver implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer)
    {
        $product = $observer->getData('product');
        // Observer logic here
    }
}
```

#### 4. Collection Performance Issues

**Problem**: Collections are slow or causing memory issues.

**Cause**: Inefficient queries, missing indexes, or loading too much data.

**Solution**:
```php
<?php
// 1. Add proper indexes
// File: etc/db_schema.xml
<index referenceId="VENDOR_MODULE_PRODUCT_SKU" indexType="btree">
    <column name="sku"/>
</index>

<index referenceId="VENDOR_MODULE_PRODUCT_STATUS_CREATED" indexType="btree">
    <column name="status"/>
    <column name="created_at"/>
</index>

// 2. Optimize collection usage
$collection = $this->productCollection->create();
$collection->addFieldToSelect(['entity_id', 'sku', 'name']) // Select only needed fields
           ->addFieldToFilter('status', 1)
           ->setPageSize(50) // Limit results
           ->setCurPage(1);

// 3. Use appropriate loading strategy
$collection->load(); // Standard loading
// OR
foreach ($collection as $item) { // Iterator loading for large datasets
    // Process item
}
```

#### 5. Validation Errors

**Problem**: Validation fails with unclear error messages.

**Cause**: Poor validation implementation or missing validation rules.

**Solution**:
```php
<?php
// Implement comprehensive validation with clear messages
public function validateBeforeSave()
{
    $errors = [];
    
    // Required field validation
    if (!$this->getSku()) {
        $errors[] = __('SKU is required.');
    }
    
    if (!$this->getName()) {
        $errors[] = __('Product name is required.');
    }
    
    // Format validation
    if ($this->getSku() && !preg_match('/^[A-Z0-9_-]+$/', $this->getSku())) {
        $errors[] = __('SKU can only contain uppercase letters, numbers, underscores, and hyphens.');
    }
    
    // Business rule validation
    if ($this->getPrice() !== null && $this->getPrice() < 0) {
        $errors[] = __('Price cannot be negative.');
    }
    
    if (!empty($errors)) {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Validation failed: %1', implode('; ', $errors))
        );
    }
    
    return parent::validateBeforeSave();
}
```

#### 6. Cache Issues

**Problem**: Model data not updating or cache not invalidating properly.

**Cause**: Missing cache tags or incorrect cache implementation.

**Solution**:
```php
<?php
// 1. Implement proper cache identities
public function getIdentities()
{
    $identities = [self::CACHE_TAG];
    
    if ($this->getId()) {
        $identities[] = self::CACHE_TAG . '_' . $this->getId();
    }
    
    // Add related cache tags
    if ($this->getCategoryIds()) {
        foreach ($this->getCategoryIds() as $categoryId) {
            $identities[] = 'category_products_' . $categoryId;
        }
    }
    
    return $identities;
}

// 2. Ensure cache invalidation on save/delete
public function afterSave()
{
    parent::afterSave();
    
    // Clear related cache
    $this->_cacheManager->clean($this->getIdentities());
    
    return $this;
}

// 3. Clear cache programmatically if needed
$this->_cacheManager->clean([
    \Vendor\Module\Model\Product::CACHE_TAG,
    'category_products'
]);
```

## Hands-On Exercises

### Exercise 1: Create a Blog Post Model

**Objective**: Create a complete blog post model with validation, events, and caching.

**Requirements**:
1. Create BlogPost model with fields: id, title, content, author_id, status, published_at
2. Implement validation for required fields and business rules
3. Add before/after save hooks for timestamps and slug generation
4. Implement proper caching with cache tags
5. Create events for post publication and status changes

**Solution Structure**:
```
Model/
├── BlogPost.php
└── ResourceModel/
    ├── BlogPost.php
    └── BlogPost/
        └── Collection.php

Observer/
├── BlogPostSaveObserver.php
└── BlogPostPublishObserver.php

etc/
├── db_schema.xml
├── events.xml
└── di.xml
```

**Implementation Steps**:

1. **Create the Model**:
```php
<?php
namespace Vendor\Blog\Model;

class BlogPost extends \Magento\Framework\Model\AbstractModel implements 
    \Magento\Framework\DataObject\IdentityInterface
{
    const STATUS_DRAFT = 0;
    const STATUS_PUBLISHED = 1;
    const STATUS_ARCHIVED = 2;
    
    const CACHE_TAG = 'vendor_blog_post';
    
    protected $_cacheTag = self::CACHE_TAG;
    protected $_eventPrefix = 'vendor_blog_post';
    
    protected function _construct()
    {
        $this->_init(\Vendor\Blog\Model\ResourceModel\BlogPost::class);
    }
    
    public function getIdentities()
    {
        return [self::CACHE_TAG . '_' . $this->getId()];
    }
    
    public function validateBeforeSave()
    {
        if (!$this->getTitle()) {
            throw new \Exception('Title is required');
        }
        
        if (!$this->getContent()) {
            throw new \Exception('Content is required');
        }
        
        return parent::validateBeforeSave();
    }
    
    public function beforeSave()
    {
        if (!$this->getId()) {
            $this->setCreatedAt(date('Y-m-d H:i:s'));
        }
        
        $this->setUpdatedAt(date('Y-m-d H:i:s'));
        
        // Generate slug from title
        if (!$this->getSlug()) {
            $slug = strtolower(preg_replace('/[^A-Za-z0-9-]+/', '-', $this->getTitle()));
            $this->setSlug(trim($slug, '-'));
        }
        
        return parent::beforeSave();
    }
    
    public function publish()
    {
        $this->setStatus(self::STATUS_PUBLISHED);
        $this->setPublishedAt(date('Y-m-d H:i:s'));
        $this->save();
        
        // Dispatch publication event
        $this->_eventManager->dispatch('vendor_blog_post_published', [
            'post' => $this
        ]);
        
        return $this;
    }
}
```

2. **Create the Resource Model**:
```php
<?php
namespace Vendor\Blog\Model\ResourceModel;

class BlogPost extends \Magento\Framework\Model\ResourceModel\Db\AbstractDb
{
    protected function _construct()
    {
        $this->_init('vendor_blog_post', 'post_id');
    }
    
    public function loadBySlug(\Magento\Framework\Model\AbstractModel $object, $slug)
    {
        $connection = $this->getConnection();
        $select = $connection->select()
            ->from($this->getMainTable())
            ->where('slug = ?', $slug);
            
        $data = $connection->fetchRow($select);
        if ($data) {
            $object->setData($data);
            $this->_afterLoad($object);
        }
        
        return $this;
    }
}
```

### Exercise 2: Implement State Machine Pattern

**Objective**: Create an order model that uses the state machine pattern for status transitions.

**Requirements**:
1. Define order states: pending, processing, shipped, delivered, cancelled
2. Implement state transition rules
3. Add validation for allowed transitions
4. Create state-specific behavior methods
5. Log all state transitions

### Exercise 3: Build Performance-Optimized Collection

**Objective**: Create a product collection with advanced performance optimizations.

**Requirements**:
1. Implement lazy loading for related data
2. Add bulk loading capabilities
3. Create efficient filtering and sorting
4. Implement pagination with memory optimization
5. Add caching layer for expensive operations

## Summary and Best Practices

### Key Takeaways

Models in Magento 2 provide a powerful foundation for business logic and data management:

1. **AbstractModel Features**: Leverages change tracking, validation, events, and caching
2. **Lifecycle Hooks**: Enable custom logic at specific points in the model lifecycle
3. **Resource Models**: Provide efficient database operations and abstraction
4. **Collections**: Offer powerful querying and bulk operation capabilities
5. **Event System**: Enables loose coupling and extensibility
6. **Performance**: Can be optimized through caching, lazy loading, and efficient queries

### Best Practices Summary

#### Model Design
- **Single Responsibility**: Each model should represent one business concept
- **Proper Inheritance**: Extend AbstractModel and implement required interfaces
- **Validation**: Implement comprehensive validation in `validateBeforeSave()`
- **Events**: Use meaningful event names and provide relevant data
- **Caching**: Implement proper cache tags and invalidation

#### Performance Optimization
- **Lazy Loading**: Load expensive data only when needed
- **Bulk Operations**: Use collections and bulk queries for multiple items
- **Indexing**: Add appropriate database indexes for frequently queried fields
- **Memory Management**: Be mindful of memory usage with large datasets
- **Query Optimization**: Select only needed fields and use efficient joins

#### Code Quality
- **Error Handling**: Implement proper exception handling with meaningful messages
- **Logging**: Log important operations and errors for debugging
- **Testing**: Write unit and integration tests for models
- **Documentation**: Use PHPDoc to document model methods and properties
- **Consistency**: Follow Magento coding standards and conventions

#### Security Considerations
- **Input Validation**: Always validate and sanitize input data
- **SQL Injection**: Use parameterized queries and avoid raw SQL
- **Access Control**: Implement proper authorization checks
- **Data Encryption**: Encrypt sensitive data before storage
- **Audit Trail**: Log important data changes for compliance

### Common Gotchas to Avoid

1. **Fat Models**: Keep models focused on data and basic business logic
2. **Direct DB Access**: Use resource models instead of direct database queries
3. **Ignoring Events**: Don't bypass the event system for important operations
4. **Poor Validation**: Validate data thoroughly before saving
5. **Cache Mismanagement**: Implement proper cache invalidation strategies
6. **N+1 Queries**: Use bulk loading to avoid performance issues
7. **Memory Leaks**: Clear object references in long-running processes
8. **Silent Failures**: Always handle and log errors appropriately

### Next Steps for Mastery

1. **Study Core Models**: Examine Magento 2 core models for advanced patterns
2. **Practice Implementation**: Create models for different business scenarios
3. **Performance Profiling**: Learn to identify and optimize bottlenecks
4. **Advanced Patterns**: Study design patterns like State Machine and Builder
5. **Integration**: Understand how models work with Service Layer and APIs
6. **Testing Expertise**: Master unit and integration testing for models
7. **Database Optimization**: Learn advanced database optimization techniques
8. **Event Architecture**: Design complex event-driven workflows

### Recommended Resources

- **Magento 2 Developer Documentation**: Official model documentation
- **Magento 2 Source Code**: Study core model implementations
- **Design Patterns**: Learn GOF patterns applicable to model design
- **Database Optimization**: Study MySQL optimization techniques
- **PHP Best Practices**: Follow modern PHP development practices
- **Testing Frameworks**: Master PHPUnit for comprehensive testing

Models form the core of any Magento 2 application. Mastering their design, implementation, and optimization will significantly improve your ability to build robust, scalable e-commerce solutions. Focus on writing clean, testable, and performant code that follows Magento's architectural principles while meeting your specific business requirements.