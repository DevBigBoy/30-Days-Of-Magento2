# The Complete Guide to Magento 2 Data Objects

## Table of Contents
1. [Learning Objectives](#learning-objectives)
2. [Introduction to Data Objects](#introduction-to-data-objects)
3. [Types of Data Objects](#types-of-data-objects)
4. [Data Object Architecture](#data-object-architecture)
5. [Creating Data Interfaces](#creating-data-interfaces)
6. [Implementing Data Objects](#implementing-data-objects)
7. [Extension Attributes System](#extension-attributes-system)
8. [Custom Attributes Framework](#custom-attributes-framework)
9. [Data Object Factories](#data-object-factories)
10. [Serialization and Hydration](#serialization-and-hydration)
11. [Real-World Implementation Examples](#real-world-implementation-examples)
12. [Advanced Patterns](#advanced-patterns)
13. [Performance Considerations](#performance-considerations)
14. [Testing Data Objects](#testing-data-objects)
15. [Common Patterns and Anti-Patterns](#common-patterns-and-anti-patterns)
16. [Troubleshooting Guide](#troubleshooting-guide)
17. [Hands-On Exercises](#hands-on-exercises)
18. [Summary and Best Practices](#summary-and-best-practices)

## Learning Objectives

By the end of this guide, you will be able to:

- Understand the role of Data Objects in Magento 2 architecture
- Create and implement Data Transfer Objects (DTOs)
- Work with Extension Attributes and Custom Attributes
- Design immutable and mutable data objects
- Implement data validation and transformation
- Use Data Object Factories effectively
- Handle complex data structures and nested objects
- Apply best practices for performance and maintainability
- Test data objects properly
- Troubleshoot common data object issues

## Introduction to Data Objects

Data Objects in Magento 2 are structured containers that hold and transfer data between different layers of the application. They serve as contracts that define the shape and behavior of data, ensuring consistency and type safety throughout the system.

### Why Data Objects Matter

**Type Safety**: Data objects provide compile-time type checking and IDE autocompletion.

**API Stability**: They create stable contracts for data exchange between modules and external systems.

**Validation**: Built-in validation ensures data integrity at the boundary layers.

**Transformation**: They facilitate data transformation between different representations (database, API, UI).

**Documentation**: Interfaces serve as living documentation of data structures.

### Core Principles

1. **Immutability**: Data objects should be immutable when possible to prevent unintended side effects
2. **Validation**: All data should be validated at the boundaries
3. **Encapsulation**: Internal data structure should be hidden behind interfaces
4. **Extensibility**: Objects should support extension without modification
5. **Serialization**: Objects should be easily serializable for API and caching purposes

## Types of Data Objects

### 1. Entity Data Objects

Represent business entities like customers, products, orders:

```php
<?php
namespace Magento\Customer\Api\Data;

interface CustomerInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    const ID = 'id';
    const EMAIL = 'email';
    const FIRSTNAME = 'firstname';
    const LASTNAME = 'lastname';
    const GENDER = 'gender';
    const STORE_ID = 'store_id';
    const WEBSITE_ID = 'website_id';
    const ADDRESSES = 'addresses';
    const DISABLE_AUTO_GROUP_CHANGE = 'disable_auto_group_change';
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';
    const CREATED_IN = 'created_in';
    const DOB = 'dob';
    const GROUP_ID = 'group_id';
    const MIDDLENAME = 'middlename';
    const PREFIX = 'prefix';
    const SUFFIX = 'suffix';
    const TAXVAT = 'taxvat';
    const CONFIRMATION = 'confirmation';
    const CUSTOM_ATTRIBUTES = 'custom_attributes';

    /**
     * Get customer ID
     */
    public function getId();

    /**
     * Set customer ID
     */
    public function setId($id);

    /**
     * Get customer email
     */
    public function getEmail();

    /**
     * Set customer email
     */
    public function setEmail($email);

    // ... more getters and setters
}
```

### 2. Search Result Objects

Handle collections of entities with metadata:

```php
<?php
namespace Magento\Customer\Api\Data;

interface CustomerSearchResultsInterface extends \Magento\Framework\Api\SearchResultsInterface
{
    /**
     * Get customers list.
     */
    public function getItems();

    /**
     * Set customers list.
     */
    public function setItems(array $items);
}
```

### 3. Value Objects

Represent simple data structures:

```php
<?php
namespace Vendor\Module\Api\Data;

interface AddressInterface
{
    const STREET = 'street';
    const CITY = 'city';
    const REGION = 'region';
    const POSTCODE = 'postcode';
    const COUNTRY_ID = 'country_id';

    /**
     * Get street address
     */
    public function getStreet();

    /**
     * Set street address
     */
    public function setStreet($street);

    /**
     * Get city
     */
    public function getCity();

    /**
     * Set city
     */
    public function setCity($city);
}
```

### 4. Configuration Objects

Represent system or module configuration:

```php
<?php
namespace Vendor\Module\Api\Data;

interface ShippingConfigInterface
{
    const ENABLED = 'enabled';
    const FREE_SHIPPING_THRESHOLD = 'free_shipping_threshold';
    const CALCULATION_METHOD = 'calculation_method';
    const ALLOWED_COUNTRIES = 'allowed_countries';

    /**
     * Check if shipping is enabled
     */
    public function isEnabled();

    /**
     * Set shipping enabled status
     */
    public function setEnabled($enabled);

    /**
     * Get free shipping threshold
     */
    public function getFreeShippingThreshold();

    /**
     * Set free shipping threshold
     */
    public function setFreeShippingThreshold($threshold);
}
```

## Data Object Architecture

### Base Classes and Interfaces

Magento 2 provides several base classes for data objects:

#### AbstractExtensibleObject

Used for objects that support extension attributes:

```php
<?php
namespace Magento\Framework\Api;

abstract class AbstractExtensibleObject extends AbstractSimpleObject implements ExtensibleDataInterface
{
    const EXTENSION_ATTRIBUTES_KEY = 'extension_attributes';

    /**
     * Return the extension attributes for the object
     */
    protected function _getExtensionAttributes()
    {
        return $this->_get(self::EXTENSION_ATTRIBUTES_KEY);
    }

    /**
     * Set the extension attributes for the object
     */
    protected function _setExtensionAttributes(ExtensionAttributesInterface $extensionAttributes)
    {
        return $this->setData(self::EXTENSION_ATTRIBUTES_KEY, $extensionAttributes);
    }
}
```

#### AbstractSimpleObject

Basic data object implementation:

```php
<?php
namespace Magento\Framework\Api;

abstract class AbstractSimpleObject implements SimpleDataObjectInterface
{
    /**
     * Array of object data
     */
    protected $_data;

    /**
     * Get data
     */
    protected function _get($key)
    {
        return isset($this->_data[$key]) ? $this->_data[$key] : null;
    }

    /**
     * Set data
     */
    public function setData($key, $value = null)
    {
        if (is_array($key)) {
            $this->_data = $key;
        } else {
            $this->_data[$key] = $value;
        }
        return $this;
    }

    /**
     * Get all data
     */
    public function __toArray()
    {
        $data = $this->_data;
        $hasToArray = function ($model) {
            return is_object($model) && method_exists($model, '__toArray') && is_callable([$model, '__toArray']);
        };
        
        foreach ($data as $key => $value) {
            if ($hasToArray($value)) {
                $data[$key] = $value->__toArray();
            } elseif (is_array($value)) {
                foreach ($value as $nestedKey => $nestedValue) {
                    if ($hasToArray($nestedValue)) {
                        $value[$nestedKey] = $nestedValue->__toArray();
                    }
                }
                $data[$key] = $value;
            }
        }
        return $data;
    }
}
```

## Creating Data Interfaces

### Basic Entity Interface

```php
<?php
namespace Vendor\Module\Api\Data;

/**
 * Product Review interface
 */
interface ProductReviewInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    /**#@+
     * Constants for keys of data array. Identical to the name of the getter in snake case
     */
    const REVIEW_ID = 'review_id';
    const PRODUCT_ID = 'product_id';
    const CUSTOMER_ID = 'customer_id';
    const TITLE = 'title';
    const DETAIL = 'detail';
    const NICKNAME = 'nickname';
    const RATING = 'rating';
    const STATUS = 'status';
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';
    /**#@-*/

    /**
     * Status values
     */
    const STATUS_PENDING = 1;
    const STATUS_APPROVED = 2;
    const STATUS_NOT_APPROVED = 3;

    /**
     * Get review ID
     *
     * @return int|null
     */
    public function getReviewId();

    /**
     * Set review ID
     *
     * @param int $reviewId
     * @return $this
     */
    public function setReviewId($reviewId);

    /**
     * Get product ID
     *
     * @return int
     */
    public function getProductId();

    /**
     * Set product ID
     *
     * @param int $productId
     * @return $this
     */
    public function setProductId($productId);

    /**
     * Get customer ID
     *
     * @return int|null
     */
    public function getCustomerId();

    /**
     * Set customer ID
     *
     * @param int $customerId
     * @return $this
     */
    public function setCustomerId($customerId);

    /**
     * Get review title
     *
     * @return string
     */
    public function getTitle();

    /**
     * Set review title
     *
     * @param string $title
     * @return $this
     */
    public function setTitle($title);

    /**
     * Get review detail
     *
     * @return string
     */
    public function getDetail();

    /**
     * Set review detail
     *
     * @param string $detail
     * @return $this
     */
    public function setDetail($detail);

    /**
     * Get nickname
     *
     * @return string
     */
    public function getNickname();

    /**
     * Set nickname
     *
     * @param string $nickname
     * @return $this
     */
    public function setNickname($nickname);

    /**
     * Get rating
     *
     * @return int
     */
    public function getRating();

    /**
     * Set rating
     *
     * @param int $rating
     * @return $this
     */
    public function setRating($rating);

    /**
     * Get status
     *
     * @return int
     */
    public function getStatus();

    /**
     * Set status
     *
     * @param int $status
     * @return $this
     */
    public function setStatus($status);

    /**
     * Get creation time
     *
     * @return string|null
     */
    public function getCreatedAt();

    /**
     * Set creation time
     *
     * @param string $createdAt
     * @return $this
     */
    public function setCreatedAt($createdAt);

    /**
     * Get update time
     *
     * @return string|null
     */
    public function getUpdatedAt();

    /**
     * Set update time
     *
     * @param string $updatedAt
     * @return $this
     */
    public function setUpdatedAt($updatedAt);

    /**
     * Retrieve existing extension attributes object or create a new one
     *
     * @return \Vendor\Module\Api\Data\ProductReviewExtensionInterface|null
     */
    public function getExtensionAttributes();

    /**
     * Set an extension attributes object
     *
     * @param \Vendor\Module\Api\Data\ProductReviewExtensionInterface $extensionAttributes
     * @return $this
     */
    public function setExtensionAttributes(
        \Vendor\Module\Api\Data\ProductReviewExtensionInterface $extensionAttributes
    );
}
```

### Search Results Interface

```php
<?php
namespace Vendor\Module\Api\Data;

/**
 * Interface for product review search results
 */
interface ProductReviewSearchResultsInterface extends \Magento\Framework\Api\SearchResultsInterface
{
    /**
     * Get reviews list
     *
     * @return \Vendor\Module\Api\Data\ProductReviewInterface[]
     */
    public function getItems();

    /**
     * Set reviews list
     *
     * @param \Vendor\Module\Api\Data\ProductReviewInterface[] $items
     * @return $this
     */
    public function setItems(array $items);
}
```

## Implementing Data Objects

### Basic Implementation

```php
<?php
namespace Vendor\Module\Model\Data;

use Vendor\Module\Api\Data\ProductReviewInterface;
use Magento\Framework\Api\AbstractExtensibleObject;

/**
 * Product Review data object
 */
class ProductReview extends AbstractExtensibleObject implements ProductReviewInterface
{
    /**
     * Get review ID
     */
    public function getReviewId()
    {
        return $this->_get(self::REVIEW_ID);
    }

    /**
     * Set review ID
     */
    public function setReviewId($reviewId)
    {
        return $this->setData(self::REVIEW_ID, $reviewId);
    }

    /**
     * Get product ID
     */
    public function getProductId()
    {
        return $this->_get(self::PRODUCT_ID);
    }

    /**
     * Set product ID
     */
    public function setProductId($productId)
    {
        return $this->setData(self::PRODUCT_ID, $productId);
    }

    /**
     * Get customer ID
     */
    public function getCustomerId()
    {
        return $this->_get(self::CUSTOMER_ID);
    }

    /**
     * Set customer ID
     */
    public function setCustomerId($customerId)
    {
        return $this->setData(self::CUSTOMER_ID, $customerId);
    }

    /**
     * Get title
     */
    public function getTitle()
    {
        return $this->_get(self::TITLE);
    }

    /**
     * Set title
     */
    public function setTitle($title)
    {
        return $this->setData(self::TITLE, $title);
    }

    /**
     * Get detail
     */
    public function getDetail()
    {
        return $this->_get(self::DETAIL);
    }

    /**
     * Set detail
     */
    public function setDetail($detail)
    {
        return $this->setData(self::DETAIL, $detail);
    }

    /**
     * Get nickname
     */
    public function getNickname()
    {
        return $this->_get(self::NICKNAME);
    }

    /**
     * Set nickname
     */
    public function setNickname($nickname)
    {
        return $this->setData(self::NICKNAME, $nickname);
    }

    /**
     * Get rating
     */
    public function getRating()
    {
        return $this->_get(self::RATING);
    }

    /**
     * Set rating
     */
    public function setRating($rating)
    {
        return $this->setData(self::RATING, $rating);
    }

    /**
     * Get status
     */
    public function getStatus()
    {
        return $this->_get(self::STATUS);
    }

    /**
     * Set status
     */
    public function setStatus($status)
    {
        return $this->setData(self::STATUS, $status);
    }

    /**
     * Get creation time
     */
    public function getCreatedAt()
    {
        return $this->_get(self::CREATED_AT);
    }

    /**
     * Set creation time
     */
    public function setCreatedAt($createdAt)
    {
        return $this->setData(self::CREATED_AT, $createdAt);
    }

    /**
     * Get update time
     */
    public function getUpdatedAt()
    {
        return $this->_get(self::UPDATED_AT);
    }

    /**
     * Set update time
     */
    public function setUpdatedAt($updatedAt)
    {
        return $this->setData(self::UPDATED_AT, $updatedAt);
    }

    /**
     * Get extension attributes
     */
    public function getExtensionAttributes()
    {
        return $this->_getExtensionAttributes();
    }

    /**
     * Set extension attributes
     */
    public function setExtensionAttributes(
        \Vendor\Module\Api\Data\ProductReviewExtensionInterface $extensionAttributes
    ) {
        return $this->_setExtensionAttributes($extensionAttributes);
    }
}
```

### Search Results Implementation

```php
<?php
namespace Vendor\Module\Model\Data;

use Vendor\Module\Api\Data\ProductReviewSearchResultsInterface;
use Magento\Framework\Api\SearchResults;

/**
 * Service Data Object with Product Review search results
 */
class ProductReviewSearchResults extends SearchResults implements ProductReviewSearchResultsInterface
{
    /**
     * Get reviews list
     */
    public function getItems()
    {
        return parent::getItems();
    }

    /**
     * Set reviews list
     */
    public function setItems(array $items)
    {
        return parent::setItems($items);
    }
}
```

## Extension Attributes System

Extension attributes allow third-party modules to extend existing data objects without modifying core code.

### Configuring Extension Attributes

```xml
<!-- etc/extension_attributes.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    
    <!-- Add custom attributes to ProductReview -->
    <extension_attributes for="Vendor\Module\Api\Data\ProductReviewInterface">
        <!-- Simple attribute -->
        <attribute code="helpful_count" type="int" />
        
        <!-- Array of primitives -->
        <attribute code="tags" type="string[]" />
        
        <!-- Complex object -->
        <attribute code="reviewer_info" type="Vendor\Module\Api\Data\ReviewerInterface" />
        
        <!-- Array of complex objects -->
        <attribute code="attachments" type="Vendor\Module\Api\Data\ReviewAttachmentInterface[]" />
        
        <!-- Join with another table -->
        <attribute code="product_details" type="Magento\Catalog\Api\Data\ProductInterface">
            <join reference_table="catalog_product_entity" 
                  reference_field="entity_id" 
                  join_on_field="product_id">
                <field>sku</field>
                <field>name</field>
                <field>type_id</field>
            </join>
        </attribute>
    </extension_attributes>

    <!-- Extend Customer object -->
    <extension_attributes for="Magento\Customer\Api\Data\CustomerInterface">
        <attribute code="loyalty_points" type="int" />
        <attribute code="membership_level" type="string" />
        <attribute code="preferred_categories" type="int[]" />
    </extension_attributes>

    <!-- Extend Product object -->
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="review_summary" type="Vendor\Module\Api\Data\ReviewSummaryInterface" />
        <attribute code="brand_info" type="Vendor\Module\Api\Data\BrandInterface" />
    </extension_attributes>
</config>
```

### Working with Extension Attributes

```php
<?php
namespace Vendor\Module\Model;

use Vendor\Module\Api\Data\ProductReviewInterface;
use Vendor\Module\Api\Data\ReviewerInterfaceFactory;

class ReviewService
{
    private $reviewerFactory;

    public function __construct(
        ReviewerInterfaceFactory $reviewerFactory
    ) {
        $this->reviewerFactory = $reviewerFactory;
    }

    /**
     * Add extension data to review
     */
    public function enrichReviewData(ProductReviewInterface $review)
    {
        $extensionAttributes = $review->getExtensionAttributes();
        
        if ($extensionAttributes === null) {
            $extensionAttributes = $this->getExtensionAttributesFactory()->create();
        }

        // Set simple extension attribute
        $extensionAttributes->setHelpfulCount($this->getHelpfulCount($review->getReviewId()));
        
        // Set array extension attribute
        $extensionAttributes->setTags($this->getReviewTags($review->getReviewId()));
        
        // Set complex object extension attribute
        $reviewerInfo = $this->reviewerFactory->create();
        $reviewerInfo->setName($review->getNickname());
        $reviewerInfo->setReviewCount($this->getReviewerReviewCount($review->getCustomerId()));
        $reviewerInfo->setVerifiedPurchase($this->isVerifiedPurchase($review));
        
        $extensionAttributes->setReviewerInfo($reviewerInfo);

        // Set extension attributes back to review
        $review->setExtensionAttributes($extensionAttributes);
        
        return $review;
    }

    /**
     * Process extension attributes when saving
     */
    public function processExtensionAttributes(ProductReviewInterface $review)
    {
        $extensionAttributes = $review->getExtensionAttributes();
        
        if ($extensionAttributes === null) {
            return;
        }

        // Save helpful count
        if ($extensionAttributes->getHelpfulCount() !== null) {
            $this->saveHelpfulCount($review->getReviewId(), $extensionAttributes->getHelpfulCount());
        }

        // Save tags
        if ($extensionAttributes->getTags() !== null) {
            $this->saveReviewTags($review->getReviewId(), $extensionAttributes->getTags());
        }

        // Save reviewer info
        if ($extensionAttributes->getReviewerInfo() !== null) {
            $this->saveReviewerInfo($review->getReviewId(), $extensionAttributes->getReviewerInfo());
        }
    }
}
```

### Extension Attribute Processors

Create processors to automatically populate extension attributes:

```php
<?php
namespace Vendor\Module\Model\Data;

use Magento\Framework\Api\ExtensionAttribute\JoinProcessorInterface;
use Magento\Framework\Api\ExtensionAttributesFactory;

class ProductReviewExtensionAttributeProcessor
{
    private $extensionAttributesFactory;
    private $joinProcessor;

    public function __construct(
        ExtensionAttributesFactory $extensionAttributesFactory,
        JoinProcessorInterface $joinProcessor
    ) {
        $this->extensionAttributesFactory = $extensionAttributesFactory;
        $this->joinProcessor = $joinProcessor;
    }

    /**
     * Process extension attributes for reviews
     */
    public function process(array $reviews)
    {
        foreach ($reviews as $review) {
            $this->processReview($review);
        }
    }

    /**
     * Process single review
     */
    private function processReview($review)
    {
        $extensionAttributes = $review->getExtensionAttributes();
        
        if ($extensionAttributes === null) {
            $extensionAttributes = $this->extensionAttributesFactory->create(
                get_class($review)
            );
        }

        // Process join attributes automatically
        $this->joinProcessor->process($review);

        $review->setExtensionAttributes($extensionAttributes);
    }
}
```

## Custom Attributes Framework

The Custom Attributes framework allows for dynamic attributes that can be configured at runtime.

### EAV Integration

```php
<?php
namespace Vendor\Module\Model\Data;

use Magento\Framework\Api\AbstractExtensibleObject;
use Magento\Framework\Api\AttributeValueFactory;

class CustomAttributeProduct extends AbstractExtensibleObject implements 
    \Magento\Framework\Api\CustomAttributesDataInterface
{
    /**
     * @var AttributeValueFactory
     */
    private $attributeValueFactory;

    /**
     * Initialize dependencies
     */
    public function __construct(
        AttributeValueFactory $attributeValueFactory,
        array $data = []
    ) {
        $this->attributeValueFactory = $attributeValueFactory;
        parent::__construct($data);
    }

    /**
     * Get custom attributes
     */
    public function getCustomAttributes()
    {
        return $this->_get(self::CUSTOM_ATTRIBUTES);
    }

    /**
     * Set custom attributes
     */
    public function setCustomAttributes(array $attributes)
    {
        return $this->setData(self::CUSTOM_ATTRIBUTES, $attributes);
    }

    /**
     * Get custom attribute value
     */
    public function getCustomAttribute($attributeCode)
    {
        $customAttributes = $this->getCustomAttributes();
        
        if ($customAttributes) {
            foreach ($customAttributes as $attribute) {
                if ($attribute->getAttributeCode() === $attributeCode) {
                    return $attribute;
                }
            }
        }
        
        return null;
    }

    /**
     * Set custom attribute value
     */
    public function setCustomAttribute($attributeCode, $attributeValue)
    {
        $customAttributes = $this->getCustomAttributes() ?: [];
        
        // Find existing attribute
        $existingAttribute = null;
        foreach ($customAttributes as $key => $attribute) {
            if ($attribute->getAttributeCode() === $attributeCode) {
                $existingAttribute = $key;
                break;
            }
        }

        // Create new attribute value
        $attribute = $this->attributeValueFactory->create()
            ->setAttributeCode($attributeCode)
            ->setValue($attributeValue);

        if ($existingAttribute !== null) {
            $customAttributes[$existingAttribute] = $attribute;
        } else {
            $customAttributes[] = $attribute;
        }

        return $this->setCustomAttributes($customAttributes);
    }
}
```

### Custom Attribute Helper

```php
<?php
namespace Vendor\Module\Helper;

use Magento\Framework\Api\CustomAttributesDataInterface;
use Magento\Framework\Api\AttributeValue;

class CustomAttributeHelper
{
    /**
     * Convert custom attributes array to key-value array
     */
    public function customAttributesToArray(CustomAttributesDataInterface $object)
    {
        $result = [];
        $customAttributes = $object->getCustomAttributes();
        
        if ($customAttributes) {
            foreach ($customAttributes as $attribute) {
                $result[$attribute->getAttributeCode()] = $attribute->getValue();
            }
        }
        
        return $result;
    }

    /**
     * Convert key-value array to custom attributes
     */
    public function arrayToCustomAttributes(array $data, $attributeValueFactory)
    {
        $customAttributes = [];
        
        foreach ($data as $code => $value) {
            $attribute = $attributeValueFactory->create()
                ->setAttributeCode($code)
                ->setValue($value);
            $customAttributes[] = $attribute;
        }
        
        return $customAttributes;
    }

    /**
     * Merge custom attributes from two objects
     */
    public function mergeCustomAttributes(
        CustomAttributesDataInterface $primary,
        CustomAttributesDataInterface $secondary
    ) {
        $primaryAttributes = $this->customAttributesToArray($primary);
        $secondaryAttributes = $this->customAttributesToArray($secondary);
        
        $mergedAttributes = array_merge($secondaryAttributes, $primaryAttributes);
        
        return $this->arrayToCustomAttributes($mergedAttributes, $this->attributeValueFactory);
    }
}
```

## Data Object Factories

Factories provide a clean way to create data objects with proper dependency injection.

### Basic Factory Usage

```php
<?php
namespace Vendor\Module\Model;

use Vendor\Module\Api\Data\ProductReviewInterfaceFactory;
use Vendor\Module\Api\Data\ProductReviewInterface;

class ReviewService
{
    private $reviewFactory;

    public function __construct(
        ProductReviewInterfaceFactory $reviewFactory
    ) {
        $this->reviewFactory = $reviewFactory;
    }

    /**
     * Create new review from array data
     */
    public function createReviewFromData(array $data)
    {
        /** @var ProductReviewInterface $review */
        $review = $this->reviewFactory->create();
        
        $review->setProductId($data['product_id']);
        $review->setCustomerId($data['customer_id'] ?? null);
        $review->setTitle($data['title']);
        $review->setDetail($data['detail']);
        $review->setNickname($data['nickname']);
        $review->setRating($data['rating']);
        $review->setStatus(ProductReviewInterface::STATUS_PENDING);
        $review->setCreatedAt(date('Y-m-d H:i:s'));
        
        return $review;
    }

    /**
     * Create review with validation
     */
    public function createValidatedReview(array $data)
    {
        $this->validateReviewData($data);
        
        $review = $this->createReviewFromData($data);
        
        // Additional business logic
        if ($this->isCustomerVerified($data['customer_id'])) {
            $review->setStatus(ProductReviewInterface::STATUS_APPROVED);
        }
        
        return $review;
    }

    /**
     * Validate review data
     */
    private function validateReviewData(array $data)
    {
        $required = ['product_id', 'title', 'detail', 'nickname', 'rating'];
        
        foreach ($required as $field) {
            if (empty($data[$field])) {
                throw new \InvalidArgumentException("Field '$field' is required");
            }
        }
        
        if ($data['rating'] < 1 || $data['rating'] > 5) {
            throw new \InvalidArgumentException('Rating must be between 1 and 5');
        }
    }
}
```

### Factory with Hydration

```php
<?php
namespace Vendor\Module\Model;

class ReviewDataHydrator
{
    private $reviewFactory;
    private $reviewerFactory;

    public function __construct(
        ProductReviewInterfaceFactory $reviewFactory,
        ReviewerInterfaceFactory $reviewerFactory
    ) {
        $this->reviewFactory = $reviewFactory;
        $this->reviewerFactory = $reviewerFactory;
    }

    /**
     * Hydrate review from database row
     */
    public function hydrateFromArray(array $data)
    {
        $review = $this->reviewFactory->create();
        
        // Map database fields to object properties
        $fieldMap = [
            'review_id' => 'setReviewId',
            'product_id' => 'setProductId',
            'customer_id' => 'setCustomerId',
            'title' => 'setTitle',
            'detail' => 'setDetail',
            'nickname' => 'setNickname',
            'rating' => 'setRating',
            'status' => 'setStatus',
            'created_at' => 'setCreatedAt',
            'updated_at' => 'setUpdatedAt'
        ];

        foreach ($fieldMap as $field => $method) {
            if (isset($data[$field])) {
                $review->$method($data[$field]);
            }
        }

        // Handle extension attributes
        if (isset($data['extension_attributes'])) {
            $this->hydrateExtensionAttributes($review, $data['extension_attributes']);
        }

        return $review;
    }

    /**
     * Hydrate extension attributes
     */
    private function hydrateExtensionAttributes($review, array $extensionData)
    {
        $extensionAttributes = $review->getExtensionAttributes();
        
        if ($extensionAttributes === null) {
            $extensionAttributes = $this->extensionAttributesFactory->create();
        }

        if (isset($extensionData['helpful_count'])) {
            $extensionAttributes->setHelpfulCount($extensionData['helpful_count']);
        }

        if (isset($extensionData['reviewer_info'])) {
            $reviewerInfo = $this->reviewerFactory->create();
            $reviewerInfo->setName($extensionData['reviewer_info']['name']);
            $reviewerInfo->setReviewCount($extensionData['reviewer_info']['review_count']);
            $extensionAttributes->setReviewerInfo($reviewerInfo);
        }

        $review->setExtensionAttributes($extensionAttributes);
    }

    /**
     * Convert object to array for storage
     */
    public function extractToArray(ProductReviewInterface $review)
    {
        $data = [
            'review_id' => $review->getReviewId(),
            'product_id' => $review->getProductId(),
            'customer_id' => $review->getCustomerId(),
            'title' => $review->getTitle(),
            'detail' => $review->getDetail(),
            'nickname' => $review->getNickname(),
            'rating' => $review->getRating(),
            'status' => $review->getStatus(),
            'created_at' => $review->getCreatedAt(),
            'updated_at' => $review->getUpdatedAt()
        ];

        // Extract extension attributes
        $extensionAttributes = $review->getExtensionAttributes();
        if ($extensionAttributes) {
            $data['extension_attributes'] = $this->extractExtensionAttributes($extensionAttributes);
        }

        return array_filter($data, function($value) {
            return $value !== null;
        });
    }
}
```

## Serialization and Hydration

### JSON Serialization

```php
<?php
namespace Vendor\Module\Model\Data;

use Magento\Framework\Api\AbstractExtensibleObject;

class SerializableReview extends AbstractExtensibleObject implements 
    ProductReviewInterface, 
    \JsonSerializable
{
    /**
     * JSON serialization
     */
    public function jsonSerialize()
    {
        $data = $this->__toArray();
        
        // Custom serialization logic
        if (isset($data['created_at'])) {
            $data['created_at'] = strtotime($data['created_at']);
        }
        
        if (isset($data['updated_at'])) {
            $data['updated_at'] = strtotime($data['updated_at']);
        }

        // Handle extension attributes
        if (isset($data['extension_attributes'])) {
            $data['extension_attributes'] = $this->serializeExtensionAttributes(
                $data['extension_attributes']
            );
        }

        return $data;
    }

    /**
     * Create from JSON
     */
    public static function fromJson($json, $factory)
    {
        $data = json_decode($json, true);
        return $factory->create($data);
    }

    /**
     * Serialize extension attributes
     */
    private function serializeExtensionAttributes($extensionAttributes)
    {
        if (is_object($extensionAttributes) && method_exists($extensionAttributes, '__toArray')) {
            return $extensionAttributes->__toArray();
        }
        
        return $extensionAttributes;
    }
}
```

### Data Converter Service

```php
<?php
namespace Vendor\Module\Service;

use Magento\Framework\Api\SimpleDataObjectConverter;
use Vendor\Module\Api\Data\ProductReviewInterface;

class ReviewDataConverter
{
    private $dataObjectConverter;

    public function __construct(
        SimpleDataObjectConverter $dataObjectConverter
    ) {
        $this->dataObjectConverter = $dataObjectConverter;
    }

    /**
     * Convert review to API format
     */
    public function toApiFormat(ProductReviewInterface $review)
    {
        $data = $this->dataObjectConverter->toFlatArray($review);
        
        // Convert snake_case to camelCase for API
        $apiData = [];
        foreach ($data as $key => $value) {
            $apiKey = $this->snakeToCamel($key);
            $apiData[$apiKey] = $this->convertValue($value);
        }

        return $apiData;
    }

    /**
     * Convert from API format to internal format
     */
    public function fromApiFormat(array $apiData)
    {
        $internalData = [];
        
        foreach ($apiData as $key => $value) {
            $internalKey = $this->camelToSnake($key);
            $internalData[$internalKey] = $this->convertValue($value);
        }

        return $internalData;
    }

    /**
     * Convert snake_case to camelCase
     */
    private function snakeToCamel($string)
    {
        return lcfirst(str_replace('_', '', ucwords($string, '_')));
    }

    /**
     * Convert camelCase to snake_case
     */
    private function camelToSnake($string)
    {
        return strtolower(preg_replace('/([a-z])([A-Z])/', '$1_$2', $string));
    }

    /**
     * Convert values based on type
     */
    private function convertValue($value)
    {
        if (is_array($value)) {
            return array_map([$this, 'convertValue'], $value);
        }
        
        if (is_object($value) && method_exists($value, '__toArray')) {
            return $value->__toArray();
        }

        return $value;
    }
}
```

## Real-World Implementation Examples

### Example 1: E-commerce Order Data Object

```php
<?php
namespace Vendor\Ecommerce\Api\Data;

/**
 * Order interface with complex nested data
 */
interface OrderInterface extends \Magento\Framework\Api\ExtensibleDataInterface
{
    const ORDER_ID = 'order_id';
    const INCREMENT_ID = 'increment_id';
    const CUSTOMER_ID = 'customer_id';
    const CUSTOMER_EMAIL = 'customer_email';
    const STATUS = 'status';
    const STATE = 'state';
    const ITEMS = 'items';
    const BILLING_ADDRESS = 'billing_address';
    const SHIPPING_ADDRESS = 'shipping_address';
    const PAYMENT_INFO = 'payment_info';
    const SHIPPING_INFO = 'shipping_info';
    const TOTALS = 'totals';
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';

    /**
     * Get order items
     *
     * @return \Vendor\Ecommerce\Api\Data\OrderItemInterface[]
     */
    public function getItems();

    /**
     * Set order items
     *
     * @param \Vendor\Ecommerce\Api\Data\OrderItemInterface[] $items
     * @return $this
     */
    public function setItems(array $items);

    /**
     * Get billing address
     *
     * @return \Vendor\Ecommerce\Api\Data\AddressInterface
     */
    public function getBillingAddress();

    /**
     * Set billing address
     *
     * @param \Vendor\Ecommerce\Api\Data\AddressInterface $address
     * @return $this
     */
    public function setBillingAddress(\Vendor\Ecommerce\Api\Data\AddressInterface $address);

    /**
     * Get shipping address
     *
     * @return \Vendor\Ecommerce\Api\Data\AddressInterface|null
     */
    public function getShippingAddress();

    /**
     * Set shipping address
     *
     * @param \Vendor\Ecommerce\Api\Data\AddressInterface $address
     * @return $this
     */
    public function setShippingAddress(\Vendor\Ecommerce\Api\Data\AddressInterface $address);

    /**
     * Get payment information
     *
     * @return \Vendor\Ecommerce\Api\Data\PaymentInfoInterface
     */
    public function getPaymentInfo();

    /**
     * Set payment information
     *
     * @param \Vendor\Ecommerce\Api\Data\PaymentInfoInterface $paymentInfo
     * @return $this
     */
    public function setPaymentInfo(\Vendor\Ecommerce\Api\Data\PaymentInfoInterface $paymentInfo);

    /**
     * Get order totals
     *
     * @return \Vendor\Ecommerce\Api\Data\OrderTotalsInterface
     */
    public function getTotals();

    /**
     * Set order totals
     *
     * @param \Vendor\Ecommerce\Api\Data\OrderTotalsInterface $totals
     * @return $this
     */
    public function setTotals(\Vendor\Ecommerce\Api\Data\OrderTotalsInterface $totals);
}
```

### Order Item Data Object

```php
<?php
namespace Vendor\Ecommerce\Api\Data;

interface OrderItemInterface
{
    const ITEM_ID = 'item_id';
    const ORDER_ID = 'order_id';
    const PRODUCT_ID = 'product_id';
    const SKU = 'sku';
    const NAME = 'name';
    const QTY_ORDERED = 'qty_ordered';
    const QTY_SHIPPED = 'qty_shipped';
    const QTY_INVOICED = 'qty_invoiced';
    const QTY_REFUNDED = 'qty_refunded';
    const PRICE = 'price';
    const DISCOUNT_AMOUNT = 'discount_amount';
    const TAX_AMOUNT = 'tax_amount';
    const ROW_TOTAL = 'row_total';
    const PRODUCT_OPTIONS = 'product_options';

    /**
     * Get item ID
     */
    public function getItemId();

    /**
     * Set item ID
     */
    public function setItemId($itemId);

    /**
     * Get product options
     *
     * @return \Vendor\Ecommerce\Api\Data\ProductOptionInterface[]
     */
    public function getProductOptions();

    /**
     * Set product options
     *
     * @param \Vendor\Ecommerce\Api\Data\ProductOptionInterface[] $options
     */
    public function setProductOptions(array $options);

    // ... other getters and setters
}
```

### Complex Data Object Implementation

```php
<?php
namespace Vendor\Ecommerce\Model\Data;

use Vendor\Ecommerce\Api\Data\OrderInterface;
use Magento\Framework\Api\AbstractExtensibleObject;

class Order extends AbstractExtensibleObject implements OrderInterface
{
    /**
     * Get items with lazy loading
     */
    public function getItems()
    {
        $items = $this->_get(self::ITEMS);
        
        // Lazy load items if not already loaded
        if ($items === null && $this->getOrderId()) {
            $items = $this->loadOrderItems();
            $this->setData(self::ITEMS, $items);
        }
        
        return $items ?: [];
    }

    /**
     * Set items with validation
     */
    public function setItems(array $items)
    {
        // Validate that all items are proper OrderItemInterface instances
        foreach ($items as $item) {
            if (!$item instanceof \Vendor\Ecommerce\Api\Data\OrderItemInterface) {
                throw new \InvalidArgumentException(
                    'All items must implement OrderItemInterface'
                );
            }
        }
        
        return $this->setData(self::ITEMS, $items);
    }

    /**
     * Get billing address with validation
     */
    public function getBillingAddress()
    {
        $address = $this->_get(self::BILLING_ADDRESS);
        
        if ($address && !$address instanceof \Vendor\Ecommerce\Api\Data\AddressInterface) {
            throw new \RuntimeException('Billing address must implement AddressInterface');
        }
        
        return $address;
    }

    /**
     * Calculate total items count
     */
    public function getTotalItemsCount()
    {
        $items = $this->getItems();
        $totalQty = 0;
        
        foreach ($items as $item) {
            $totalQty += $item->getQtyOrdered();
        }
        
        return $totalQty;
    }

    /**
     * Check if order can be shipped
     */
    public function canShip()
    {
        if (!in_array($this->getStatus(), ['processing', 'pending'])) {
            return false;
        }

        $items = $this->getItems();
        foreach ($items as $item) {
            if ($item->getQtyOrdered() > $item->getQtyShipped()) {
                return true;
            }
        }

        return false;
    }

    /**
     * Get extension attributes
     */
    public function getExtensionAttributes()
    {
        return $this->_getExtensionAttributes();
    }

    /**
     * Set extension attributes
     */
    public function setExtensionAttributes(
        \Vendor\Ecommerce\Api\Data\OrderExtensionInterface $extensionAttributes
    ) {
        return $this->_setExtensionAttributes($extensionAttributes);
    }

    /**
     * Load order items (would typically inject repository)
     */
    private function loadOrderItems()
    {
        // This would typically use an injected repository
        // For example purposes, returning empty array
        return [];
    }

    /**
     * Convert to array with nested objects
     */
    public function __toArray()
    {
        $data = parent::__toArray();
        
        // Convert nested objects to arrays
        if (isset($data[self::ITEMS])) {
            $itemsArray = [];
            foreach ($data[self::ITEMS] as $item) {
                $itemsArray[] = $item instanceof AbstractExtensibleObject ? 
                    $item->__toArray() : $item;
            }
            $data[self::ITEMS] = $itemsArray;
        }

        if (isset($data[self::BILLING_ADDRESS]) && 
            $data[self::BILLING_ADDRESS] instanceof AbstractExtensibleObject) {
            $data[self::BILLING_ADDRESS] = $data[self::BILLING_ADDRESS]->__toArray();
        }

        if (isset($data[self::SHIPPING_ADDRESS]) && 
            $data[self::SHIPPING_ADDRESS] instanceof AbstractExtensibleObject) {
            $data[self::SHIPPING_ADDRESS] = $data[self::SHIPPING_ADDRESS]->__toArray();
        }

        return $data;
    }
}
```

### Example 2: Configuration Data Object

```php
<?php
namespace Vendor\Module\Api\Data;

/**
 * Module configuration interface
 */
interface ModuleConfigInterface
{
    const ENABLED = 'enabled';
    const API_KEY = 'api_key';
    const API_ENDPOINT = 'api_endpoint';
    const TIMEOUT = 'timeout';
    const RETRY_ATTEMPTS = 'retry_attempts';
    const LOG_LEVEL = 'log_level';
    const ALLOWED_IPS = 'allowed_ips';
    const RATE_LIMIT = 'rate_limit';
    const FEATURES = 'features';

    /**
     * Check if module is enabled
     */
    public function isEnabled();

    /**
     * Get API configuration
     */
    public function getApiConfig();

    /**
     * Get security configuration
     */
    public function getSecurityConfig();

    /**
     * Get feature flags
     */
    public function getFeatures();

    /**
     * Validate configuration
     */
    public function validate();
}
```

```php
<?php
namespace Vendor\Module\Model\Data;

use Vendor\Module\Api\Data\ModuleConfigInterface;
use Magento\Framework\Api\AbstractSimpleObject;

class ModuleConfig extends AbstractSimpleObject implements ModuleConfigInterface
{
    /**
     * Check if module is enabled
     */
    public function isEnabled()
    {
        return (bool) $this->_get(self::ENABLED);
    }

    /**
     * Get API configuration as object
     */
    public function getApiConfig()
    {
        return new class($this) {
            private $config;
            
            public function __construct($config) {
                $this->config = $config;
            }
            
            public function getKey() {
                return $this->config->_get(ModuleConfigInterface::API_KEY);
            }
            
            public function getEndpoint() {
                return $this->config->_get(ModuleConfigInterface::API_ENDPOINT);
            }
            
            public function getTimeout() {
                return (int) $this->config->_get(ModuleConfigInterface::TIMEOUT) ?: 30;
            }
            
            public function getRetryAttempts() {
                return (int) $this->config->_get(ModuleConfigInterface::RETRY_ATTEMPTS) ?: 3;
            }
        };
    }

    /**
     * Get security configuration
     */
    public function getSecurityConfig()
    {
        return new class($this) {
            private $config;
            
            public function __construct($config) {
                $this->config = $config;
            }
            
            public function getAllowedIps() {
                $ips = $this->config->_get(ModuleConfigInterface::ALLOWED_IPS);
                return $ips ? explode(',', $ips) : [];
            }
            
            public function getRateLimit() {
                return (int) $this->config->_get(ModuleConfigInterface::RATE_LIMIT) ?: 100;
            }
        };
    }

    /**
     * Get feature flags
     */
    public function getFeatures()
    {
        $features = $this->_get(self::FEATURES);
        return $features ? json_decode($features, true) : [];
    }

    /**
     * Validate configuration
     */
    public function validate()
    {
        $errors = [];

        if (!$this->isEnabled()) {
            return $errors; // Skip validation if disabled
        }

        if (!$this->_get(self::API_KEY)) {
            $errors[] = 'API key is required when module is enabled';
        }

        if (!$this->_get(self::API_ENDPOINT)) {
            $errors[] = 'API endpoint is required when module is enabled';
        }

        $timeout = $this->getApiConfig()->getTimeout();
        if ($timeout < 1 || $timeout > 300) {
            $errors[] = 'Timeout must be between 1 and 300 seconds';
        }

        $retryAttempts = $this->getApiConfig()->getRetryAttempts();
        if ($retryAttempts < 0 || $retryAttempts > 10) {
            $errors[] = 'Retry attempts must be between 0 and 10';
        }

        return $errors;
    }

    /**
     * Create from system configuration
     */
    public static function fromSystemConfig($scopeConfig, $scope = 'default', $scopeId = null)
    {
        $configPath = 'vendor_module/general/';
        
        $data = [
            self::ENABLED => $scopeConfig->isSetFlag($configPath . 'enabled', $scope, $scopeId),
            self::API_KEY => $scopeConfig->getValue($configPath . 'api_key', $scope, $scopeId),
            self::API_ENDPOINT => $scopeConfig->getValue($configPath . 'api_endpoint', $scope, $scopeId),
            self::TIMEOUT => $scopeConfig->getValue($configPath . 'timeout', $scope, $scopeId),
            self::RETRY_ATTEMPTS => $scopeConfig->getValue($configPath . 'retry_attempts', $scope, $scopeId),
            self::LOG_LEVEL => $scopeConfig->getValue($configPath . 'log_level', $scope, $scopeId),
            self::ALLOWED_IPS => $scopeConfig->getValue($configPath . 'allowed_ips', $scope, $scopeId),
            self::RATE_LIMIT => $scopeConfig->getValue($configPath . 'rate_limit', $scope, $scopeId),
            self::FEATURES => $scopeConfig->getValue($configPath . 'features', $scope, $scopeId),
        ];

        return new self($data);
    }
}
```

## Advanced Patterns

### 1. Immutable Data Objects

```php
<?php
namespace Vendor\Module\Model\Data;

use Vendor\Module\Api\Data\ImmutableProductInterface;
use Magento\Framework\Api\AbstractSimpleObject;

/**
 * Immutable product data object
 */
class ImmutableProduct extends AbstractSimpleObject implements ImmutableProductInterface
{
    /**
     * Override setData to prevent modification after creation
     */
    public function setData($key, $value = null)
    {
        if (!empty($this->_data)) {
            throw new \LogicException('Cannot modify immutable object');
        }
        
        return parent::setData($key, $value);
    }

    /**
     * Create new instance with modified data
     */
    public function withName($name)
    {
        $data = $this->__toArray();
        $data['name'] = $name;
        
        return new self($data);
    }

    /**
     * Create new instance with modified price
     */
    public function withPrice($price)
    {
        $data = $this->__toArray();
        $data['price'] = $price;
        
        return new self($data);
    }

    /**
     * Create builder for complex modifications
     */
    public function toBuilder()
    {
        return new ProductBuilder($this->__toArray());
    }
}

/**
 * Builder pattern for immutable objects
 */
class ProductBuilder
{
    private $data;

    public function __construct(array $data = [])
    {
        $this->data = $data;
    }

    public function setName($name)
    {
        $this->data['name'] = $name;
        return $this;
    }

    public function setPrice($price)
    {
        $this->data['price'] = $price;
        return $this;
    }

    public function setSku($sku)
    {
        $this->data['sku'] = $sku;
        return $this;
    }

    public function build()
    {
        return new ImmutableProduct($this->data);
    }
}
```

### 2. Composite Data Objects

```php
<?php
namespace Vendor\Module\Model\Data;

/**
 * Composite data object that aggregates multiple related objects
 */
class CustomerProfile extends AbstractExtensibleObject implements CustomerProfileInterface
{
    private $customerRepository;
    private $addressRepository;
    private $orderRepository;

    public function __construct(
        CustomerRepositoryInterface $customerRepository,
        AddressRepositoryInterface $addressRepository,
        OrderRepositoryInterface $orderRepository,
        array $data = []
    ) {
        $this->customerRepository = $customerRepository;
        $this->addressRepository = $addressRepository;
        $this->orderRepository = $orderRepository;
        parent::__construct($data);
    }

    /**
     * Get customer data
     */
    public function getCustomer()
    {
        $customer = $this->_get('customer');
        
        if ($customer === null && $this->getCustomerId()) {
            $customer = $this->customerRepository->getById($this->getCustomerId());
            $this->setData('customer', $customer);
        }
        
        return $customer;
    }

    /**
     * Get customer addresses
     */
    public function getAddresses()
    {
        $addresses = $this->_get('addresses');
        
        if ($addresses === null && $this->getCustomerId()) {
            $searchCriteria = $this->searchCriteriaBuilder
                ->addFilter('customer_id', $this->getCustomerId())
                ->create();
            $addresses = $this->addressRepository->getList($searchCriteria)->getItems();
            $this->setData('addresses', $addresses);
        }
        
        return $addresses ?: [];
    }

    /**
     * Get recent orders
     */
    public function getRecentOrders($limit = 10)
    {
        $orders = $this->_get('recent_orders');
        
        if ($orders === null && $this->getCustomerId()) {
            $searchCriteria = $this->searchCriteriaBuilder
                ->addFilter('customer_id', $this->getCustomerId())
                ->addSortOrder('created_at', 'DESC')
                ->setPageSize($limit)
                ->create();
            $orders = $this->orderRepository->getList($searchCriteria)->getItems();
            $this->setData('recent_orders', $orders);
        }
        
        return $orders ?: [];
    }

    /**
     * Get customer statistics
     */
    public function getStatistics()
    {
        $stats = $this->_get('statistics');
        
        if ($stats === null) {
            $stats = $this->calculateStatistics();
            $this->setData('statistics', $stats);
        }
        
        return $stats;
    }

    /**
     * Calculate customer statistics
     */
    private function calculateStatistics()
    {
        $orders = $this->getRecentOrders(100); // Get more for calculation
        
        $totalSpent = 0;
        $orderCount = count($orders);
        $averageOrderValue = 0;
        
        foreach ($orders as $order) {
            $totalSpent += $order->getTotalPaid();
        }
        
        if ($orderCount > 0) {
            $averageOrderValue = $totalSpent / $orderCount;
        }

        return [
            'total_spent' => $totalSpent,
            'order_count' => $orderCount,
            'average_order_value' => $averageOrderValue,
            'lifetime_value' => $this->calculateLifetimeValue($totalSpent, $orderCount)
        ];
    }
}
```

### 3. Data Object Validation

```php
<?php
namespace Vendor\Module\Model\Data;

use Magento\Framework\Api\AbstractExtensibleObject;
use Vendor\Module\Api\Data\ValidatedProductInterface;

class ValidatedProduct extends AbstractExtensibleObject implements ValidatedProductInterface
{
    /**
     * Validation rules
     */
    private $validationRules = [
        'sku' => ['required', 'string', 'max:64'],
        'name' => ['required', 'string', 'max:255'],
        'price' => ['required', 'numeric', 'min:0'],
        'weight' => ['numeric', 'min:0'],
        'status' => ['required', 'in:1,2'], // enabled/disabled
        'visibility' => ['required', 'in:1,2,3,4'],
        'type_id' => ['required', 'in:simple,configurable,virtual,downloadable']
    ];

    /**
     * Set data with validation
     */
    public function setData($key, $value = null)
    {
        if (is_array($key)) {
            foreach ($key as $k => $v) {
                $this->validateField($k, $v);
            }
        } else {
            $this->validateField($key, $value);
        }

        return parent::setData($key, $value);
    }

    /**
     * Validate entire object
     */
    public function validate()
    {
        $errors = [];
        
        foreach ($this->validationRules as $field => $rules) {
            $value = $this->_get($field);
            $fieldErrors = $this->validateFieldRules($field, $value, $rules);
            if (!empty($fieldErrors)) {
                $errors[$field] = $fieldErrors;
            }
        }

        // Custom business rules
        $businessErrors = $this->validateBusinessRules();
        if (!empty($businessErrors)) {
            $errors = array_merge($errors, $businessErrors);
        }

        return $errors;
    }

    /**
     * Validate single field
     */
    private function validateField($field, $value)
    {
        if (!isset($this->validationRules[$field])) {
            return; // No validation rules for this field
        }

        $rules = $this->validationRules[$field];
        $errors = $this->validateFieldRules($field, $value, $rules);
        
        if (!empty($errors)) {
            throw new \InvalidArgumentException(
                "Validation failed for field '$field': " . implode(', ', $errors)
            );
        }
    }

    /**
     * Validate field against rules
     */
    private function validateFieldRules($field, $value, array $rules)
    {
        $errors = [];

        foreach ($rules as $rule) {
            if (is_string($rule)) {
                $errors = array_merge($errors, $this->validateRule($field, $value, $rule));
            }
        }

        return $errors;
    }

    /**
     * Validate individual rule
     */
    private function validateRule($field, $value, $rule)
    {
        $errors = [];

        switch ($rule) {
            case 'required':
                if (empty($value) && $value !== '0' && $value !== 0) {
                    $errors[] = "$field is required";
                }
                break;

            case 'string':
                if (!is_string($value) && $value !== null) {
                    $errors[] = "$field must be a string";
                }
                break;

            case 'numeric':
                if (!is_numeric($value) && $value !== null) {
                    $errors[] = "$field must be numeric";
                }
                break;

            default:
                if (strpos($rule, 'max:') === 0) {
                    $max = (int) substr($rule, 4);
                    if (strlen($value) > $max) {
                        $errors[] = "$field must not exceed $max characters";
                    }
                } elseif (strpos($rule, 'min:') === 0) {
                    $min = (float) substr($rule, 4);
                    if ((float) $value < $min) {
                        $errors[] = "$field must be at least $min";
                    }
                } elseif (strpos($rule, 'in:') === 0) {
                    $allowed = explode(',', substr($rule, 3));
                    if (!in_array($value, $allowed) && $value !== null) {
                        $errors[] = "$field must be one of: " . implode(', ', $allowed);
                    }
                }
                break;
        }

        return $errors;
    }

    /**
     * Custom business validation rules
     */
    private function validateBusinessRules()
    {
        $errors = [];

        // SKU uniqueness would be checked at repository level
        // Price validation for different product types
        if ($this->_get('type_id') === 'configurable' && $this->_get('price') > 0) {
            $errors['price'][] = 'Configurable products should not have a price';
        }

        // Weight validation for virtual products
        if (in_array($this->_get('type_id'), ['virtual', 'downloadable']) && $this->_get('weight') > 0) {
            $errors['weight'][] = 'Virtual and downloadable products should not have weight';
        }

        return $errors;
    }

    /**
     * Get validation errors without throwing exception
     */
    public function getValidationErrors()
    {
        return $this->validate();
    }

    /**
     * Check if object is valid
     */
    public function isValid()
    {
        return empty($this->validate());
    }
}
```

## Performance Considerations

### 1. Lazy Loading

```php
<?php
namespace Vendor\Module\Model\Data;

class LazyLoadingProduct extends AbstractExtensibleObject implements ProductInterface
{
    private $loaded = [];
    private $productRepository;
    private $stockRepository;
    private $priceRepository;

    public function __construct(
        ProductRepositoryInterface $productRepository,
        StockRepositoryInterface $stockRepository,
        PriceRepositoryInterface $priceRepository,
        array $data = []
    ) {
        $this->productRepository = $productRepository;
        $this->stockRepository = $stockRepository;
        $this->priceRepository = $priceRepository;
        parent::__construct($data);
    }

    /**
     * Get stock information with lazy loading
     */
    public function getStockItem()
    {
        if (!isset($this->loaded['stock']) && $this->getSku()) {
            $stockItem = $this->stockRepository->getBySku($this->getSku());
            $this->setData('stock_item', $stockItem);
            $this->loaded['stock'] = true;
        }

        return $this->_get('stock_item');
    }

    /**
     * Get price information with lazy loading
     */
    public function getPriceInfo()
    {
        if (!isset($this->loaded['price']) && $this->getId()) {
            $priceInfo = $this->priceRepository->getByProductId($this->getId());
            $this->setData('price_info', $priceInfo);
            $this->loaded['price'] = true;
        }

        return $this->_get('price_info');
    }

    /**
     * Get related products with lazy loading
     */
    public function getRelatedProducts()
    {
        if (!isset($this->loaded['related']) && $this->getId()) {
            $relatedProducts = $this->productRepository->getRelatedProducts($this->getId());
            $this->setData('related_products', $relatedProducts);
            $this->loaded['related'] = true;
        }

        return $this->_get('related_products') ?: [];
    }

    /**
     * Preload multiple data sets
     */
    public function preload(array $dataTypes = ['stock', 'price', 'related'])
    {
        foreach ($dataTypes as $type) {
            switch ($type) {
                case 'stock':
                    $this->getStockItem();
                    break;
                case 'price':
                    $this->getPriceInfo();
                    break;
                case 'related':
                    $this->getRelatedProducts();
                    break;
            }
        }

        return $this;
    }
}
```

### 2. Data Object Caching

```php
<?php
namespace Vendor\Module\Model\Data;

use Magento\Framework\Cache\FrontendInterface;
use Magento\Framework\Serialize\SerializerInterface;

class CachingDataObject extends AbstractExtensibleObject
{
    private $cache;
    private $serializer;
    private $cacheLifetime = 3600; // 1 hour

    public function __construct(
        FrontendInterface $cache,
        SerializerInterface $serializer,
        array $data = []
    ) {
        $this->cache = $cache;
        $this->serializer = $serializer;
        parent::__construct($data);
    }

    /**
     * Get cached data or compute and cache
     */
    protected function getCachedData($key, callable $dataProvider, $cacheKey = null)
    {
        if ($cacheKey === null) {
            $cacheKey = $this->generateCacheKey($key);
        }

        // Try to get from cache first
        $cachedData = $this->cache->load($cacheKey);
        if ($cachedData !== false) {
            return $this->serializer->unserialize($cachedData);
        }

        // Compute data
        $data = $dataProvider();

        // Cache the result
        $this->cache->save(
            $this->serializer->serialize($data),
            $cacheKey,
            [$this->getCacheTag()],
            $this->cacheLifetime
        );

        return $data;
    }

    /**
     * Generate cache key
     */
    private function generateCacheKey($key)
    {
        $identifier = $this->getId() ?: $this->getSku() ?: 'unknown';
        return sprintf('data_object_%s_%s_%s', get_class($this), $identifier, $key);
    }

    /**
     * Get cache tag for invalidation
     */
    private function getCacheTag()
    {
        return sprintf('data_object_%s', get_class($this));
    }

    /**
     * Invalidate cache for this object
     */
    public function invalidateCache()
    {
        $this->cache->clean([$this->getCacheTag()]);
    }
}
```

### 3. Bulk Operations

```php
<?php
namespace Vendor\Module\Service;

class BulkDataObjectProcessor
{
    private $dataObjectFactory;
    private $batchSize = 100;

    public function __construct(
        DataObjectFactory $dataObjectFactory
    ) {
        $this->dataObjectFactory = $dataObjectFactory;
    }

    /**
     * Process data objects in batches
     */
    public function processBatch(array $rawData, callable $processor)
    {
        $results = [];
        $batches = array_chunk($rawData, $this->batchSize);

        foreach ($batches as $batch) {
            $dataObjects = $this->createDataObjects($batch);
            $batchResults = $processor($dataObjects);
            $results = array_merge($results, $batchResults);
        }

        return $results;
    }

    /**
     * Create multiple data objects efficiently
     */
    private function createDataObjects(array $rawDataBatch)
    {
        $dataObjects = [];

        foreach ($rawDataBatch as $rawData) {
            $dataObject = $this->dataObjectFactory->create($rawData);
            $dataObjects[] = $dataObject;
        }

        return $dataObjects;
    }

    /**
     * Hydrate data objects from database result
     */
    public function hydrateFromDbResult($dbResult)
    {
        $dataObjects = [];

        while ($row = $dbResult->fetch()) {
            $dataObject = $this->dataObjectFactory->create();
            $this->hydrateFromArray($dataObject, $row);
            $dataObjects[] = $dataObject;
        }

        return $dataObjects;
    }

    /**
     * Efficient hydration from array
     */
    private function hydrateFromArray($dataObject, array $data)
    {
        // Use setData instead of individual setters for performance
        $dataObject->setData($data);
    }
}
```

## Testing Data Objects

### Unit Testing

```php
<?php
namespace Vendor\Module\Test\Unit\Model\Data;

use PHPUnit\Framework\TestCase;
use Vendor\Module\Model\Data\ProductReview;
use Vendor\Module\Api\Data\ProductReviewInterface;

class ProductReviewTest extends TestCase
{
    private $review;

    protected function setUp(): void
    {
        $this->review = new ProductReview();
    }

    public function testSetAndGetReviewId()
    {
        $reviewId = 123;
        $this->review->setReviewId($reviewId);
        
        $this->assertEquals($reviewId, $this->review->getReviewId());
    }

    public function testSetAndGetTitle()
    {
        $title = 'Great Product';
        $this->review->setTitle($title);
        
        $this->assertEquals($title, $this->review->getTitle());
    }

    public function testSetAndGetRating()
    {
        $rating = 5;
        $this->review->setRating($rating);
        
        $this->assertEquals($rating, $this->review->getRating());
    }

    public function testStatusConstants()
    {
        $this->assertEquals(1, ProductReviewInterface::STATUS_PENDING);
        $this->assertEquals(2, ProductReviewInterface::STATUS_APPROVED);
        $this->assertEquals(3, ProductReviewInterface::STATUS_NOT_APPROVED);
    }

    public function testToArray()
    {
        $data = [
            'review_id' => 123,
            'product_id' => 456,
            'title' => 'Great Product',
            'detail' => 'This is a detailed review',
            'rating' => 5,
            'status' => ProductReviewInterface::STATUS_APPROVED
        ];

        foreach ($data as $key => $value) {
            $setter = 'set' . str_replace('_', '', ucwords($key, '_'));
            $this->review->$setter($value);
        }

        $arrayData = $this->review->__toArray();
        
        foreach ($data as $key => $value) {
            $this->assertEquals($value, $arrayData[$key]);
        }
    }

    public function testExtensionAttributes()
    {
        $extensionAttributes = $this->createMock(
            \Vendor\Module\Api\Data\ProductReviewExtensionInterface::class
        );
        
        $this->review->setExtensionAttributes($extensionAttributes);
        
        $this->assertSame($extensionAttributes, $this->review->getExtensionAttributes());
    }

    /**
     * Test data object immutability (if implemented)
     */
    public function testImmutability()
    {
        $review = new ImmutableProductReview([
            'title' => 'Original Title',
            'rating' => 4
        ]);

        // Should create new instance
        $newReview = $review->withTitle('New Title');
        
        $this->assertEquals('Original Title', $review->getTitle());
        $this->assertEquals('New Title', $newReview->getTitle());
        $this->assertNotSame($review, $newReview);
    }
}
```

### Integration Testing

```php
<?php
namespace Vendor\Module\Test\Integration\Model\Data;

use Magento\TestFramework\Helper\Bootstrap;
use PHPUnit\Framework\TestCase;

class ProductReviewIntegrationTest extends TestCase
{
    private $objectManager;
    private $reviewFactory;
    private $reviewRepository;

    protected function setUp(): void
    {
        $this->objectManager = Bootstrap::getObjectManager();
        $this->reviewFactory = $this->objectManager->get(
            \Vendor\Module\Api\Data\ProductReviewInterfaceFactory::class
        );
        $this->reviewRepository = $this->objectManager->get(
            \Vendor\Module\Api\ProductReviewRepositoryInterface::class
        );
    }

    public function testCreateAndSaveReview()
    {
        $review = $this->reviewFactory->create();
        $review->setProductId(1);
        $review->setTitle('Integration Test Review');
        $review->setDetail('This is a test review for integration testing');
        $review->setNickname('Test User');
        $review->setRating(5);
        $review->setStatus(\Vendor\Module\Api\Data\ProductReviewInterface::STATUS_PENDING);

        $savedReview = $this->reviewRepository->save($review);
        
        $this->assertNotNull($savedReview->getReviewId());
        $this->assertEquals('Integration Test Review', $savedReview->getTitle());
    }

    public function testExtensionAttributesIntegration()
    {
        $review = $this->reviewFactory->create();
        $review->setProductId(1);
        $review->setTitle('Extension Test');
        $review->setDetail('Testing extension attributes');
        $review->setNickname('Extension Tester');
        $review->setRating(4);

        // Test extension attributes
        $extensionAttributes = $review->getExtensionAttributes();
        if ($extensionAttributes === null) {
            $extensionAttributesFactory = $this->objectManager->get(
                \Vendor\Module\Api\Data\ProductReviewExtensionInterfaceFactory::class
            );
            $extensionAttributes = $extensionAttributesFactory->create();
        }

        $extensionAttributes->setHelpfulCount(10);
        $review->setExtensionAttributes($extensionAttributes);

        $savedReview = $this->reviewRepository->save($review);
        $retrievedReview = $this->reviewRepository->getById($savedReview->getReviewId());

        $this->assertEquals(10, 
            $retrievedReview->getExtensionAttributes()->getHelpfulCount()
        );
    }
}
```

### Data Provider Testing

```php
<?php
namespace Vendor\Module\Test\Unit\Model\Data;

use PHPUnit\Framework\TestCase;

class ProductReviewDataProviderTest extends TestCase
{
    /**
     * @dataProvider validReviewDataProvider
     */
    public function testValidReviewData($data)
    {
        $review = new ProductReview($data);
        
        $this->assertInstanceOf(ProductReviewInterface::class, $review);
        $this->assertEquals($data['title'], $review->getTitle());
        $this->assertEquals($data['rating'], $review->getRating());
    }

    /**
     * @dataProvider invalidReviewDataProvider
     */
    public function testInvalidReviewData($data, $expectedExceptionMessage)
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage($expectedExceptionMessage);
        
        $review = new ValidatedProductReview($data);
    }

    public function validReviewDataProvider()
    {
        return [
            'basic_review' => [[
                'title' => 'Great Product',
                'detail' => 'I love this product',
                'rating' => 5,
                'nickname' => 'Happy Customer'
            ]],
            'minimal_review' => [[
                'title' => 'OK',
                'detail' => 'It works',
                'rating' => 3,
                'nickname' => 'User'
            ]],
            'detailed_review' => [[
                'title' => 'Comprehensive Review of This Amazing Product',
                'detail' => str_repeat('This is a very detailed review. ', 50),
                'rating' => 4,
                'nickname' => 'Detailed Reviewer'
            ]]
        ];
    }

    public function invalidReviewDataProvider()
    {
        return [
            'missing_title' => [
                ['detail' => 'Good', 'rating' => 5, 'nickname' => 'User'],
                'title is required'
            ],
            'invalid_rating_high' => [
                ['title' => 'Good', 'detail' => 'Good', 'rating' => 6, 'nickname' => 'User'],
                'rating must be between 1 and 5'
            ],
            'invalid_rating_low' => [
                ['title' => 'Bad', 'detail' => 'Bad', 'rating' => 0, 'nickname' => 'User'],
                'rating must be between 1 and 5'
            ],
            'empty_nickname' => [
                ['title' => 'Good', 'detail' => 'Good', 'rating' => 5, 'nickname' => ''],
                'nickname is required'
            ]
        ];
    }
}
```

## Common Patterns and Anti-Patterns

### ✅ Good Patterns

#### 1. Fluent Interface

```php
<?php
$product = $productFactory->create()
    ->setSku('PROD-123')
    ->setName('Amazing Product')
    ->setPrice(99.99)
    ->setStatus(ProductInterface::STATUS_ENABLED)
    ->setWeight(1.5);
```

#### 2. Factory with Validation

```php
<?php
class ValidatedProductFactory
{
    public function create(array $data = [])
    {
        $this->validateData($data);
        return new Product($data);
    }

    public function createFromSku($sku)
    {
        if (empty($sku)) {
            throw new \InvalidArgumentException('SKU cannot be empty');
        }
        
        return $this->create(['sku' => $sku]);
    }
}
```

#### 3. Data Transfer with Validation

```php
<?php
public function transferCustomerData(CustomerInterface $source, CustomerInterface $target)
{
    $allowedFields = ['firstname', 'lastname', 'email', 'dob'];
    
    foreach ($allowedFields as $field) {
        $getter = 'get' . ucfirst($field);
        $setter = 'set' . ucfirst($field);
        
        if (method_exists($source, $getter) && method_exists($target, $setter)) {
            $value = $source->$getter();
            if ($value !== null) {
                $target->$setter($value);
            }
        }
    }
    
    return $target;
}
```

### ❌ Anti-Patterns to Avoid

#### 1. Mutable Data in Getters

```php
<?php
// Bad: Returning mutable arrays that can be modified
public function getItems()
{
    return $this->items; // Direct reference to internal array
}

// Good: Return immutable copy
public function getItems()
{
    return array_values($this->items); // Return copy
}
```

#### 2. Heavy Computation in Getters

```php
<?php
// Bad: Complex computation every time
public function getTotalPrice()
{
    $total = 0;
    foreach ($this->getItems() as $item) {
        $total += $item->getPrice() * $item->getQuantity();
        // Additional complex calculations...
    }
    return $total;
}

// Good: Cache computed values
private $totalPrice;

public function getTotalPrice()
{
    if ($this->totalPrice === null) {
        $this->totalPrice = $this->calculateTotalPrice();
    }
    return $this->totalPrice;
}
```

#### 3. Side Effects in Setters

```php
<?php
// Bad: Side effects in setters
public function setEmail($email)
{
    $this->email = $email;
    $this->sendWelcomeEmail(); // Side effect!
    $this->updateCustomerRecord(); // Side effect!
    return $this;
}

// Good: Pure setters, separate methods for actions
public function setEmail($email)
{
    $this->email = $email;
    return $this;
}

public function sendWelcomeEmail()
{
    // Send email logic
}
```

#### 4. Direct Database Access in Data Objects

```php
<?php
// Bad: Data objects should not know about databases
public function getRelatedProducts()
{
    $connection = $this->resourceConnection->getConnection();
    $select = $connection->select()
        ->from('product_relations')
        ->where('product_id = ?', $this->getId());
    return $connection->fetchAll($select);
}

// Good: Use repositories and services
public function getRelatedProducts()
{
    return $this->_get('related_products');
}
```

## Troubleshooting Guide

### Common Issues and Solutions

#### 1. Extension Attributes Not Loading

**Problem**: Extension attributes return null even when configured

**Solution**:
```php
// Check extension_attributes.xml configuration
// Ensure proper DI configuration
// Verify extension attribute processor is called

// Debug extension attributes
$extensionAttributes = $object->getExtensionAttributes();
if ($extensionAttributes === null) {
    // Create extension attributes if missing
    $extensionAttributesFactory = $this->objectManager->get(
        ExtensionAttributesFactory::class
    );
    $extensionAttributes = $extensionAttributesFactory->create(get_class($object));
    $object->setExtensionAttributes($extensionAttributes);
}
```

#### 2. Data Object Factory Issues

**Problem**: Factory creates wrong class or missing dependencies

**Solution**:
```xml
<!-- Verify DI configuration in di.xml -->
<preference for="Vendor\Module\Api\Data\EntityInterface" 
            type="Vendor\Module\Model\Data\Entity" />

<type name="Vendor\Module\Api\Data\EntityInterfaceFactory">
    <arguments>
        <argument name="instanceName" xsi:type="string">Vendor\Module\Model\Data\Entity</argument>
    </arguments>
</type>
```

#### 3. Serialization Problems

**Problem**: Data objects don't serialize/unserialize properly

**Solution**:
```php
// Implement proper __toArray method
public function __toArray()
{
    $data = parent::__toArray();
    
    // Handle nested objects
    foreach ($data as $key => $value) {
        if (is_object($value) && method_exists($value, '__toArray')) {
            $data[$key] = $value->__toArray();
        } elseif (is_array($value)) {
            $data[$key] = array_map(function($item) {
                return is_object($item) && method_exists($item, '__toArray') 
                    ? $item->__toArray() 
                    : $item;
            }, $value);
        }
    }
    
    return $data;
}
```

#### 4. Performance Issues with Large Objects

**Problem**: Memory usage too high with large data objects

**Solution**:
```php
// Use lazy loading for expensive data
// Implement pagination for collections
// Use weak references for circular dependencies

class LazyDataObject
{
    private $expensiveDataLoader;
    private $expensiveData;
    
    public function getExpensiveData()
    {
        if ($this->expensiveData === null) {
            $this->expensiveData = $this->expensiveDataLoader->load($this->getId());
        }
        return $this->expensiveData;
    }
    
    // Clear expensive data when not needed
    public function clearCache()
    {
        $this->expensiveData = null;
    }
}
```

## Hands-On Exercises

### Exercise 1: Create a Blog Post Data Object

**Objective**: Create a complete blog post data object with comments and tags.

**Requirements**:
1. Create BlogPostInterface with fields: id, title, content, author_id, status, published_at, tags, comments_count
2. Implement BlogPost data object with validation
3. Add extension attributes for: featured_image, meta_description, related_posts
4. Create factory and test the implementation

**Solution Structure**:
```
Api/Data/
├── BlogPostInterface.php
├── BlogPostExtensionInterface.php
└── BlogPostSearchResultsInterface.php

Model/Data/
├── BlogPost.php
└── BlogPostSearchResults.php

Test/Unit/Model/Data/
└── BlogPostTest.php
```

### Exercise 2: Implement Configuration Data Object

**Objective**: Create a configuration data object for payment method settings.

**Requirements**:
1. Support nested configuration (API settings, security settings, features)
2. Implement validation for all configuration values
3. Add method to export/import configuration as JSON
4. Support environment-specific overrides

### Exercise 3: Create Immutable Order Summary

**Objective**: Design an immutable order summary with builder pattern.

**Requirements**:
1. Immutable order summary with totals, items count, customer info
2. Builder pattern for construction
3. Methods to create modified copies (withDiscount, withShipping, etc.)
4. Comprehensive test coverage

## Summary and Best Practices

### Key Takeaways

Data Objects in Magento 2 are essential for:

1. **Type Safety**: Ensuring consistent data structures across the application
2. **API Contracts**: Providing stable interfaces for external integrations
3. **Validation**: Enforcing data integrity at the boundary layers
4. **Extensibility**: Supporting customization through extension attributes
5. **Performance**: Enabling optimization through lazy loading and caching

### Best Practices Summary

#### Design Principles
- **Single Responsibility**: Each data object should represent one concept
- **Immutability**: Prefer immutable objects where possible
- **Validation**: Validate data at creation and modification points
- **Documentation**: Use clear interfaces and PHPDoc comments
- **Consistency**: Follow naming conventions and patterns

#### Implementation Guidelines
- Use AbstractExtensibleObject for objects needing extension attributes
- Implement proper __toArray() methods for serialization
- Use factories for object creation with validation
- Handle null values gracefully in getters and setters
- Implement proper equals() and hashCode() methods when needed

#### Performance Considerations
- Use lazy loading for expensive operations
- Implement caching for frequently accessed computed data
- Avoid heavy computation in getters
- Use bulk operations for processing multiple objects
- Consider memory usage with large collections

#### Testing Strategy
- Write unit tests for all public methods
- Test validation rules thoroughly
- Use data providers for testing multiple scenarios
- Test serialization and deserialization
- Include integration tests for complex interactions

### Next Steps

1. **Practice Implementation**: Create data objects for your custom entities
2. **Study Core Examples**: Examine Magento 2 core data objects for patterns
3. **Explore Advanced Features**: Learn about custom attributes and EAV integration
4. **Performance Optimization**: Implement caching and lazy loading strategies
5. **API Integration**: Use data objects in REST and GraphQL APIs
6. **Testing Mastery**: Develop comprehensive test suites for data objects

Data objects form the foundation of clean, maintainable Magento 2 applications. Mastering their design and implementation will significantly improve your development practices and application architecture.