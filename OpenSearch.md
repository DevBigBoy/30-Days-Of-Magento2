# Complete Guide: OpenSearch Integration with Magento 2

## Table of Contents
1. [Overview](#overview)
2. [Core Modules and Architecture](#core-modules-and-architecture)
3. [Data Flow and Indexing Process](#data-flow-and-indexing-process)
4. [Search Functionality](#search-functionality)
5. [Configuration and Setup](#configuration-and-setup)
6. [Customization and Extension](#customization-and-extension)
7. [Performance Optimization](#performance-optimization)
8. [Troubleshooting](#troubleshooting)

## Overview

OpenSearch serves as Magento 2's primary search engine, replacing the traditional MySQL-based search with a more powerful, scalable solution. It provides fast, relevant search results, advanced filtering capabilities, and sophisticated ranking algorithms.

### Key Benefits
- **Performance**: Sub-second search response times even for large catalogs
- **Relevance**: Advanced scoring algorithms for better search results
- **Scalability**: Handles millions of products efficiently
- **Flexibility**: Supports complex filtering and faceted search
- **Analytics**: Search analytics and reporting capabilities

## Core Modules and Architecture

### Primary Modules

#### 1. Magento_Elasticsearch Module
**Location**: `vendor/magento/module-elasticsearch`

**Purpose**: Core integration layer between Magento and OpenSearch/Elasticsearch

**Key Components**:
- `Model/Adapter/Elasticsearch.php` - Main adapter class
- `Model/Client/Elasticsearch.php` - Client connection management
- `SearchAdapter/` - Query translation and response handling
- `Model/Indexer/` - Indexing logic and data processors

#### 2. Magento_AdvancedSearch Module
**Location**: `vendor/magento/module-advanced-search`

**Purpose**: Provides advanced search configuration and management

**Key Components**:
- Search configuration interfaces
- Advanced search form handling
- Search suggestions and recommendations

#### 3. Magento_CatalogSearch Module
**Location**: `vendor/magento/module-catalog-search`

**Purpose**: Catalog-specific search functionality

**Key Components**:
- `Model/Indexer/Fulltext.php` - Product fulltext indexer
- `Model/Search/` - Search request builders
- `Block/` - Frontend search blocks and forms

#### 4. Magento_Search Module
**Location**: `vendor/magento/module-search`

**Purpose**: Core search framework and interfaces

**Key Components**:
- `Api/SearchInterface.php` - Search API contracts
- `Model/QueryFactory.php` - Search query management
- `Model/SearchEngine/` - Search engine abstractions

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Magento 2 Frontend                      │
│  ┌─────────────────┐    ┌─────────────────┐               │
│  │  Search Box     │    │  Category Page  │               │
│  │  Quick Search   │    │  Layered Nav    │               │
│  └─────────────────┘    └─────────────────┘               │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────┐
│             Magento 2 Application Layer                    │
│                             │                             │
│  ┌─────────────────────────┼───────────────────────────┐   │
│  │        Search Framework │                           │   │
│  │  ┌─────────────────────┼─────────────────────────┐ │   │
│  │  │   CatalogSearch     │                         │ │   │
│  │  │   - Query Builder   │                         │ │   │
│  │  │   - Request Handler │                         │ │   │
│  │  └─────────────────────┼─────────────────────────┘ │   │
│  │                        │                           │   │
│  │  ┌─────────────────────┼─────────────────────────┐ │   │
│  │  │   Elasticsearch     │                         │ │   │
│  │  │   - Adapter         │                         │ │   │
│  │  │   - Query Mapper    │                         │ │   │
│  │  │   - Response Parser │                         │ │   │
│  │  └─────────────────────┼─────────────────────────┘ │   │
│  └─────────────────────────┼───────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────┐
│                    Data Processing Layer                   │
│                             │                             │
│  ┌─────────────────────────┼───────────────────────────┐   │
│  │        Indexing         │                           │   │
│  │  ┌─────────────────────┼─────────────────────────┐ │   │
│  │  │   Product Indexer   │                         │ │   │
│  │  │   - Data Collection │                         │ │   │
│  │  │   - Data Mapping    │                         │ │   │
│  │  │   - Index Building  │                         │ │   │
│  │  └─────────────────────┼─────────────────────────┘ │   │
│  └─────────────────────────┼───────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────┐
│                      OpenSearch Cluster                    │
│                             │                             │
│  ┌─────────────────┐   ┌───┼────────────┐   ┌────────────┐ │
│  │   Index Node    │   │   │ Data Node  │   │ Data Node  │ │
│  │   - Routing     │   │   │ - Shards   │   │ - Shards   │ │
│  │   - Querying    │   │   │ - Storage  │   │ - Storage  │ │
│  └─────────────────┘   └───┼────────────┘   └────────────┘ │
└─────────────────────────────┼───────────────────────────────┘
                              │
                    ┌─────────┼─────────┐
                    │      Database     │
                    │   - Products      │
                    │   - Attributes    │
                    │   - Categories    │
                    └───────────────────┘
```

## Data Flow and Indexing Process

### Indexing Workflow

#### 1. Data Collection Phase
```php
// File: vendor/magento/module-catalog-search/Model/Indexer/Fulltext.php
class Fulltext implements \Magento\Framework\Indexer\ActionInterface
{
    public function executeFull()
    {
        // 1. Collect all product data
        $products = $this->getProductCollection();
        
        // 2. Process attributes and prepare data
        foreach ($products as $product) {
            $productData = $this->prepareProductData($product);
            $this->addToIndex($productData);
        }
        
        // 3. Send to OpenSearch
        $this->indexBuilder->build();
    }
}
```

#### 2. Data Mapping and Transformation
```php
// File: vendor/magento/module-elasticsearch/Model/Adapter/BatchDataMapper/ProductDataMapper.php
class ProductDataMapper
{
    public function map($documentData, $storeId, $context = [])
    {
        $documents = [];
        foreach ($documentData as $productId => $indexData) {
            $documents[$productId] = [
                'sku' => $indexData['sku'],
                'name' => $indexData['name'],
                'description' => $indexData['description'],
                'price' => $indexData['price'],
                'category_ids' => $indexData['category_ids'],
                'visibility' => $indexData['visibility'],
                'status' => $indexData['status'],
                // Custom attributes mapping
                'color' => $indexData['color'] ?? null,
                'size' => $indexData['size'] ?? null,
            ];
        }
        return $documents;
    }
}
```

#### 3. Index Structure Creation
```php
// File: vendor/magento/module-elasticsearch/Model/Adapter/Elasticsearch.php
public function prepareIndex($storeId, $mappings)
{
    $settings = [
        'analysis' => [
            'analyzer' => [
                'default' => [
                    'type' => 'standard',
                    'stopwords' => '_english_'
                ],
                'keyword_analyzer' => [
                    'type' => 'keyword',
                    'normalizer' => 'lowercase_normalizer'
                ]
            ]
        ]
    ];
    
    $indexName = $this->getIndexName($storeId);
    $this->client->createIndex($indexName, $settings, $mappings);
}
```

### Index Triggers

#### Automatic Reindexing
- **Product Save**: When a product is saved/updated
- **Category Changes**: When product categories are modified
- **Attribute Updates**: When searchable attributes change
- **Inventory Changes**: When stock status changes
- **Price Updates**: When product prices are modified

#### Manual Reindexing
```bash
# Full reindex
php bin/magento indexer:reindex catalogsearch_fulltext

# Specific store reindex
php bin/magento indexer:reindex catalogsearch_fulltext --store=1

# Check indexer status
php bin/magento indexer:status
```

## Search Functionality

### Query Processing Pipeline

#### 1. Request Building
```php
// File: vendor/magento/module-catalog-search/Model/Search/RequestGenerator.php
class RequestGenerator
{
    public function generate()
    {
        $request = [
            'index' => $this->getIndexName(),
            'type' => 'document',
            'body' => [
                'query' => $this->buildQuery(),
                'aggregations' => $this->buildAggregations(),
                'sort' => $this->buildSort(),
                'from' => $this->getFrom(),
                'size' => $this->getSize()
            ]
        ];
        return $request;
    }
}
```

#### 2. Query Types

**Full-Text Search**
```php
private function buildFullTextQuery($searchTerm)
{
    return [
        'bool' => [
            'must' => [
                'multi_match' => [
                    'query' => $searchTerm,
                    'fields' => [
                        'name^5',           // Higher boost for name
                        'sku^3',            // Medium boost for SKU
                        'description^1',    // Normal boost for description
                        'short_description^2'
                    ],
                    'type' => 'best_fields',
                    'tie_breaker' => 0.3
                ]
            ]
        ]
    ];
}
```

**Filtered Search**
```php
private function buildFilteredQuery($filters)
{
    $boolQuery = ['bool' => ['filter' => []]];
    
    foreach ($filters as $field => $values) {
        if (is_array($values)) {
            $boolQuery['bool']['filter'][] = [
                'terms' => [$field => $values]
            ];
        } else {
            $boolQuery['bool']['filter'][] = [
                'term' => [$field => $values]
            ];
        }
    }
    
    return $boolQuery;
}
```

#### 3. Aggregations (Faceted Search)
```php
private function buildAggregations()
{
    return [
        'category_bucket' => [
            'terms' => [
                'field' => 'category_ids',
                'size' => 100
            ]
        ],
        'price_bucket' => [
            'histogram' => [
                'field' => 'price',
                'interval' => 50
            ]
        ],
        'color_bucket' => [
            'terms' => [
                'field' => 'color',
                'size' => 50
            ]
        ]
    ];
}
```

### Search Response Processing

#### Response Parsing
```php
// File: vendor/magento/module-elasticsearch/SearchAdapter/ResponseFactory.php
class ResponseFactory
{
    public function create($rawResponse)
    {
        $documents = [];
        foreach ($rawResponse['hits']['hits'] as $hit) {
            $documents[] = new Document([
                'id' => $hit['_id'],
                'score' => $hit['_score'],
                'source' => $hit['_source']
            ]);
        }
        
        $aggregations = $this->parseAggregations($rawResponse['aggregations']);
        
        return new SearchResponse($documents, $aggregations);
    }
}
```

## Configuration and Setup

### Core Configuration Files

#### 1. Module Configuration
```xml
<!-- File: vendor/magento/module-elasticsearch/etc/config.xml -->
<config>
    <default>
        <catalog>
            <search>
                <engine>opensearch</engine>
                <opensearch_server_hostname>localhost</opensearch_server_hostname>
                <opensearch_server_port>9200</opensearch_server_port>
                <opensearch_index_prefix>magento2</opensearch_index_prefix>
                <opensearch_enable_auth>0</opensearch_enable_auth>
                <opensearch_username></opensearch_username>
                <opensearch_password></opensearch_password>
                <opensearch_server_timeout>15</opensearch_server_timeout>
            </search>
        </catalog>
    </default>
</config>
```

#### 2. Dependency Injection
```xml
<!-- File: vendor/magento/module-elasticsearch/etc/di.xml -->
<config>
    <type name="Magento\Search\Model\Adminhtml\System\Config\Source\Engine">
        <arguments>
            <argument name="engines" xsi:type="array">
                <item name="opensearch" xsi:type="string">OpenSearch</item>
            </argument>
        </arguments>
    </type>
    
    <type name="Magento\Elasticsearch\Model\Config">
        <arguments>
            <argument name="searchConfigData" xsi:type="array">
                <item name="opensearch_server_hostname" xsi:type="string">catalog/search/opensearch_server_hostname</item>
                <item name="opensearch_server_port" xsi:type="string">catalog/search/opensearch_server_port</item>
            </argument>
        </arguments>
    </type>
</config>
```

### Admin Configuration Options

#### Search Engine Settings
- **Engine**: Select OpenSearch as search engine
- **Server Hostname**: OpenSearch server host
- **Server Port**: OpenSearch server port (default: 9200)
- **Index Prefix**: Prefix for index names
- **Enable Authentication**: Enable/disable authentication
- **Server Timeout**: Connection timeout in seconds

#### Index Management
- **Minimum Should Match**: Query matching requirements
- **Enable EAV Indexer**: Use EAV attributes in search
- **Search Suggestions**: Enable search suggestions
- **Search Recommendations**: Enable search recommendations

### Environment Configuration
```php
// File: app/etc/env.php
return [
    'system' => [
        'default' => [
            'catalog' => [
                'search' => [
                    'engine' => 'opensearch',
                    'opensearch_server_hostname' => 'opensearch-cluster.example.com',
                    'opensearch_server_port' => '443',
                    'opensearch_index_prefix' => 'magento_prod',
                    'opensearch_enable_auth' => '1',
                    'opensearch_username' => 'magento_user',
                    'opensearch_password' => 'secure_password',
                    'opensearch_server_timeout' => '30'
                ]
            ]
        ]
    ]
];
```

## Customization and Extension

### Creating Custom Search Attributes

#### 1. Make Attribute Searchable
```php
// In your module's InstallData.php
public function install(ModuleDataSetupInterface $setup, ModuleContextInterface $context)
{
    $eavSetup = $this->eavSetupFactory->create(['setup' => $setup]);
    
    $eavSetup->addAttribute(
        \Magento\Catalog\Model\Product::ENTITY,
        'custom_search_field',
        [
            'type' => 'text',
            'label' => 'Custom Search Field',
            'input' => 'text',
            'required' => false,
            'searchable' => true,           // Make searchable
            'filterable' => true,           // Make filterable
            'comparable' => false,
            'visible_on_front' => true,
            'used_in_product_listing' => true,
            'unique' => false
        ]
    );
}
```

#### 2. Custom Data Mapper
```php
// File: app/code/YourVendor/YourModule/Model/Adapter/BatchDataMapper/CustomDataMapper.php
class CustomDataMapper implements BatchDataMapperInterface
{
    public function map($documentData, $storeId, $context = [])
    {
        $documents = [];
        
        foreach ($documentData as $productId => $indexData) {
            $documents[$productId] = [
                'custom_field' => $this->processCustomField($indexData),
                'calculated_score' => $this->calculateCustomScore($indexData),
                'tags' => $this->extractTags($indexData)
            ];
        }
        
        return $documents;
    }
    
    private function processCustomField($data)
    {
        // Custom processing logic
        return strtolower(trim($data['custom_field'] ?? ''));
    }
}
```

### Custom Search Queries

#### 1. Custom Query Builder
```php
// File: app/code/YourVendor/YourModule/Model/Search/CustomQuery.php
class CustomQuery
{
    public function buildCustomQuery($searchTerm, $filters = [])
    {
        $query = [
            'bool' => [
                'must' => [
                    [
                        'function_score' => [
                            'query' => [
                                'multi_match' => [
                                    'query' => $searchTerm,
                                    'fields' => ['name^2', 'description', 'custom_field^1.5']
                                ]
                            ],
                            'functions' => [
                                [
                                    'filter' => ['term' => ['is_new' => true]],
                                    'weight' => 1.2
                                ],
                                [
                                    'field_value_factor' => [
                                        'field' => 'popularity_score',
                                        'factor' => 1.5,
                                        'modifier' => 'log1p'
                                    ]
                                ]
                            ],
                            'score_mode' => 'multiply',
                            'boost_mode' => 'multiply'
                        ]
                    ]
                ]
            ]
        ];
        
        return $query;
    }
}
```

#### 2. Custom Search Adapter
```php
// File: app/code/YourVendor/YourModule/SearchAdapter/CustomAdapter.php
class CustomAdapter implements AdapterInterface
{
    public function query(RequestInterface $request)
    {
        $client = $this->clientFactory->create();
        
        // Build custom query
        $query = $this->buildQuery($request);
        
        // Execute search
        $rawResponse = $client->query($query);
        
        // Process response
        return $this->responseFactory->create($rawResponse);
    }
    
    private function buildQuery(RequestInterface $request)
    {
        // Custom query building logic
        return [
            'index' => $this->getIndexName($request),
            'body' => [
                'query' => $this->customQuery->buildCustomQuery(
                    $request->getName(),
                    $request->getFilters()
                )
            ]
        ];
    }
}
```

### Event Observers

#### Modify Index Data
```php
// File: app/code/YourVendor/YourModule/Observer/ModifyIndexData.php
class ModifyIndexData implements ObserverInterface
{
    public function execute(\Magento\Framework\Event\Observer $observer)
    {
        $productData = $observer->getEvent()->getProductData();
        $productId = $observer->getEvent()->getProductId();
        $storeId = $observer->getEvent()->getStoreId();
        
        // Add custom data to index
        $productData['custom_boost'] = $this->calculateBoost($productId);
        $productData['search_tags'] = $this->getSearchTags($productId);
        
        $observer->getEvent()->setProductData($productData);
    }
}
```

### Plugin Examples

#### Modify Search Results
```php
// File: app/code/YourVendor/YourModule/Plugin/SearchResultPlugin.php
class SearchResultPlugin
{
    public function afterGetItems(
        \Magento\CatalogSearch\Model\ResourceModel\Fulltext\Collection $subject,
        $result
    ) {
        // Modify search results after retrieval
        foreach ($result as $item) {
            // Add custom scoring or modify items
            $item->setCustomScore($this->calculateCustomScore($item));
        }
        
        return $result;
    }
}
```

## Performance Optimization

### Index Optimization

#### 1. Mapping Optimization
```php
public function getMapping()
{
    return [
        'properties' => [
            'name' => [
                'type' => 'text',
                'analyzer' => 'standard',
                'fields' => [
                    'keyword' => [
                        'type' => 'keyword',
                        'ignore_above' => 256
                    ],
                    'suggest' => [
                        'type' => 'completion'
                    ]
                ]
            ],
            'price' => [
                'type' => 'double',
                'index' => true
            ],
            'created_at' => [
                'type' => 'date',
                'format' => 'yyyy-MM-dd HH:mm:ss'
            ]
        ]
    ];
}
```

#### 2. Bulk Indexing
```php
public function bulkIndex($products, $storeId)
{
    $bulkData = [];
    $batchSize = 1000;
    
    foreach (array_chunk($products, $batchSize) as $batch) {
        foreach ($batch as $product) {
            $bulkData[] = [
                'index' => [
                    '_index' => $this->getIndexName($storeId),
                    '_id' => $product->getId()
                ]
            ];
            $bulkData[] = $this->prepareProductData($product);
        }
        
        $this->client->bulk(['body' => $bulkData]);
        $bulkData = []; // Clear for next batch
    }
}
```

### Query Optimization

#### 1. Query Caching
```php
class CachedSearchAdapter
{
    public function query(RequestInterface $request)
    {
        $cacheKey = $this->generateCacheKey($request);
        
        if ($cachedResult = $this->cache->load($cacheKey)) {
            return unserialize($cachedResult);
        }
        
        $result = $this->adapter->query($request);
        
        $this->cache->save(
            serialize($result),
            $cacheKey,
            ['search_results'],
            3600 // 1 hour cache
        );
        
        return $result;
    }
}
```

#### 2. Connection Pooling
```php
public function getClient()
{
    if (!$this->client) {
        $hosts = [
            [
                'host' => $this->config->getHost(),
                'port' => $this->config->getPort(),
                'scheme' => 'https'
            ]
        ];
        
        $this->client = ClientBuilder::create()
            ->setHosts($hosts)
            ->setConnectionPool('\Elasticsearch\ConnectionPool\StaticNoPingConnectionPool')
            ->setSelector('\Elasticsearch\ConnectionPool\Selectors\RoundRobinSelector')
            ->setRetries(2)
            ->build();
    }
    
    return $this->client;
}
```

### Performance Configuration

#### OpenSearch Settings
```yaml
# opensearch.yml
cluster.name: magento-search
node.name: node-1
network.host: 0.0.0.0
discovery.type: single-node

# Memory settings
bootstrap.memory_lock: true
indices.memory.index_buffer_size: 30%
indices.queries.cache.size: 20%

# Thread pool settings
thread_pool.search.size: 8
thread_pool.search.queue_size: 1000
```

#### Magento Configuration
```xml
<config>
    <default>
        <catalog>
            <search>
                <opensearch_batch_size>1000</opensearch_batch_size>
                <opensearch_connection_timeout>30</opensearch_connection_timeout>
                <opensearch_max_query_size>128</opensearch_max_query_size>
            </search>
        </catalog>
    </default>
</config>
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Connection Issues
```bash
# Test connection
curl -X GET "localhost:9200/_cluster/health?pretty"

# Check Magento logs
tail -f var/log/elasticsearch.log
tail -f var/log/system.log
```

#### 2. Index Issues
```bash
# Check index status
php bin/magento indexer:status catalogsearch_fulltext

# Reset and reindex
php bin/magento indexer:reset catalogsearch_fulltext
php bin/magento indexer:reindex catalogsearch_fulltext

# Check index in OpenSearch
curl -X GET "localhost:9200/_cat/indices?v"
```

#### 3. Search Issues
```php
// Debug search queries
public function debugQuery($searchTerm)
{
    $query = [
        'index' => $this->getIndexName(),
        'body' => [
            'query' => [
                'multi_match' => [
                    'query' => $searchTerm,
                    'fields' => ['name', 'description']
                ]
            ]
        ]
    ];
    
    // Add explain parameter
    $query['body']['explain'] = true;
    
    $response = $this->client->search($query);
    
    // Log the response for debugging
    $this->logger->debug('Search Debug', ['response' => $response]);
    
    return $response;
}
```

### Monitoring and Maintenance

#### Health Checks
```php
public function checkHealth()
{
    try {
        $health = $this->client->cluster()->health();
        return [
            'status' => $health['status'],
            'nodes' => $health['number_of_nodes'],
            'indices' => $health['active_primary_shards']
        ];
    } catch (\Exception $e) {
        return ['status' => 'error', 'message' => $e->getMessage()];
    }
}
```

#### Performance Monitoring
```php
public function getPerformanceMetrics()
{
    $stats = $this->client->indices()->stats();
    
    return [
        'total_docs' => $stats['_all']['total']['docs']['count'],
        'index_size' => $stats['_all']['total']['store']['size_in_bytes'],
        'query_time' => $stats['_all']['total']['search']['query_time_in_millis'],
        'query_count' => $stats['_all']['total']['search']['query_total']
    ];
}
```

This comprehensive guide covers all aspects of OpenSearch integration with Magento 2, from basic setup to advanced customization and optimization techniques.