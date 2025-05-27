# Magento 2 Collection Operations - Filtering, Loading, Sorting & Data Manipulation Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Loading Data](#loading-data)
3. [Filtering Operations](#filtering-operations)
4. [Sorting Operations](#sorting-operations)
5. [Data Manipulation](#data-manipulation)
6. [Advanced Operations](#advanced-operations)
7. [Performance Optimization](#performance-optimization)
8. [Real-World Examples](#real-world-examples)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Introduction

This comprehensive guide focuses on the **four core operations** that make Magento 2 collections powerful:

1. **Loading** - How and when to retrieve data from the database
2. **Filtering** - Narrowing down results based on conditions
3. **Sorting** - Ordering data in meaningful ways
4. **Manipulating** - Processing and transforming collection data

These operations form the foundation of data handling in Magento 2 and are essential for building efficient, scalable applications.

## Loading Data

Loading is the process of retrieving data from the database and converting it into collection items. Understanding different loading strategies is crucial for performance optimization.

### Core Loading Methods

#### **load($printQuery = false, $logQuery = false)**
The primary method to load collection data:

```php
<?php
// Basic loading
$collection = $this->productCollectionFactory->create();
$collection->load();

// Debug mode - prints SQL query
$collection->load(true);

// Log query to debug.log
$collection->load(false, true);

// Check if collection is loaded
if ($collection->isLoaded()) {
    // Process loaded items
    foreach ($collection as $item) {
        echo $item->getName();
    }
}
```

#### **loadData($printQuery = false, $logQuery = false)**
Loads raw data without creating model instances (memory efficient):

```php
// Load raw data array
$collection = $this->productCollectionFactory->create()
    ->addFieldToSelect(['entity_id', 'name', 'price']);

$rawData = $collection->loadData();
// Returns: [['entity_id' => 1, 'name' => 'Product 1', 'price' => 10.00], ...]

// Process raw data (faster for large datasets)
foreach ($rawData as $row) {
    echo "Product ID: " . $row['entity_id'] . " - " . $row['name'];
}
```

#### **getData($key = '', $index = null)**
Retrieves loaded data without triggering load:

```php
$collection->load();

// Get all data as array
$allData = $collection->getData();

// Get specific field from all items
$productNames = $collection->getData('name');

// Get data from specific item
$firstItemData = $collection->getData('', 0);
```

### Lazy Loading Strategies

#### **Conditional Loading**
```php
class SmartProductCollection
{
    private $collection;
    private $isLoaded = false;

    public function getProducts()
    {
        if (!$this->isLoaded) {
            $this->collection->load();
            $this->isLoaded = true;
        }
        return $this->collection;
    }

    public function getProductCount()
    {
        // Use getSize() to get count without loading items
        return $this->collection->getSize();
    }

    public function hasProducts()
    {
        return $this->getProductCount() > 0;
    }
}
```

#### **Partial Loading**
```php
// Load only essential fields first
$collection = $this->productCollectionFactory->create()
    ->addFieldToSelect(['entity_id', 'name', 'price'])
    ->addFieldToFilter('status', 1)
    ->load();

// Later, load additional data for specific items
foreach ($collection as $product) {
    if ($this->needsDetailedInfo($product)) {
        $detailedProduct = $this->productRepository->getById($product->getId());
        // Use detailed product data
    }
}
```

#### **Batch Loading**
```php
class BatchProductLoader
{
    public function loadInBatches($batchSize = 1000, callable $processor = null)
    {
        $collection = $this->productCollectionFactory->create();
        $collection->setPageSize($batchSize);
        
        $page = 1;
        do {
            $collection->setCurPage($page);
            $collection->clear(); // Clear previous page data
            $collection->load();
            
            if ($processor) {
                foreach ($collection as $item) {
                    $processor($item);
                }
            }
            
            $itemCount = $collection->count();
            $page++;
            
        } while ($itemCount === $batchSize);
    }
}

// Usage
$loader = new BatchProductLoader();
$loader->loadInBatches(500, function($product) {
    // Process each product
    $this->updateProductCache($product);
});
```

### Memory-Efficient Loading

#### **Iterator Pattern for Large Collections**
```php
class MemoryEfficientIterator implements \Iterator
{
    private $collection;
    private $batchSize;
    private $currentPage = 1;
    private $currentIndex = 0;
    private $items = [];
    private $totalProcessed = 0;

    public function __construct($collection, $batchSize = 1000)
    {
        $this->collection = $collection;
        $this->batchSize = $batchSize;
        $this->collection->setPageSize($batchSize);
    }

    public function rewind(): void
    {
        $this->currentPage = 1;
        $this->currentIndex = 0;
        $this->totalProcessed = 0;
        $this->loadBatch();
    }

    public function current()
    {
        return $this->items[$this->currentIndex] ?? null;
    }

    public function key()
    {
        return $this->totalProcessed;
    }

    public function next(): void
    {
        $this->currentIndex++;
        $this->totalProcessed++;

        if ($this->currentIndex >= count($this->items)) {
            $this->currentPage++;
            $this->currentIndex = 0;
            $this->loadBatch();
        }
    }

    public function valid(): bool
    {
        return isset($this->items[$this->currentIndex]);
    }

    private function loadBatch(): void
    {
        $this->collection->setCurPage($this->currentPage);
        $this->collection->clear();
        $this->collection->load();
        $this->items = $this->collection->getItems();
    }
}

// Usage
$collection = $this->productCollectionFactory->create()
    ->addFieldToFilter('status', 1);

$iterator = new MemoryEfficientIterator($collection, 1000);

foreach ($iterator as $product) {
    // Process without loading entire collection into memory
    $this->processProduct($product);
}
```

## Filtering Operations

Filtering is the most commonly used operation to narrow down collection results based on specific conditions.

### Basic Filtering Methods

#### **addFieldToFilter($field, $condition)**
The primary filtering method for flat table fields:

```php
// Equal condition
$collection->addFieldToFilter('status', 1);
$collection->addFieldToFilter('name', 'Product Name');

// Not equal
$collection->addFieldToFilter('status', ['neq' => 0]);

// Greater than / Less than
$collection->addFieldToFilter('price', ['gt' => 100]);
$collection->addFieldToFilter('price', ['lt' => 500]);
$collection->addFieldToFilter('price', ['gteq' => 100]); // >=
$collection->addFieldToFilter('price', ['lteq' => 500]); // <=

// LIKE patterns
$collection->addFieldToFilter('name', ['like' => '%product%']);
$collection->addFieldToFilter('sku', ['like' => 'ABC%']);
$collection->addFieldToFilter('description', ['nlike' => '%old%']); // NOT LIKE

// IN / NOT IN arrays
$collection->addFieldToFilter('category_id', ['in' => [1, 2, 3, 4]]);
$collection->addFieldToFilter('status', ['nin' => [0, 2]]); // NOT IN

// NULL / NOT NULL
$collection->addFieldToFilter('special_price', ['null' => true]);
$collection->addFieldToFilter('image', ['notnull' => true]);

// Date ranges
$collection->addFieldToFilter('created_at', [
    'from' => '2024-01-01 00:00:00',
    'to' => '2024-12-31 23:59:59'
]);

// Date comparisons
$collection->addFieldToFilter('created_at', ['gteq' => '2024-01-01']);
$collection->addFieldToFilter('updated_at', ['lt' => date('Y-m-d H:i:s', strtotime('-30 days'))]);
```

#### **addAttributeToFilter($attribute, $condition)**
For EAV entities (products, customers, categories):

```php
// Product attribute filtering
$productCollection = $this->productCollectionFactory->create();

// Simple attribute filter
$productCollection->addAttributeToFilter('status', 1);
$productCollection->addAttributeToFilter('visibility', ['neq' => 1]);

// Attribute with options
$productCollection->addAttributeToFilter('color', ['in' => [23, 24, 25]]); // Option IDs

// Text attributes
$productCollection->addAttributeToFilter('name', ['like' => '%shirt%']);
$productCollection->addAttributeToFilter('manufacturer', 'Nike');

// Numeric attributes
$productCollection->addAttributeToFilter('price', ['from' => 50, 'to' => 200]);
$productCollection->addAttributeToFilter('weight', ['gt' => 1.5]);

// Boolean attributes
$productCollection->addAttributeToFilter('is_featured', 1);

// Date attributes
$productCollection->addAttributeToFilter('special_from_date', ['lteq' => date('Y-m-d')]);
$productCollection->addAttributeToFilter('special_to_date', ['gteq' => date('Y-m-d')]);
```

### Advanced Filtering Techniques

#### **Complex OR Conditions**
```php
// Method 1: Using array of conditions
$collection->addFieldToFilter([
    ['attribute' => 'status', 'eq' => 1],
    ['attribute' => 'featured', 'eq' => 1]
]); // status = 1 OR featured = 1

// Method 2: Manual WHERE clause
$collection->getSelect()->where(
    'status = 1 OR featured = 1 OR (price < 100 AND category_id IN (1,2,3))'
);

// Method 3: Building complex conditions
$conditions = [];
$conditions[] = $collection->getConnection()->prepareSqlCondition('status', ['eq' => 1]);
$conditions[] = $collection->getConnection()->prepareSqlCondition('featured', ['eq' => 1]);
$collection->getSelect()->where('(' . implode(' OR ', $conditions) . ')');
```

#### **Nested AND/OR Logic**
```php
class AdvancedFilterCollection
{
    public function addComplexFilter($filters)
    {
        // Example: (status = 1 AND price > 100) OR (featured = 1 AND category_id IN (1,2))
        $andConditions1 = [
            $this->connection->prepareSqlCondition('status', ['eq' => 1]),
            $this->connection->prepareSqlCondition('price', ['gt' => 100])
        ];

        $andConditions2 = [
            $this->connection->prepareSqlCondition('featured', ['eq' => 1]),
            $this->connection->prepareSqlCondition('category_id', ['in' => [1, 2]])
        ];

        $orCondition = '(' . implode(' AND ', $andConditions1) . ') OR (' . implode(' AND ', $andConditions2) . ')';
        
        $this->collection->getSelect()->where($orCondition);
        return $this;
    }

    public function addDynamicFilters(array $filterGroups)
    {
        $groupConditions = [];
        
        foreach ($filterGroups as $group) {
            $groupFilters = [];
            foreach ($group['filters'] as $filter) {
                $groupFilters[] = $this->connection->prepareSqlCondition(
                    $filter['field'], 
                    [$filter['condition'] => $filter['value']]
                );
            }
            
            $operator = $group['operator'] ?? 'AND';
            $groupConditions[] = '(' . implode(' ' . $operator . ' ', $groupFilters) . ')';
        }
        
        $finalCondition = implode(' OR ', $groupConditions);
        $this->collection->getSelect()->where($finalCondition);
        
        return $this;
    }
}

// Usage
$filters = [
    [
        'operator' => 'AND',
        'filters' => [
            ['field' => 'status', 'condition' => 'eq', 'value' => 1],
            ['field' => 'price', 'condition' => 'gt', 'value' => 100]
        ]
    ],
    [
        'operator' => 'AND', 
        'filters' => [
            ['field' => 'featured', 'condition' => 'eq', 'value' => 1],
            ['field' => 'category_id', 'condition' => 'in', 'value' => [1, 2, 3]]
        ]
    ]
];

$advancedCollection = new AdvancedFilterCollection();
$advancedCollection->addDynamicFilters($filters);
```

#### **Subquery Filtering**
```php
// Filter products that have orders in the last 30 days
$orderSubQuery = $collection->getConnection()->select()
    ->from(['order_item' => $collection->getTable('sales_order_item')], ['product_id'])
    ->joinInner(
        ['order' => $collection->getTable('sales_order')],
        'order_item.order_id = order.entity_id',
        []
    )
    ->where('order.created_at >= ?', date('Y-m-d', strtotime('-30 days')))
    ->where('order.state = ?', 'complete')
    ->group('order_item.product_id');

$collection->addFieldToFilter('entity_id', ['in' => $orderSubQuery]);

// Filter customers who haven't placed orders
$customerOrderSubQuery = $collection->getConnection()->select()
    ->from($collection->getTable('sales_order'), ['customer_id'])
    ->where('customer_id IS NOT NULL')
    ->group('customer_id');

$collection->addFieldToFilter('entity_id', ['nin' => $customerOrderSubQuery]);

// EXISTS condition
$existsSubQuery = $collection->getConnection()->select()
    ->from($collection->getTable('catalog_category_product'), '1')
    ->where('catalog_category_product.product_id = main_table.entity_id')
    ->where('catalog_category_product.category_id = ?', 5);

$collection->getSelect()->where('EXISTS (?)', $existsSubQuery);
```

#### **Custom Field Filtering**
```php
class CustomFilterCollection
{
    public function addPriceRangeFilter($minPrice, $maxPrice, $includeTax = false)
    {
        if ($includeTax) {
            // Filter by price including tax
            $this->collection->getSelect()->where(
                'price_index.final_price * (1 + tax_rate.rate/100) BETWEEN ? AND ?',
                $minPrice,
                $maxPrice
            );
        } else {
            $this->collection->addFieldToFilter('price', ['from' => $minPrice, 'to' => $maxPrice]);
        }
        
        return $this;
    }

    public function addInventoryFilter($inStock = true, $minQty = 1)
    {
        $this->collection->joinTable(
            'cataloginventory_stock_item',
            'product_id=entity_id',
            ['stock_qty' => 'qty', 'stock_status' => 'is_in_stock']
        );

        if ($inStock) {
            $this->collection->addFieldToFilter('stock_status', 1);
            $this->collection->addFieldToFilter('stock_qty', ['gteq' => $minQty]);
        } else {
            $this->collection->addFieldToFilter([
                ['attribute' => 'stock_status', 'eq' => 0],
                ['attribute' => 'stock_qty', 'lt' => $minQty]
            ]);
        }

        return $this;
    }

    public function addSearchFilter($searchTerm, $searchFields = ['name', 'description', 'sku'])
    {
        if (empty($searchTerm)) {
            return $this;
        }

        $conditions = [];
        foreach ($searchFields as $field) {
            $conditions[] = $this->collection->getConnection()->prepareSqlCondition(
                $field,
                ['like' => '%' . $searchTerm . '%']
            );
        }

        $this->collection->getSelect()->where('(' . implode(' OR ', $conditions) . ')');
        return $this;
    }

    public function addDateRangeFilter($field, $fromDate = null, $toDate = null, $includeTime = false)
    {
        if (!$fromDate && !$toDate) {
            return $this;
        }

        if ($fromDate && !$includeTime) {
            $fromDate .= ' 00:00:00';
        }
        
        if ($toDate && !$includeTime) {
            $toDate .= ' 23:59:59';
        }

        if ($fromDate && $toDate) {
            $this->collection->addFieldToFilter($field, ['from' => $fromDate, 'to' => $toDate]);
        } elseif ($fromDate) {
            $this->collection->addFieldToFilter($field, ['gteq' => $fromDate]);
        } elseif ($toDate) {
            $this->collection->addFieldToFilter($field, ['lteq' => $toDate]);
        }

        return $this;
    }
}
```

### Store and Website Filtering

```php
// Store-specific filtering
$collection->addStoreFilter($storeId);

// Multiple stores
$collection->addStoreFilter([$storeId1, $storeId2]);

// Current store
$collection->addStoreFilter($this->storeManager->getStore()->getId());

// Website filtering
$websiteId = $this->storeManager->getWebsite()->getId();
$storeIds = $this->storeManager->getWebsite($websiteId)->getStoreIds();
$collection->addStoreFilter($storeIds);

// Global scope (admin store)
$collection->addStoreFilter(0);
```

## Sorting Operations

Sorting determines the order in which collection items are returned. Proper sorting is crucial for user experience and performance.

### Basic Sorting Methods

#### **setOrder($field, $direction)**
Primary sorting method:

```php
// Single field sorting
$collection->setOrder('created_at', 'DESC');
$collection->setOrder('name', 'ASC');
$collection->setOrder('price', \Magento\Framework\Data\Collection::SORT_ORDER_DESC);

// Sort by position (common for categories)
$collection->setOrder('position', 'ASC');

// Sort by multiple criteria (later calls don't override)
$collection->setOrder('status', 'DESC')
          ->setOrder('priority', 'DESC')
          ->setOrder('name', 'ASC');
```

#### **addOrder($field, $direction)**
Add additional sorting criteria:

```php
// Primary sort by status, secondary by name
$collection->addOrder('status', 'DESC');
$collection->addOrder('name', 'ASC');

// Using constants
$collection->addOrder('created_at', \Magento\Framework\Data\Collection::SORT_ORDER_DESC);
$collection->addOrder('updated_at', \Magento\Framework\Data\Collection::SORT_ORDER_ASC);
```

#### **unshiftOrder($field, $direction)**
Add sorting at the beginning (highest priority):

```php
$collection->addOrder('name', 'ASC');
$collection->addOrder('price', 'DESC');

// This will become the primary sort
$collection->unshiftOrder('featured', 'DESC');
// Final order: featured DESC, name ASC, price DESC
```

### Advanced Sorting Techniques

#### **Custom Sort Expressions**
```php
// Sort by calculated field
$collection->getSelect()->order(new \Zend_Db_Expr('(price * discount_percent / 100) DESC'));

// Sort by CASE expression
$collection->getSelect()->order(new \Zend_Db_Expr(
    'CASE 
        WHEN status = 1 THEN 1 
        WHEN featured = 1 THEN 2 
        ELSE 3 
    END ASC'
));

// Sort by custom calculation
$collection->getSelect()->order(new \Zend_Db_Expr(
    'CASE 
        WHEN special_price IS NOT NULL AND special_from_date <= NOW() AND special_to_date >= NOW() 
        THEN special_price 
        ELSE price 
    END ASC'
));

// Sort by distance (for store locator)
$userLat = 40.7128;
$userLng = -74.0060;
$collection->getSelect()->order(new \Zend_Db_Expr(
    "(6371 * acos(cos(radians($userLat)) * cos(radians(latitude)) * cos(radians(longitude) - radians($userLng)) + sin(radians($userLat)) * sin(radians(latitude)))) ASC"
));
```

#### **Conditional Sorting**
```php
class ConditionalSortCollection
{
    public function addSmartSort($sortBy, $direction = 'ASC')
    {
        switch ($sortBy) {
            case 'popularity':
                $this->addPopularitySort($direction);
                break;
            case 'relevance':
                $this->addRelevanceSort($direction);
                break;
            case 'price_low_high':
                $this->addPriceSort('ASC');
                break;
            case 'price_high_low':
                $this->addPriceSort('DESC');
                break;
            case 'newest':
                $this->collection->setOrder('created_at', 'DESC');
                break;
            case 'alphabetical':
                $this->collection->setOrder('name', 'ASC');
                break;
            default:
                $this->collection->setOrder('position', 'ASC');
        }
        
        return $this;
    }

    private function addPopularitySort($direction)
    {
        // Join with sales data
        $salesSubQuery = $this->collection->getConnection()->select()
            ->from(
                ['sales' => $this->collection->getTable('sales_order_item')],
                ['product_id', 'total_sold' => 'SUM(qty_ordered)']
            )
            ->group('product_id');

        $this->collection->getSelect()->joinLeft(
            ['popularity' => $salesSubQuery],
            'main_table.entity_id = popularity.product_id',
            ['total_sold' => 'COALESCE(popularity.total_sold, 0)']
        );

        $this->collection->getSelect()->order('total_sold ' . $direction);
        return $this;
    }

    private function addRelevanceSort($searchTerm)
    {
        if (empty($searchTerm)) {
            return $this->collection->setOrder('position', 'ASC');
        }

        // Calculate relevance score
        $relevanceScore = new \Zend_Db_Expr(
            "(CASE 
                WHEN name LIKE '%{$searchTerm}%' THEN 10
                WHEN sku LIKE '%{$searchTerm}%' THEN 8
                WHEN description LIKE '%{$searchTerm}%' THEN 5
                ELSE 0
            END + 
            CASE 
                WHEN name LIKE '{$searchTerm}%' THEN 5
                WHEN sku LIKE '{$searchTerm}%' THEN 3
                ELSE 0
            END)"
        );

        $this->collection->getSelect()->columns(['relevance' => $relevanceScore]);
        $this->collection->getSelect()->order('relevance DESC');
        
        return $this;
    }

    private function addPriceSort($direction)
    {
        // Sort by effective price (considering special prices)
        $effectivePrice = new \Zend_Db_Expr(
            'CASE 
                WHEN special_price IS NOT NULL 
                AND (special_from_date IS NULL OR special_from_date <= NOW()) 
                AND (special_to_date IS NULL OR special_to_date >= NOW())
                THEN special_price 
                ELSE price 
            END'
        );

        $this->collection->getSelect()->columns(['effective_price' => $effectivePrice]);
        $this->collection->getSelect()->order('effective_price ' . $direction);
        
        return $this;
    }
}
```

#### **Multi-Level Sorting with Priorities**
```php
class PrioritySortCollection
{
    private $sortCriteria = [];

    public function addSortCriteria($field, $direction, $priority = 0)
    {
        $this->sortCriteria[] = [
            'field' => $field,
            'direction' => $direction,
            'priority' => $priority
        ];
        
        return $this;
    }

    public function applySorting()
    {
        // Sort criteria by priority
        usort($this->sortCriteria, function($a, $b) {
            return $b['priority'] - $a['priority'];
        });

        // Apply sorting in priority order
        foreach ($this->sortCriteria as $criteria) {
            $this->collection->addOrder($criteria['field'], $criteria['direction']);
        }

        return $this;
    }

    public function addBusinessLogicSort()
    {
        // Business rules for product sorting
        $this->addSortCriteria('is_featured', 'DESC', 100); // Featured first
        $this->addSortCriteria('in_stock', 'DESC', 90);     // In stock before out of stock
        $this->addSortCriteria('has_discount', 'DESC', 80); // Discounted items next
        $this->addSortCriteria('rating', 'DESC', 70);       // High rated items
        $this->addSortCriteria('created_at', 'DESC', 60);   // Newer items
        $this->addSortCriteria('position', 'ASC', 50);      // Default position
        
        return $this->applySorting();
    }
}
```

### EAV Attribute Sorting

```php
// Sort by EAV attributes
$productCollection->addAttributeToSort('name', 'ASC');
$productCollection->addAttributeToSort('price', 'DESC');
$productCollection->addAttributeToSort('created_at', 'DESC');

// Sort by custom attributes
$productCollection->addAttributeToSort('manufacturer', 'ASC');
$productCollection->addAttributeToSort('color', 'ASC');

// Sort by attribute with specific store scope
$productCollection->addAttributeToSort('name', 'ASC', $storeId);

// Complex EAV sorting
class EavSortCollection
{
    public function addAttributeSortWithFallback($attribute, $direction, $fallbackAttribute = 'name')
    {
        // Try to sort by primary attribute, fallback to secondary
        $this->collection->addAttributeToSort($attribute, $direction);
        $this->collection->addAttributeToSort($fallbackAttribute, 'ASC');
        
        return $this;
    }

    public function addMultiAttributeSort(array $attributes)
    {
        foreach ($attributes as $attribute => $direction) {
            $this->collection->addAttributeToSort($attribute, $direction);
        }
        
        return $this;
    }
}

// Usage
$eavSort = new EavSortCollection();
$eavSort->addMultiAttributeSort([
    'featured' => 'DESC',
    'price' => 'ASC', 
    'name' => 'ASC'
]);
```

## Data Manipulation

Data manipulation involves processing, transforming, and working with collection data after it's loaded.

### Data Extraction Methods

#### **Basic Data Retrieval**
```php
// Get all loaded items
$items = $collection->getItems();

// Get specific item by ID
$item = $collection->getItemById(123);

// Get first/last items
$firstItem = $collection->getFirstItem();
$lastItem = $collection->getLastItem();

// Get column values as array
$productIds = $collection->getColumnValues('entity_id');
$productNames = $collection->getColumnValues('name');
$productPrices = $collection->getColumnValues('price');

// Get data as arrays
$allData = $collection->getData();
$specificData = $collection->getData('name'); // Array of names

// Convert to option arrays
$optionArray = $collection->toOptionArray(); // [['value' => id, 'label' => name], ...]
$optionHash = $collection->toOptionHash();   // [id => name, ...]
```

#### **Advanced Data Extraction**
```php
class DataExtractionCollection
{
    public function extractGroupedData($groupField, $valueField)
    {
        $groupedData = [];
        
        foreach ($this->collection as $item) {
            $groupKey = $item->getData($groupField);
            $value = $item->getData($valueField);
            
            if (!isset($groupedData[$groupKey])) {
                $groupedData[$groupKey] = [];
            }
            
            $groupedData[$groupKey][] = $value;
        }
        
        return $groupedData;
    }

    public function extractStatistics()
    {
        $this->collection->load();
        
        $stats = [
            'total_count' => $this->collection->count(),
            'total_value' => 0,
            'average_value' => 0,
            'min_value' => null,
            'max_value' => null,
            'unique_values' => []
        ];

        $values = [];
        foreach ($this->collection as $item) {
            $value = (float) $item->getData('price');
            $values[] = $value;
            $stats['total_value'] += $value;
            $stats['unique_values'][] = $item->getData('category_id');
        }

        if (count($values) > 0) {
            $stats['average_value'] = $stats['total_value'] / count($values);
            $stats['min_value'] = min($values);
            $stats['max_value'] = max($values);
        }

        $stats['unique_values'] = array_unique($stats['unique_values']);
        
        return $stats;
    }

    public function extractHierarchicalData($parentField = 'parent_id', $nameField = 'name')
    {
        $flatData = [];
        $hierarchical = [];

        // First pass: collect all items
        foreach ($this->collection as $item) {
            $flatData[$item->getId()] = [
                'id' => $item->getId(),
                'parent_id' => $item->getData($parentField),
                'name' => $item->getData($nameField),
                'data' => $item->getData(),
                'children' => []
            ];
        }

        // Second pass: build hierarchy
        foreach ($flatData as $id => $item) {
            if ($item['parent_id'] && isset($flatData[$item['parent_id']])) {
                $flatData[$item['parent_id']]['children'][$id] = &$flatData[$id];
            } else {
                $hierarchical[$id] = &$flatData[$id];
            }
        }

        return $hierarchical;
    }
}
```

### Data Transformation

#### **In-Memory Data Processing**
```php
class DataTransformationCollection
{
    public function transformPrices($currencyRate = 1.0, $taxRate = 0.0)
    {
        foreach ($this->collection as $item) {
            $originalPrice = (float) $item->getData('price');
            
            // Apply currency conversion
            $convertedPrice = $originalPrice * $currencyRate;
            
            // Apply tax
            $finalPrice = $convertedPrice * (1 + $taxRate);
            
            // Set transformed data
            $item->setData('original_price', $originalPrice);
            $item->setData('converted_price', $convertedPrice);
            $item->setData('final_price', $finalPrice);
            $item->setData('currency_rate', $currencyRate);
            $item->setData('tax_rate', $taxRate);
        }
        
        return $this;
    }

    public function addCalculatedFields()
    {
        foreach ($this->collection as $item) {
            // Calculate discount percentage
            $price = (float) $item->getData('price');
            $specialPrice = (float) $item->getData('special_price');
            
            if ($specialPrice > 0 && $specialPrice < $price) {
                $discountPercent = round((($price - $specialPrice) / $price) * 100, 2);
                $item->setData('discount_percent', $discountPercent);
                $item->setData('has_discount', true);
                $item->setData('savings_amount', $price - $specialPrice);
            } else {
                $item->setData('discount_percent', 0);
                $item->setData('has_discount', false);
                $item->setData('savings_amount', 0);
            }

            // Calculate price per unit
            $weight = (float) $item->getData('weight');
            if ($weight > 0) {
                $pricePerUnit = $price / $weight;
                $item->setData('price_per_unit', $pricePerUnit);
            }

            // Format dates
            $createdAt = $item->getData('created_at');
            if ($createdAt) {
                $item->setData('created_at_formatted', date('M j, Y', strtotime($createdAt)));
                $item->setData('days_since_created', floor((time() - strtotime($createdAt)) / 86400));
            }
        }
        
        return $this;
    }

    public function enrichWithExternalData(array $externalData)
    {
        foreach ($this->collection as $item) {
            $itemId = $item->getId();
            
            if (isset($externalData[$itemId])) {
                $enrichmentData = $externalData[$itemId];
                
                foreach ($enrichmentData as $key => $value) {
                    $item->setData('external_' . $key, $value);
                }
            }
        }
        
        return $this;
    }

    public function filterAndTransform(callable $filter, callable $transformer)
    {
        $filteredItems = [];
        
        foreach ($this->collection as $item) {
            if ($filter($item)) {
                $transformedItem = $transformer($item);
                $filteredItems[] = $transformedItem;
            }
        }
        
        // Replace collection items
        $this->collection->removeAllItems();
        foreach ($filteredItems as $item) {
            $this->collection->addItem($item);
        }
        
        return $this;
    }
}

// Usage examples
$transformer = new DataTransformationCollection();

// Transform prices with 15% tax and 1.2 EUR rate
$transformer->transformPrices(1.2, 0.15);

// Add calculated fields
$transformer->addCalculatedFields();

// Filter products with discount and add formatted data
$transformer->filterAndTransform(
    function($item) {
        return $item->getData('has_discount') === true;
    },
    function($item) {
        $item->setData('discount_label', 'Save ' . $item->getData('discount_percent') . '%');
        return $item;
    }
);
```

#### **Batch Processing and Aggregation**
```php
class BatchProcessingCollection
{
    public function processInBatches($batchSize = 100, callable $processor = null)
    {
        $processedItems = [];
        $batch = [];
        $batchNumber = 1;

        foreach ($this->collection as $item) {
            $batch[] = $item;
            
            if (count($batch) >= $batchSize) {
                if ($processor) {
                    $processedBatch = $processor($batch, $batchNumber);
                    $processedItems = array_merge($processedItems, $processedBatch);
                }
                
                $batch = [];
                $batchNumber++;
            }
        }

        // Process remaining items
        if (!empty($batch) && $processor) {
            $processedBatch = $processor($batch, $batchNumber);
            $processedItems = array_merge($processedItems, $processedBatch);
        }

        return $processedItems;
    }

    public function aggregateData($groupByField, $aggregateFields = [])
    {
        $aggregated = [];

        foreach ($this->collection as $item) {
            $groupKey = $item->getData($groupByField);
            
            if (!isset($aggregated[$groupKey])) {
                $aggregated[$groupKey] = [
                    'group_key' => $groupKey,
                    'count' => 0,
                    'items' => []
                ];
                
                // Initialize aggregate fields
                foreach ($aggregateFields as $field => $operations) {
                    foreach ($operations as $operation) {
                        $aggregated[$groupKey][$field . '_' . $operation] = 0;
                    }
                }
            }

            $aggregated[$groupKey]['count']++;
            $aggregated[$groupKey]['items'][] = $item;

            // Calculate aggregations
            foreach ($aggregateFields as $field => $operations) {
                $value = (float) $item->getData($field);
                
                foreach ($operations as $operation) {
                    $key = $field . '_' . $operation;
                    
                    switch ($operation) {
                        case 'sum':
                            $aggregated[$groupKey][$key] += $value;
                            break;
                        case 'min':
                            $aggregated[$groupKey][$key] = isset($aggregated[$groupKey][$key]) 
                                ? min($aggregated[$groupKey][$key], $value) 
                                : $value;
                            break;
                        case 'max':
                            $aggregated[$groupKey][$key] = isset($aggregated[$groupKey][$key]) 
                                ? max($aggregated[$groupKey][$key], $value) 
                                : $value;
                            break;
                    }
                }
            }
        }

        // Calculate averages
        foreach ($aggregated as &$group) {
            foreach ($aggregateFields as $field => $operations) {
                if (in_array('avg', $operations)) {
                    $sumKey = $field . '_sum';
                    $avgKey = $field . '_avg';
                    $group[$avgKey] = $group['count'] > 0 ? $group[$sumKey] / $group['count'] : 0;
                }
            }
        }

        return $aggregated;
    }

    public function exportToFormat($format = 'array', $selectedFields = [])
    {
        $data = [];
        
        foreach ($this->collection as $item) {
            $itemData = [];
            
            if (empty($selectedFields)) {
                $itemData = $item->getData();
            } else {
                foreach ($selectedFields as $field) {
                    $itemData[$field] = $item->getData($field);
                }
            }
            
            $data[] = $itemData;
        }

        switch ($format) {
            case 'json':
                return json_encode($data, JSON_PRETTY_PRINT);
                
            case 'csv':
                if (empty($data)) {
                    return '';
                }
                
                $output = fopen('php://temp', 'r+');
                
                // Write header
                fputcsv($output, array_keys($data[0]));
                
                // Write data
                foreach ($data as $row) {
                    fputcsv($output, $row);
                }
                
                rewind($output);
                $csv = stream_get_contents($output);
                fclose($output);
                
                return $csv;
                
            case 'xml':
                $xml = new \SimpleXMLElement('<items/>');
                
                foreach ($data as $item) {
                    $xmlItem = $xml->addChild('item');
                    foreach ($item as $key => $value) {
                        $xmlItem->addChild($key, htmlspecialchars($value));
                    }
                }
                
                return $xml->asXML();
                
            default:
                return $data;
        }
    }
}

// Usage
$processor = new BatchProcessingCollection();

// Process in batches of 50
$results = $processor->processInBatches(50, function($batch, $batchNumber) {
    // Process each batch
    foreach ($batch as $item) {
        // Update item data
        $item->setData('batch_number', $batchNumber);
        $item->setData('processed_at', date('Y-m-d H:i:s'));
    }
    return $batch;
});

// Aggregate sales data by category
$aggregated = $processor->aggregateData('category_id', [
    'price' => ['sum', 'avg', 'min', 'max'],
    'qty' => ['sum', 'avg']
]);

// Export to different formats
$jsonData = $processor->exportToFormat('json', ['entity_id', 'name', 'price']);
$csvData = $processor->exportToFormat('csv');
```

## Advanced Operations

Advanced operations combine multiple techniques to create sophisticated data handling capabilities.

### Multi-Collection Operations

#### **Collection Merging and Comparison**
```php
class MultiCollectionOperations
{
    public function mergeCollections(array $collections, $sortField = null)
    {
        $mergedItems = [];
        
        foreach ($collections as $collection) {
            foreach ($collection as $item) {
                $mergedItems[] = $item;
            }
        }
        
        // Sort merged items if specified
        if ($sortField) {
            usort($mergedItems, function($a, $b) use ($sortField) {
                return strcmp($a->getData($sortField), $b->getData($sortField));
            });
        }
        
        return $mergedItems;
    }

    public function findIntersection($collection1, $collection2, $compareField = 'entity_id')
    {
        $intersection = [];
        $collection2Ids = [];
        
        // Build lookup array for collection2
        foreach ($collection2 as $item) {
            $collection2Ids[$item->getData($compareField)] = $item;
        }
        
        // Find items that exist in both collections
        foreach ($collection1 as $item) {
            $id = $item->getData($compareField);
            if (isset($collection2Ids[$id])) {
                $intersection[] = $item;
            }
        }
        
        return $intersection;
    }

    public function findDifference($collection1, $collection2, $compareField = 'entity_id')
    {
        $difference = [];
        $collection2Ids = [];
        
        foreach ($collection2 as $item) {
            $collection2Ids[] = $item->getData($compareField);
        }
        
        foreach ($collection1 as $item) {
            if (!in_array($item->getData($compareField), $collection2Ids)) {
                $difference[] = $item;
            }
        }
        
        return $difference;
    }

    public function unionCollections($collection1, $collection2, $compareField = 'entity_id')
    {
        $union = [];
        $seenIds = [];
        
        // Add all items from collection1
        foreach ($collection1 as $item) {
            $id = $item->getData($compareField);
            if (!in_array($id, $seenIds)) {
                $union[] = $item;
                $seenIds[] = $id;
            }
        }
        
        // Add unique items from collection2
        foreach ($collection2 as $item) {
            $id = $item->getData($compareField);
            if (!in_array($id, $seenIds)) {
                $union[] = $item;
                $seenIds[] = $id;
            }
        }
        
        return $union;
    }
}
```

#### **Dynamic Collection Building**
```php
class DynamicCollectionBuilder
{
    private $baseCollection;
    private $filters = [];
    private $sorts = [];
    private $joins = [];

    public function __construct($collectionFactory)
    {
        $this->baseCollection = $collectionFactory->create();
    }

    public function addDynamicFilter($field, $operator, $value)
    {
        $this->filters[] = [
            'field' => $field,
            'operator' => $operator,
            'value' => $value
        ];
        
        return $this;
    }

    public function addDynamicSort($field, $direction)
    {
        $this->sorts[] = [
            'field' => $field,
            'direction' => $direction
        ];
        
        return $this;
    }

    public function addDynamicJoin($table, $condition, $columns = [])
    {
        $this->joins[] = [
            'table' => $table,
            'condition' => $condition,
            'columns' => $columns
        ];
        
        return $this;
    }

    public function build()
    {
        // Apply joins
        foreach ($this->joins as $join) {
            $this->baseCollection->getSelect()->joinLeft(
                $join['table'],
                $join['condition'],
                $join['columns']
            );
        }

        // Apply filters
        foreach ($this->filters as $filter) {
            $condition = [$filter['operator'] => $filter['value']];
            $this->baseCollection->addFieldToFilter($filter['field'], $condition);
        }

        // Apply sorting
        foreach ($this->sorts as $sort) {
            $this->baseCollection->addOrder($sort['field'], $sort['direction']);
        }

        return $this->baseCollection;
    }

    public function buildFromConfig(array $config)
    {
        // Filters
        if (isset($config['filters'])) {
            foreach ($config['filters'] as $filter) {
                $this->addDynamicFilter(
                    $filter['field'],
                    $filter['operator'],
                    $filter['value']
                );
            }
        }

        // Sorting
        if (isset($config['sorts'])) {
            foreach ($config['sorts'] as $sort) {
                $this->addDynamicSort($sort['field'], $sort['direction']);
            }
        }

        // Joins
        if (isset($config['joins'])) {
            foreach ($config['joins'] as $join) {
                $this->addDynamicJoin(
                    $join['table'],
                    $join['condition'],
                    $join['columns'] ?? []
                );
            }
        }

        return $this->build();
    }
}

// Usage
$builder = new DynamicCollectionBuilder($this->productCollectionFactory);

$config = [
    'filters' => [
        ['field' => 'status', 'operator' => 'eq', 'value' => 1],
        ['field' => 'price', 'operator' => 'gt', 'value' => 100]
    ],
    'sorts' => [
        ['field' => 'created_at', 'direction' => 'DESC']
    ],
    'joins' => [
        [
            'table' => ['inventory' => 'cataloginventory_stock_item'],
            'condition' => 'main_table.entity_id = inventory.product_id',
            'columns' => ['stock_qty' => 'qty']
        ]
    ]
];

$collection = $builder->buildFromConfig($config);
```

## Performance Optimization

### Query Performance

#### **Index Optimization Hints**
```php
class OptimizedCollection
{
    public function addIndexHints(array $indexes)
    {
        foreach ($indexes as $tableAlias => $indexName) {
            $this->collection->getSelect()->forceIndex($indexName, $tableAlias);
        }
        
        return $this;
    }

    public function optimizeForLargeDataset()
    {
        // Use covering indexes when possible
        $this->collection->addFieldToSelect(['entity_id', 'name', 'price']); // Only needed fields
        
        // Force use of specific index
        $this->collection->getSelect()->forceIndex('IDX_STATUS_CREATED_AT');
        
        // Use EXISTS instead of JOIN for large tables
        $subQuery = $this->collection->getConnection()->select()
            ->from($this->collection->getTable('catalog_category_product'), '1')
            ->where('catalog_category_product.product_id = main_table.entity_id')
            ->where('catalog_category_product.category_id = ?', 5);
            
        $this->collection->getSelect()->where('EXISTS (?)', $subQuery);
        
        return $this;
    }

    public function addQueryOptimizations()
    {
        // Limit result set early
        $this->collection->getSelect()->limit(1000);
        
        // Use specific column selection
        $this->collection->getSelect()->reset(\Magento\Framework\DB\Select::COLUMNS);
        $this->collection->getSelect()->columns([
            'entity_id',
            'name',
            'price',
            'status'
        ]);
        
        // Avoid filesort when possible
        $this->collection->getSelect()->order('entity_id ASC'); // Uses primary key index
        
        return $this;
    }
}
```

#### **Memory Management**
```php
class MemoryOptimizedCollection
{
    public function processLargeCollection($processor, $batchSize = 1000)
    {
        $collection = $this->collection;
        $collection->setPageSize($batchSize);
        
        $page = 1;
        $totalProcessed = 0;
        
        do {
            $collection->setCurPage($page);
            $collection->clear(); // Free previous page memory
            $collection->load();
            
            $itemsInBatch = 0;
            foreach ($collection as $item) {
                $processor($item);
                $totalProcessed++;
                $itemsInBatch++;
                
                // Unset item to free memory immediately
                unset($item);
            }
            
            // Force garbage collection periodically
            if ($page % 10 === 0) {
                gc_collect_cycles();
            }
            
            $page++;
            
        } while ($itemsInBatch === $batchSize);
        
        return $totalProcessed;
    }

    public function getMemoryEfficientIterator($batchSize = 1000)
    {
        return new class($this->collection, $batchSize) implements \Iterator {
            private $collection;
            private $batchSize;
            private $currentPage = 1;
            private $currentIndex = 0;
            private $items = [];
            private $totalProcessed = 0;

            public function __construct($collection, $batchSize) {
                $this->collection = $collection;
                $this->batchSize = $batchSize;
                $this->collection->setPageSize($batchSize);
            }

            public function rewind(): void {
                $this->currentPage = 1;
                $this->currentIndex = 0;
                $this->totalProcessed = 0;
                $this->loadBatch();
            }

            public function current() {
                return $this->items[$this->currentIndex] ?? null;
            }

            public function key() {
                return $this->totalProcessed;
            }

            public function next(): void {
                $this->currentIndex++;
                $this->totalProcessed++;

                if ($this->currentIndex >= count($this->items)) {
                    $this->currentPage++;
                    $this->currentIndex = 0;
                    $this->loadBatch();
                }
            }

            public function valid(): bool {
                return isset($this->items[$this->currentIndex]);
            }

            private function loadBatch(): void {
                $this->collection->setCurPage($this->currentPage);
                $this->collection->clear();
                $this->collection->load();
                $this->items = $this->collection->getItems();
            }
        };
    }
}
```

### Caching Strategies

#### **Result Caching**
```php
class CachedCollectionOperations
{
    private $cache;
    private $serializer;

    public function __construct($cache, $serializer)
    {
        $this->cache = $cache;
        $this->serializer = $serializer;
    }

    public function getCachedCollection($cacheKey, $ttl = 3600)
    {
        $cachedData = $this->cache->load($cacheKey);
        
        if ($cachedData) {
            $data = $this->serializer->unserialize($cachedData);
            $this->populateCollectionFromCache($data);
            return $this->collection;
        }
        
        // Load and cache
        $this->collection->load();
        $this->cacheCollectionData($cacheKey, $ttl);
        
        return $this->collection;
    }

    public function getCachedAggregates($cacheKey, $aggregateFunction, $ttl = 1800)
    {
        $cachedResult = $this->cache->load($cacheKey);
        
        if ($cachedResult !== false) {
            return $this->serializer->unserialize($cachedResult);
        }
        
        $result = $aggregateFunction($this->collection);
        
        $this->cache->save(
            $this->serializer->serialize($result),
            $cacheKey,
            ['collection_aggregates'],
            $ttl
        );
        
        return $result;
    }

    private function cacheCollectionData($cacheKey, $ttl)
    {
        $data = [];
        foreach ($this->collection as $item) {
            $data[] = $item->getData();
        }
        
        $this->cache->save(
            $this->serializer->serialize($data),
            $cacheKey,
            ['collection_data'],
            $ttl
        );
    }

    private function populateCollectionFromCache($data)
    {
        $this->collection->removeAllItems();
        
        foreach ($data as $itemData) {
            $item = $this->collection->getNewEmptyItem();
            $item->setData($itemData);
            $this->collection->addItem($item);
        }
        
        $this->collection->setFlag('is_loaded', true);
    }
}
```

## Real-World Examples

### E-commerce Product Filtering System

```php
class ProductFilteringSystem
{
    private $productCollectionFactory;
    private $storeManager;

    public function __construct($productCollectionFactory, $storeManager)
    {
        $this->productCollectionFactory = $productCollectionFactory;
        $this->storeManager = $storeManager;
    }

    public function getFilteredProducts(array $filters = [])
    {
        $collection = $this->productCollectionFactory->create();
        
        // Basic product filters
        $collection->addAttributeToSelect(['name', 'price', 'special_price', 'image'])
                  ->addAttributeToFilter('status', 1)
                  ->addAttributeToFilter('visibility', ['neq' => 1])
                  ->addStoreFilter($this->storeManager->getStore()->getId());

        // Price range filter
        if (isset($filters['price_from']) || isset($filters['price_to'])) {
            $this->addPriceFilter($collection, $filters);
        }

        // Category filter
        if (isset($filters['category_id'])) {
            $this->addCategoryFilter($collection, $filters['category_id']);
        }

        // Attribute filters (color, size, brand, etc.)
        if (isset($filters['attributes'])) {
            $this->addAttributeFilters($collection, $filters['attributes']);
        }

        // Search term
        if (isset($filters['search']) && $filters['search']) {
            $this->addSearchFilter($collection, $filters['search']);
        }

        // Availability filter
        if (isset($filters['in_stock']) && $filters['in_stock']) {
            $this->addStockFilter($collection);
        }

        // Discount filter
        if (isset($filters['on_sale']) && $filters['on_sale']) {
            $this->addDiscountFilter($collection);
        }

        // Rating filter
        if (isset($filters['min_rating'])) {
            $this->addRatingFilter($collection, $filters['min_rating']);
        }

        // Sorting
        $this->applySorting($collection, $filters['sort_by'] ?? 'position');

        // Pagination
        if (isset($filters['page_size'])) {
            $collection->setPageSize($filters['page_size']);
        }
        if (isset($filters['current_page'])) {
            $collection->setCurPage($filters['current_page']);
        }

        return $collection;
    }

    private function addPriceFilter($collection, $filters)
    {
        $priceFilter = [];
        
        if (isset($filters['price_from']) && $filters['price_from'] > 0) {
            $priceFilter['from'] = $filters['price_from'];
        }
        
        if (isset($filters['price_to']) && $filters['price_to'] > 0) {
            $priceFilter['to'] = $filters['price_to'];
        }

        if (!empty($priceFilter)) {
            // Consider special prices in filtering
            $collection->getSelect()->where(
                'CASE 
                    WHEN at_special_price.value IS NOT NULL 
                    AND (at_special_from_date.value IS NULL OR at_special_from_date.value <= NOW()) 
                    AND (at_special_to_date.value IS NULL OR at_special_to_date.value >= NOW())
                    THEN at_special_price.value 
                    ELSE at_price.value 
                END BETWEEN ? AND ?',
                $priceFilter['from'] ?? 0,
                $priceFilter['to'] ?? 999999
            );
        }
    }

    private function addCategoryFilter($collection, $categoryIds)
    {
        if (!is_array($categoryIds)) {
            $categoryIds = [$categoryIds];
        }

        $collection->joinField(
            'category_id',
            'catalog_category_product',
            'category_id',
            'product_id=entity_id',
            null,
            'left'
        );

        $collection->addFieldToFilter('category_id', ['in' => $categoryIds]);
    }

    private function addAttributeFilters($collection, $attributes)
    {
        foreach ($attributes as $attributeCode => $values) {
            if (!is_array($values)) {
                $values = [$values];
            }
            
            $collection->addAttributeToFilter($attributeCode, ['in' => $values]);
        }
    }

    private function addSearchFilter($collection, $searchTerm)
    {
        $searchTerm = trim($searchTerm);
        
        if (strlen($searchTerm) < 3) {
            return; // Minimum search length
        }

        // Search in multiple fields with different weights
        $searchConditions = [
            $collection->getConnection()->prepareSqlCondition('at_name.value', ['like' => '%' . $searchTerm . '%']),
            $collection->getConnection()->prepareSqlCondition('at_description.value', ['like' => '%' . $searchTerm . '%']),
            $collection->getConnection()->prepareSqlCondition('sku', ['like' => '%' . $searchTerm . '%'])
        ];

        $collection->getSelect()->where('(' . implode(' OR ', $searchConditions) . ')');

        // Add relevance scoring
        $relevanceScore = new \Zend_Db_Expr(
            "(CASE 
                WHEN at_name.value LIKE '%{$searchTerm}%' THEN 10
                WHEN sku LIKE '%{$searchTerm}%' THEN 8
                WHEN at_description.value LIKE '%{$searchTerm}%' THEN 5
                ELSE 0
            END + 
            CASE 
                WHEN at_name.value LIKE '{$searchTerm}%' THEN 5
                WHEN sku LIKE '{$searchTerm}%' THEN 3
                ELSE 0
            END)"
        );

        $collection->getSelect()->columns(['relevance' => $relevanceScore]);
    }

    private function addStockFilter($collection)
    {
        $collection->joinTable(
            'cataloginventory_stock_item',
            'product_id=entity_id',
            ['stock_qty' => 'qty', 'is_in_stock'],
            null,
            'left'
        );

        $collection->addFieldToFilter('is_in_stock', 1);
    }

    private function addDiscountFilter($collection)
    {
        $collection->addAttributeToSelect(['special_price', 'special_from_date', 'special_to_date']);
        
        $collection->getSelect()->where(
            'at_special_price.value IS NOT NULL 
            AND at_special_price.value < at_price.value
            AND (at_special_from_date.value IS NULL OR at_special_from_date.value <= NOW()) 
            AND (at_special_to_date.value IS NULL OR at_special_to_date.value >= NOW())'
        );
    }

    private function addRatingFilter($collection, $minRating)
    {
        $collection->joinTable(
            'review_entity_summary',
            'entity_pk_value=entity_id',
            ['rating_summary', 'reviews_count'],
            '{{table}}.entity_type=1 AND {{table}}.store_id=' . $this->storeManager->getStore()->getId(),
            'left'
        );

        $collection->addFieldToFilter('rating_summary', ['gteq' => $minRating * 20]); // Rating summary is 0-100
    }

    private function applySorting($collection, $sortBy)
    {
        switch ($sortBy) {
            case 'price_low_high':
                $collection->getSelect()->order('final_price ASC');
                break;
            case 'price_high_low':
                $collection->getSelect()->order('final_price DESC');
                break;
            case 'name_asc':
                $collection->addAttributeToSort('name', 'ASC');
                break;
            case 'name_desc':
                $collection->addAttributeToSort('name', 'DESC');
                break;
            case 'newest':
                $collection->addAttributeToSort('created_at', 'DESC');
                break;
            case 'popularity':
                $this->addPopularitySort($collection);
                break;
            case 'rating':
                $collection->getSelect()->order('rating_summary DESC');
                break;
            case 'relevance':
                if ($collection->getSelect()->getPart('columns')) {
                    $collection->getSelect()->order('relevance DESC');
                }
                break;
            default:
                $collection->addAttributeToSort('position', 'ASC');
        }
    }

    private function addPopularitySort($collection)
    {
        $salesSubQuery = $collection->getConnection()->select()
            ->from(
                ['sales' => $collection->getTable('sales_order_item')],
                ['product_id', 'total_sold' => 'SUM(qty_ordered)']
            )
            ->joinInner(
                ['order' => $collection->getTable('sales_order')],
                'sales.order_id = order.entity_id',
                []
            )
            ->where('order.created_at >= ?', date('Y-m-d', strtotime('-90 days')))
            ->where('order.state = ?', 'complete')
            ->group('sales.product_id');

        $collection->getSelect()->joinLeft(
            ['popularity' => $salesSubQuery],
            'e.entity_id = popularity.product_id',
            ['total_sold' => 'COALESCE(popularity.total_sold, 0)']
        );

        $collection->getSelect()->order('total_sold DESC');
    }

    public function getFacetData($filters = [])
    {
        $baseCollection = $this->getFilteredProducts($filters);
        
        // Remove current filters to get all available options
        $facetCollection = clone $baseCollection;
        
        $facets = [
            'categories' => $this->getCategoryFacets($facetCollection),
            'price_ranges' => $this->getPriceFacets($facetCollection),
            'brands' => $this->getBrandFacets($facetCollection),
            'colors' => $this->getColorFacets($facetCollection),
            'sizes' => $this->getSizeFacets($facetCollection)
        ];
        
        return $facets;
    }

    private function getCategoryFacets($collection)
    {
        $facetCollection = clone $collection;
        $facetCollection->joinField(
            'category_id',
            'catalog_category_product',
            'category_id',
            'product_id=entity_id',
            null,
            'left'
        );
        
        $facetCollection->getSelect()
            ->reset(\Magento\Framework\DB\Select::COLUMNS)
            ->columns(['category_id', 'product_count' => 'COUNT(DISTINCT e.entity_id)'])
            ->group('category_id');
            
        $categoryFacets = [];
        foreach ($facetCollection as $item) {
            $categoryFacets[] = [
                'category_id' => $item->getCategoryId(),
                'count' => $item->getProductCount()
            ];
        }
        
        return $categoryFacets;
    }
}
```

### Customer Analytics Dashboard

```php
class CustomerAnalyticsDashboard
{
    public function getCustomerSegmentationData()
    {
        $collection = $this->customerCollectionFactory->create();
        
        // Add order statistics
        $this->addOrderStatistics($collection);
        
        // Add segmentation logic
        $this->addCustomerSegmentation($collection);
        
        // Load and process
        $collection->load();
        
        return $this->processSegmentationData($collection);
    }

    private function addOrderStatistics($collection)
    {
        $orderStatsSelect = $collection->getConnection()->select()
            ->from(
                ['orders' => $collection->getTable('sales_order')],
                [
                    'customer_id',
                    'order_count' => 'COUNT(*)',
                    'total_spent' => 'SUM(grand_total)',
                    'avg_order_value' => 'AVG(grand_total)',
                    'first_order_date' => 'MIN(created_at)',
                    'last_order_date' => 'MAX(created_at)',
                    'days_since_last_order' => 'DATEDIFF(NOW(), MAX(created_at))'
                ]
            )
            ->where('orders.state IN (?)', ['complete', 'processing'])
            ->group('customer_id');

        $collection->getSelect()->joinLeft(
            ['order_stats' => $orderStatsSelect],
            'e.entity_id = order_stats.customer_id',
            [
                'order_count' => 'COALESCE(order_stats.order_count, 0)',
                'total_spent' => 'COALESCE(order_stats.total_spent, 0)',
                'avg_order_value' => 'COALESCE(order_stats.avg_order_value, 0)',
                'first_order_date' => 'order_stats.first_order_date',
                'last_order_date' => 'order_stats.last_order_date',
                'days_since_last_order' => 'order_stats.days_since_last_order'
            ]
        );
    }

    private function addCustomerSegmentation($collection)
    {
        $collection->getSelect()->columns([
            'customer_segment' => new \Zend_Db_Expr(
                'CASE 
                    WHEN order_stats.total_spent > 1000 AND order_stats.days_since_last_order < 90 THEN "VIP Active"
                    WHEN order_stats.total_spent > 1000 AND order_stats.days_since_last_order >= 90 THEN "VIP Inactive"
                    WHEN order_stats.total_spent > 500 AND order_stats.days_since_last_order < 180 THEN "High Value"
                    WHEN order_stats.order_count > 5 AND order_stats.days_since_last_order < 180 THEN "Loyal"
                    WHEN order_stats.days_since_last_order > 365 THEN "Churned"
                    WHEN order_stats.order_count = 0 OR order_stats.order_count IS NULL THEN "Never Purchased"
                    ELSE "Regular"
                END'
            ),
            'customer_value_score' => new \Zend_Db_Expr(
                'CASE 
                    WHEN order_stats.total_spent IS NULL THEN 0
                    ELSE (
                        (order_stats.total_spent / 100) + 
                        (order_stats.order_count * 10) + 
                        CASE 
                            WHEN order_stats.days_since_last_order < 30 THEN 50
                            WHEN order_stats.days_since_last_order < 90 THEN 30
                            WHEN order_stats.days_since_last_order < 180 THEN 10
                            ELSE 0
                        END
                    )
                END'
            )
        ]);
    }

    private function processSegmentationData($collection)
    {
        $segmentData = [];
        $totalCustomers = 0;
        
        foreach ($collection as $customer) {
            $segment = $customer->getCustomerSegment();
            
            if (!isset($segmentData[$segment])) {
                $segmentData[$segment] = [
                    'count' => 0,
                    'total_value' => 0,
                    'avg_value' => 0,
                    'customers' => []
                ];
            }
            
            $segmentData[$segment]['count']++;
            $segmentData[$segment]['total_value'] += (float) $customer->getTotalSpent();
            $segmentData[$segment]['customers'][] = [
                'id' => $customer->getId(),
                'email' => $customer->getEmail(),
                'name' => $customer->getFirstname() . ' ' . $customer->getLastname(),
                'total_spent' => $customer->getTotalSpent(),
                'order_count' => $customer->getOrderCount(),
                'value_score' => $customer->getCustomerValueScore()
            ];
            
            $totalCustomers++;
        }
        
        // Calculate percentages and averages
        foreach ($segmentData as $segment => &$data) {
            $data['percentage'] = ($data['count'] / $totalCustomers) * 100;
            $data['avg_value'] = $data['count'] > 0 ? $data['total_value'] / $data['count'] : 0;
            
            // Sort customers by value score
            usort($data['customers'], function($a, $b) {
                return $b['value_score'] - $a['value_score'];
            });
        }
        
        return $segmentData;
    }
}
```

## Best Practices

### Performance Best Practices

1. **Select Only Needed Fields**
```php
// Good
$collection->addFieldToSelect(['entity_id', 'name', 'price']);

// Avoid
$collection->addFieldToSelect('*');
```

2. **Use Proper Pagination**
```php
// Good - for large datasets
$collection->setPageSize(100);
$collection->setCurPage($currentPage);

// Avoid - loading everything
$allItems = $collection->getItems();
```

3. **Optimize Joins**
```php
// Good - use LEFT JOIN when records might not exist
$collection->getSelect()->joinLeft(...);

// Good - use EXISTS for filtering
$subQuery = $connection->select()->from('related_table', '1')...;
$collection->getSelect()->where('EXISTS (?)', $subQuery);
```

4. **Cache Frequently Used Collections**
```php
$cacheKey = 'collection_' . md5($collection->getSelectSql(true));
$cachedData = $this->cache->load($cacheKey);

if (!$cachedData) {
    $collection->load();
    $this->cache->save($collection->getData(), $cacheKey, [], 3600);
}
```

### Code Organization Best Practices

1. **Use Factory Pattern**
```php
// Good
public function __construct(CollectionFactory $collectionFactory) {
    $this->collectionFactory = $collectionFactory;
}

$collection = $this->collectionFactory->create();
```

2. **Create Reusable Filter Methods**
```php
class ProductCollectionHelper 
{
    public function addActiveProductsFilter($collection) {
        return $collection->addAttributeToFilter('status', 1)
                         ->addAttributeToFilter('visibility', ['neq' => 1]);
    }
    
    public function addStockFilter($collection, $inStock = true) {
        // Reusable stock filtering logic
    }
}
```

3. **Use Method Chaining**
```php
$collection = $this->productCollectionFactory->create()
    ->addAttributeToSelect(['name', 'price'])
    ->addAttributeToFilter('status', 1)
    ->addStoreFilter($storeId)
    ->setOrder('position', 'ASC')
    ->setPageSize(20);
```

## Troubleshooting

### Common Issues

1. **Memory Issues**
```php
// Problem: Loading large collections
// Solution: Use pagination and batch processing
foreach ($this->getBatchIterator($collection, 1000) as $item) {
    // Process item
    unset($item); // Free memory
}
```

2. **Performance Issues**
```php
// Problem: N+1 queries
// Solution: Use proper joins
$collection->joinAttribute('category_name', 'category/name', 'category_id');

// Instead of loading category for each product in loop
```

3. **Incorrect Filtering**
```php
// Problem: EAV attribute not found
// Solution: Check if attribute exists and is enabled
$attribute = $collection->getResource()->getAttribute('custom_attr');
if ($attribute && $attribute->getId()) {
    $collection->addAttributeToFilter('custom_attr', $value);
}
```

4. **Query Debugging**
```php
// Debug collection queries
echo $collection->getSelectSql(true);

// Enable query logging
$collection->load(false, true);

// Check query execution plan
$explain = $connection->query('EXPLAIN ' . $collection->getSelectSql(true));
```

## Conclusion

Mastering collection operations in Magento 2 enables you to:

### **Core Capabilities:**
1. **Efficient Data Loading** - Lazy loading, batch processing, memory optimization
2. **Flexible Filtering** - Complex conditions, EAV attributes, subqueries
3. **Smart Sorting** - Multiple criteria, custom expressions, business logic
4. **Data Transformation** - In-memory processing, aggregation, format conversion

### **Key Benefits:**
- **Performance** - Optimized database queries and memory usage
- **Scalability** - Handle large datasets efficiently
- **Maintainability** - Clean, reusable code patterns
- **Flexibility** - Adapt to complex business requirements

### **Essential Techniques:**
- Use factory pattern for collection creation
- Implement proper pagination for large datasets
- Select only necessary fields to reduce memory usage
- Cache frequently accessed collection data
- Use batch processing for memory-intensive operations
- Implement proper error handling and debugging

Understanding these collection operations deeply allows you to build high-performance Magento 2 applications that can handle enterprise-scale data processing while maintaining optimal user experience and system resources.