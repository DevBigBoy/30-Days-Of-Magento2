# Complete GraphQL in Magento Tutorial Guide

## Table of Contents

### Introduction
- [Welcome & Setup](#welcome--setup)
- [What Is GraphQL?](#what-is-graphql)
- [Magento GraphQL Setup](#magento-graphql-setup)

### Chapter 1: GraphQL Basics
- [1.1 Overview](#11-overview)
- [1.2 GraphQL Queries](#12-graphql-queries)
- [1.3 Advanced Query Syntax](#13-advanced-query-syntax)
- [1.4 Mutations](#14-mutations)
- [1.5 Schema Language](#15-schema-language)
- [1.6 graphql-php and Magento](#16-graphql-php-and-magento)

### Chapter 2: Building GraphQL in Magento
- [2.1 Existing Queries](#21-existing-queries)
- [2.2 Custom Scenario Development](#22-custom-scenario-development)
- [2.3 Resolvers](#23-resolvers)
- [2.4 Context & Authorization](#24-context--authorization)

### Chapter 3: Caching
- [3.1 Caching Basics](#31-caching-basics)
- [3.2 Varnish Configuration](#32-varnish-configuration)
- [3.3 Making Queries Cacheable](#33-making-queries-cacheable)

### Chapter 4: Advanced Concepts
- [4.1 Deep Dives](#41-deep-dives)
- [4.2 Product Attributes & Types](#42-product-attributes--types)
- [4.3 Type Resolvers & Staging](#43-type-resolvers--staging)
- [4.4 Query Limits & Security](#44-query-limits--security)

---

## Introduction

### Welcome & Setup

Welcome to the complete GraphQL in Magento tutorial! This guide will take you from GraphQL basics to advanced Magento implementations with real-world examples.

**Prerequisites:**
- Magento 2.3+ installation
- Basic PHP knowledge
- Understanding of Magento module development
- GraphiQL or similar GraphQL client

**Module Setup:**
Create a new module for our examples:

```bash
mkdir -p app/code/Tutorial/GraphQLDemo
```

**registration.php:**
```php
<?php
use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Tutorial_GraphQLDemo',
    __DIR__
);
```

**module.xml:**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Tutorial_GraphQLDemo" setup_version="1.0.0">
        <sequence>
            <module name="Magento_GraphQl"/>
        </sequence>
    </module>
</config>
```

### What Is GraphQL?

GraphQL is a query language and runtime for APIs that provides:

**Key Benefits:**
- **Single Endpoint**: One URL for all data operations
- **Flexible Queries**: Request exactly the data you need
- **Strong Type System**: Self-documenting schemas
- **Real-time**: Built-in subscription support
- **Efficient**: Reduces over-fetching and under-fetching

**GraphQL vs REST:**
```
REST: GET /api/products/1, GET /api/products/1/reviews
GraphQL: Single query for product + reviews in one request
```

**Basic GraphQL Query Structure:**
```graphql
query {
  products(filter: {sku: {eq: "24-MB01"}}) {
    items {
      id
      name
      sku
      price_range {
        minimum_price {
          regular_price {
            value
            currency
          }
        }
      }
    }
  }
}
```

### Magento GraphQL Setup

**Access GraphQL Endpoint:**
```
GET/POST: https://your-magento-site.com/graphql
```

**Enable GraphQL in Magento:**
```bash
php bin/magento module:enable Magento_GraphQl
php bin/magento setup:upgrade
```

**Test with curl:**
```bash
curl -X POST \
  https://your-site.com/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"query { storeConfig { store_name } }"}'
```

---

## Chapter 1: GraphQL Basics

### 1.1 Overview

GraphQL is built around three main concepts:

**1. Schema**: Defines the structure of your API
**2. Queries**: Read operations to fetch data
**3. Mutations**: Write operations to modify data

**Magento's GraphQL Architecture:**
```
Frontend Request → GraphQL Endpoint → Schema → Resolver → Magento Core → Database
```

**Key Components:**
- **Schema Definition**: `.graphqls` files
- **Resolvers**: PHP classes that fetch data
- **Type System**: Scalars, Objects, Interfaces, Unions
- **Introspection**: Self-documenting capabilities

### 1.2 GraphQL Queries

**Basic Query Structure:**
```graphql
query {
  field
  objectField {
    nestedField
  }
}
```

**Real Magento Examples:**

**1. Store Configuration:**
```graphql
query GetStoreConfig {
  storeConfig {
    id
    code
    store_name
    is_default_store
    store_group_code
    website_id
    locale
    base_currency_code
    default_display_currency_code
    timezone
    weight_unit
    base_url
    base_media_url
    secure_base_url
  }
}
```

**2. Product Query:**
```graphql
query GetProducts {
  products(filter: {name: {match: "bag"}}) {
    total_count
    items {
      id
      name
      sku
      url_key
      image {
        url
        label
      }
      price_range {
        minimum_price {
          regular_price {
            value
            currency
          }
          discount {
            amount_off
            percent_off
          }
        }
      }
      categories {
        id
        name
        url_path
      }
    }
    page_info {
      page_size
      current_page
      total_pages
    }
  }
}
```

**3. Category Query:**
```graphql
query GetCategory {
  category(id: 2) {
    id
    name
    description
    image
    children {
      id
      name
      level
      children_count
    }
    products {
      total_count
      items {
        name
        sku
      }
    }
  }
}
```

### 1.3 Advanced Query Syntax

**Variables:**
```graphql
query GetProductsBySku($sku: String!) {
  products(filter: {sku: {eq: $sku}}) {
    items {
      name
      price_range {
        minimum_price {
          regular_price {
            value
          }
        }
      }
    }
  }
}

# Variables:
{
  "sku": "24-MB01"
}
```

**Aliases:**
```graphql
query GetMultipleProducts {
  product1: products(filter: {sku: {eq: "24-MB01"}}) {
    items {
      name
      sku
    }
  }
  product2: products(filter: {sku: {eq: "24-MB02"}}) {
    items {
      name
      sku
    }
  }
}
```

**Fragments:**
```graphql
fragment ProductInfo on ProductInterface {
  id
  name
  sku
  url_key
  price_range {
    minimum_price {
      regular_price {
        value
        currency
      }
    }
  }
}

query GetProducts {
  products(filter: {name: {match: "shirt"}}) {
    items {
      ...ProductInfo
      categories {
        name
      }
    }
  }
}
```

**Inline Fragments:**
```graphql
query GetProducts {
  products(filter: {type_id: {in: ["simple", "configurable"]}}) {
    items {
      name
      sku
      ... on SimpleProduct {
        weight
      }
      ... on ConfigurableProduct {
        configurable_options {
          attribute_code
          label
          values {
            value_index
            label
          }
        }
      }
    }
  }
}
```

### 1.4 Mutations

Mutations modify data. Magento provides several built-in mutations:

**1. Add Product to Cart:**
```graphql
mutation AddProductToCart($cartId: String!, $sku: String!, $qty: Float!) {
  addSimpleProductsToCart(
    input: {
      cart_id: $cartId
      cart_items: [
        {
          data: {
            quantity: $qty
            sku: $sku
          }
        }
      ]
    }
  ) {
    cart {
      id
      items {
        id
        product {
          name
          sku
        }
        quantity
      }
      prices {
        grand_total {
          value
          currency
        }
      }
    }
  }
}

# Variables:
{
  "cartId": "customer_cart_id",
  "sku": "24-MB01",
  "qty": 1
}
```

**2. Create Empty Cart:**
```graphql
mutation CreateEmptyCart {
  createEmptyCart
}
```

**3. Apply Coupon:**
```graphql
mutation ApplyCoupon($cartId: String!, $couponCode: String!) {
  applyCouponToCart(
    input: {
      cart_id: $cartId
      coupon_code: $couponCode
    }
  ) {
    cart {
      applied_coupons {
        code
      }
      prices {
        grand_total {
          value
        }
        discounts {
          amount {
            value
          }
          label
        }
      }
    }
  }
}
```

### 1.5 Schema Language

**Schema Definition Files (.graphqls):**

**etc/schema.graphqls:**
```graphql
type Query {
    customProducts(
        search: String @doc(description: "Search term")
        filter: ProductAttributeFilterInput @doc(description: "Product filter")
        pageSize: Int = 20 @doc(description: "Number of items per page")
        currentPage: Int = 1 @doc(description: "Current page number")
    ): CustomProductsOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\Products") @doc(description: "Custom product search")
}

type CustomProductsOutput @doc(description: "Custom products output") {
    items: [ProductInterface] @doc(description: "Product items")
    total_count: Int @doc(description: "Total number of products")
    page_info: SearchResultPageInfo @doc(description: "Page information")
}

type Mutation {
    createCustomProduct(
        input: CreateCustomProductInput! @doc(description: "Product input data")
    ): CreateCustomProductOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\CreateProduct") @doc(description: "Create a custom product")
}

input CreateCustomProductInput @doc(description: "Input for creating custom product") {
    name: String! @doc(description: "Product name")
    sku: String! @doc(description: "Product SKU")
    price: Float! @doc(description: "Product price")
    description: String @doc(description: "Product description")
}

type CreateCustomProductOutput @doc(description: "Output for created product") {
    product: ProductInterface @doc(description: "Created product")
    success: Boolean! @doc(description: "Success status")
    message: String @doc(description: "Result message")
}
```

**Type System Examples:**

**Scalar Types:**
```graphql
scalar String
scalar Int
scalar Float
scalar Boolean
scalar ID
```

**Object Types:**
```graphql
type CustomProduct implements ProductInterface {
    id: Int
    name: String
    sku: String
    custom_attribute: String
}
```

**Interface Types:**
```graphql
interface ProductInterface {
    id: Int
    name: String
    sku: String
}
```

**Union Types:**
```graphql
union SearchResult = Product | Category | CmsPage
```

**Enum Types:**
```graphql
enum ProductStatus {
    ENABLED
    DISABLED
}
```

### 1.6 graphql-php and Magento

Magento uses the `webonyx/graphql-php` library. Key integration points:

**1. Schema Building:**
```php
// Magento\Framework\GraphQl\Schema\Type\TypeRegistry
public function get(string $typeName): TypeInterface
{
    if (!isset($this->types[$typeName])) {
        $this->types[$typeName] = $this->configElement->get($typeName);
    }
    return $this->types[$typeName];
}
```

**2. Query Execution:**
```php
// Magento\GraphQl\Controller\GraphQl
public function execute()
{
    $schema = $this->schemaGenerator->generate();
    $result = GraphQL::executeQuery(
        $schema,
        $this->extractQuery($request),
        null,
        $this->contextFactory->create($request),
        $this->extractVariables($request)
    );
    return $this->jsonSerializer->serialize($result->toArray());
}
```

**3. Resolver Integration:**
```php
// Base resolver structure
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;

class CustomResolver implements ResolverInterface
{
    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        // Resolution logic here
        return $result;
    }
}
```

---

## Chapter 2: Building GraphQL in Magento

### 2.1 Existing Queries

**2.1.1 Existing Queries Overview**

Magento provides extensive built-in GraphQL queries:

**Core Queries:**
- `products` - Product catalog
- `category` - Category information
- `storeConfig` - Store configuration
- `cart` - Shopping cart operations
- `customer` - Customer data
- `countries` - Country/region data
- `currency` - Currency information

**2.1.2 Store Config Query**

**Implementation Example:**
```graphql
query GetFullStoreConfig {
  storeConfig {
    # Basic store info
    id
    code
    store_name
    is_default_store
    store_group_code
    website_id
    
    # Localization
    locale
    timezone
    weight_unit
    
    # Currency settings
    base_currency_code
    default_display_currency_code
    available_currency_codes
    
    # URLs
    base_url
    base_media_url
    base_static_url
    secure_base_url
    secure_base_media_url
    secure_base_static_url
    
    # Catalog settings
    catalog_default_sort_by
    category_fixed_product_tax_display_setting
    category_url_suffix
    product_url_suffix
    
    # Customer settings
    autocomplete_on_storefront
    create_account_confirmation
    minimum_password_length
    required_character_classes_number
    
    # Shopping cart
    minicart_display
    minicart_max_items
    shopping_cart_display_tax_column
    shopping_cart_display_shipping_column
    
    # Checkout
    checkout_agreement_enabled
    guest_checkout
    
    # Email settings
    send_friend {
      enabled_for_customers
      enabled_for_guests
    }
    
    # CMS
    cms_home_page
    cms_no_route
    
    # Header/Footer
    header_logo_src
    logo_alt
    welcome
    copyright
  }
}
```

**2.1.3 Products Query**

**Advanced Product Query:**
```graphql
query GetProductsAdvanced(
    $filter: ProductAttributeFilterInput!
    $sort: ProductAttributeSortInput
    $pageSize: Int = 20
    $currentPage: Int = 1
) {
  products(
    filter: $filter
    sort: $sort
    pageSize: $pageSize
    currentPage: $currentPage
  ) {
    total_count
    aggregations {
      attribute_code
      count
      label
      options {
        count
        label
        value
      }
    }
    sort_fields {
      default
      options {
        label
        value
      }
    }
    items {
      __typename
      id
      name
      sku
      url_key
      url_suffix
      
      # Media
      image {
        url
        label
        position
      }
      media_gallery {
        url
        label
        position
        disabled
        ... on ProductVideo {
          video_content {
            media_type
            video_provider
            video_url
            video_title
            video_description
            video_metadata
          }
        }
      }
      
      # Pricing
      price_range {
        minimum_price {
          regular_price {
            value
            currency
          }
          final_price {
            value
            currency
          }
          discount {
            amount_off
            percent_off
          }
        }
        maximum_price {
          regular_price {
            value
            currency
          }
          final_price {
            value
            currency
          }
          discount {
            amount_off
            percent_off
          }
        }
      }
      
      # Categories
      categories {
        id
        name
        level
        url_path
        breadcrumbs {
          category_id
          category_name
          category_level
          category_url_path
        }
      }
      
      # Attributes
      manufacturer
      color
      size
      material
      custom_attributes {
        attribute_metadata {
          uid
          code
          label
          data_type
        }
        entered_attribute_value {
          value
        }
        selected_attribute_options {
          attribute_option {
            uid
            label
            value
          }
        }
      }
      
      # Reviews
      review_count
      rating_summary
      reviews {
        items {
          average_rating
          ratings_breakdown {
            name
            value
          }
          summary
          text
          created_at
          nickname
        }
      }
      
      # Stock
      stock_status
      only_x_left_in_stock
      
      # Product Type Specific
      ... on SimpleProduct {
        weight
      }
      ... on ConfigurableProduct {
        configurable_options {
          id
          attribute_id
          attribute_code
          label
          position
          use_default
          values {
            value_index
            label
            default_label
            store_label
            use_default_value
            swatch_data {
              value
              ... on ImageSwatchData {
                thumbnail
              }
              ... on ColorSwatchData {
                value
              }
              ... on TextSwatchData {
                value
              }
            }
          }
        }
        variants {
          product {
            id
            name
            sku
            attribute_set_id
            ... on PhysicalProductInterface {
              weight
            }
            price_range {
              minimum_price {
                regular_price {
                  value
                  currency
                }
              }
            }
          }
          attributes {
            label
            code
            value_index
          }
        }
      }
      ... on BundleProduct {
        dynamic_price
        dynamic_sku
        dynamic_weight
        price_view
        ship_bundle_items
        items {
          option_id
          title
          required
          type
          position
          sku
          options {
            id
            label
            quantity
            position
            is_default
            price
            price_type
            can_change_quantity
            product {
              id
              name
              sku
            }
          }
        }
      }
      ... on GroupedProduct {
        items {
          qty
          position
          product {
            id
            name
            sku
            price_range {
              minimum_price {
                regular_price {
                  value
                  currency
                }
              }
            }
          }
        }
      }
      ... on DownloadableProduct {
        downloadable_product_samples {
          id
          title
          sort_order
          sample_url
        }
        downloadable_product_links {
          id
          title
          sort_order
          is_shareable
          price
          number_of_downloads
          link_type
          sample_url
        }
      }
    }
    page_info {
      page_size
      current_page
      total_pages
    }
  }
}

# Variables:
{
  "filter": {
    "name": {"match": "shirt"},
    "price": {"from": "10", "to": "100"},
    "category_id": {"eq": "11"}
  },
  "sort": {
    "name": "ASC"
  },
  "pageSize": 12,
  "currentPage": 1
}
```

**2.1.4 Product Attributes and Filtering**

**Custom Attribute Query:**
```graphql
query GetCustomAttributes {
  customAttributeMetadata(attributes: [
    {attribute_code: "color", entity_type: "catalog_product"}
    {attribute_code: "size", entity_type: "catalog_product"}
    {attribute_code: "material", entity_type: "catalog_product"}
  ]) {
    items {
      code
      label
      data_type
      is_system
      entity_type
      frontend_input
      is_required
      default_value
      is_unique
      ... on CatalogAttributeMetadata {
        apply_to
        is_comparable
        is_filterable
        is_filterable_in_search
        is_html_allowed_on_front
        is_searchable
        is_used_for_promo_rules
        is_visible_in_advanced_search
        is_visible_on_front
        is_wysiwyg_enabled
        used_for_sort_by
        used_in_product_listing
      }
      options {
        label
        value
        is_default
      }
    }
  }
}
```

**Advanced Filtering:**
```graphql
query GetFilteredProducts {
  products(
    filter: {
      # Text filters
      name: {match: "mens shirt"}
      sku: {like: "MS%"}
      description: {match: "cotton"}
      
      # Numeric filters
      price: {from: "20", to: "100"}
      weight: {gteq: "0.5"}
      
      # Category filters
      category_id: {in: ["11", "12", "13"]}
      category_uid: {eq: "MTE="}
      
      # Attribute filters
      color: {in: ["blue", "red"]}
      size: {eq: "large"}
      manufacturer: {eq: "Nike"}
      
      # Boolean filters
      news_from_date: {date: true}
      special_price: {neq: "null"}
      
      # Custom attribute filters
      custom_attribute: {eq: "custom_value"}
    }
    sort: {
      name: ASC
      price: DESC
      created_at: DESC
    }
  ) {
    items {
      name
      sku
      price_range {
        minimum_price {
          final_price {
            value
          }
        }
      }
    }
    aggregations {
      attribute_code
      label
      count
      options {
        label
        value
        count
      }
    }
  }
}
```

**2.1.5 Mutations**

**Customer Account Mutations:**
```graphql
# Create Customer Account
mutation CreateCustomer {
  createCustomer(
    input: {
      firstname: "John"
      lastname: "Doe"
      email: "john.doe@example.com"
      password: "SecurePassword123!"
      is_subscribed: true
    }
  ) {
    customer {
      id
      firstname
      lastname
      email
      is_subscribed
    }
  }
}

# Generate Customer Token
mutation GenerateCustomerToken {
  generateCustomerToken(
    email: "john.doe@example.com"
    password: "SecurePassword123!"
  ) {
    token
  }
}

# Update Customer
mutation UpdateCustomer {
  updateCustomer(
    input: {
      firstname: "John"
      lastname: "Smith"
      date_of_birth: "1990-01-01"
      gender: 1
    }
  ) {
    customer {
      firstname
      lastname
      date_of_birth
      gender
    }
  }
}
```

**Cart Mutations:**
```graphql
# Create Empty Cart
mutation CreateEmptyCart {
  createEmptyCart
}

# Add Simple Product
mutation AddSimpleProduct($cartId: String!) {
  addSimpleProductsToCart(
    input: {
      cart_id: $cartId
      cart_items: [
        {
          data: {
            quantity: 2
            sku: "24-MB01"
          }
          customizable_options: [
            {
              id: 1
              value_string: "Custom Text"
            }
          ]
        }
      ]
    }
  ) {
    cart {
      id
      items {
        id
        quantity
        product {
          name
          sku
        }
        prices {
          price {
            value
          }
        }
      }
    }
  }
}

# Add Configurable Product
mutation AddConfigurableProduct($cartId: String!) {
  addConfigurableProductsToCart(
    input: {
      cart_id: $cartId
      cart_items: [
        {
          data: {
            quantity: 1
            sku: "MH01"
          }
          variant_sku: "MH01-XS-Black"
        }
      ]
    }
  ) {
    cart {
      items {
        id
        quantity
        product {
          name
          sku
        }
        ... on ConfigurableCartItem {
          configurable_options {
            option_label
            value_label
          }
        }
      }
    }
  }
}

# Apply Coupon
mutation ApplyCoupon($cartId: String!) {
  applyCouponToCart(
    input: {
      cart_id: $cartId
      coupon_code: "SAVE10"
    }
  ) {
    cart {
      applied_coupons {
        code
      }
      prices {
        discounts {
          amount {
            value
          }
          label
        }
      }
    }
  }
}
```

**2.1.6 Schema**

Understanding Magento's schema structure:

**Core Schema Files:**
- `Magento/CatalogGraphQl/etc/schema.graphqls` - Products, categories
- `Magento/QuoteGraphQl/etc/schema.graphqls` - Cart operations  
- `Magento/CustomerGraphQl/etc/schema.graphqls` - Customer data
- `Magento/StoreGraphQl/etc/schema.graphqls` - Store configuration

**Schema Inheritance:**
```graphql
interface ProductInterface {
    id: Int
    name: String
    sku: String
    # ... common fields
}

type SimpleProduct implements ProductInterface & PhysicalProductInterface {
    # Inherits all ProductInterface fields
    weight: Float  # From PhysicalProductInterface
    # Simple product specific fields
}

type ConfigurableProduct implements ProductInterface & PhysicalProductInterface {
    # Inherits all ProductInterface fields  
    weight: Float  # From PhysicalProductInterface
    configurable_options: [ConfigurableProductOptions]
    variants: [ConfigurableVariant]
}
```

### 2.2 Custom Scenario Development

**2.2.1 Our Custom Scenario**

Let's build a custom shipping policy management system with GraphQL:

**Requirements:**
- Query shipping policies by region
- Create/update shipping policies
- User-specific shipping rules
- Caching support

**2.2.2 Requirements**

**Functional Requirements:**
1. **Query Operations:**
   - Get shipping policies by country/region
   - Filter policies by criteria
   - Paginated results

2. **Mutation Operations:**
   - Create new shipping policy
   - Update existing policy
   - Delete policy

3. **Authorization:**
   - Admin-only mutations
   - Customer-specific policy viewing

4. **Caching:**
   - Cache policies by region
   - Invalidate on updates

**2.2.3 Prerequisites**

**Database Schema:**
```sql
CREATE TABLE tutorial_shipping_policies (
    entity_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    country_code VARCHAR(2) NOT NULL,
    region_code VARCHAR(10),
    min_order_amount DECIMAL(12,4) DEFAULT 0,
    shipping_cost DECIMAL(12,4) NOT NULL,
    delivery_days INT DEFAULT 5,
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_country_region (country_code, region_code),
    INDEX idx_active (is_active)
);
```

**Module Structure:**
```
Tutorial/GraphQLDemo/
├── Api/
│   ├── Data/
│   │   └── ShippingPolicyInterface.php
│   └── ShippingPolicyRepositoryInterface.php
├── Model/
│   ├── Data/
│   │   └── ShippingPolicy.php
│   ├── ResourceModel/
│   │   ├── ShippingPolicy.php
│   │   └── ShippingPolicy/
│   │       └── Collection.php
│   ├── Resolver/
│   │   ├── ShippingPolicies.php
│   │   └── CreateShippingPolicy.php
│   └── ShippingPolicyRepository.php
├── etc/
│   ├── db_schema.xml
│   ├── di.xml
│   └── schema.graphqls
└── Setup/
    └── InstallData.php
```

**2.2.4 Creating the Basic Schema**

**etc/schema.graphqls:**
```graphql
type Query {
    shippingPolicies(
        filter: ShippingPolicyFilterInput @doc(description: "Filter criteria")
        pageSize: Int = 20 @doc(description: "Number of items per page")
        currentPage: Int = 1 @doc(description: "Current page number")
    ): ShippingPoliciesOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\ShippingPolicies") @doc(description: "Get shipping policies")
    
    shippingPolicy(
        id: Int! @doc(description: "Policy ID")
    ): ShippingPolicy @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\ShippingPolicy") @doc(description: "Get single shipping policy")
}

type Mutation {
    createShippingPolicy(
        input: CreateShippingPolicyInput! @doc(description: "Policy input data")
    ): CreateShippingPolicyOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\CreateShippingPolicy") @doc(description: "Create shipping policy")
    
    updateShippingPolicy(
        id: Int! @doc(description: "Policy ID")
        input: UpdateShippingPolicyInput! @doc(description: "Policy update data")
    ): UpdateShippingPolicyOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\UpdateShippingPolicy") @doc(description: "Update shipping policy")
    
    deleteShippingPolicy(
        id: Int! @doc(description: "Policy ID")
    ): DeleteShippingPolicyOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\DeleteShippingPolicy") @doc(description: "Delete shipping policy")
}

type ShippingPolicy @doc(description: "Shipping policy") {
    id: Int @doc(description: "Policy ID")
    name: String @doc(description: "Policy name")
    country_code: String @doc(description: "Country code")
    region_code: String @doc(description: "Region code")
    min_order_amount: Float @doc(description: "Minimum order amount")
    shipping_cost: Float @doc(description: "Shipping cost")
    delivery_days: Int @doc(description: "Delivery days")
    is_active: Boolean @doc(description: "Is active")
    created_at: String @doc(description: "Created date")
    updated_at: String @doc(description: "Updated date")
}

type ShippingPoliciesOutput @doc(description: "Shipping policies output") {
    items: [ShippingPolicy] @doc(description: "Policy items")
    total_count: Int @doc(description: "Total count")
    page_info: SearchResultPageInfo @doc(description: "Page info")
}

input ShippingPolicyFilterInput @doc(description: "Shipping policy filter") {
    country_code: FilterTypeInput @doc(description: "Country code filter")
    region_code: FilterTypeInput @doc(description: "Region code filter")
    name: FilterTypeInput @doc(description: "Name filter")
    min_order_amount: FilterRangeTypeInput @doc(description: "Min order amount filter")
    shipping_cost: FilterRangeTypeInput @doc(description: "Shipping cost filter")
    is_active: FilterTypeInput @doc(description: "Active status filter")
}

input CreateShippingPolicyInput @doc(description: "Create shipping policy input") {
    name: String! @doc(description: "Policy name")
    country_code: String! @doc(description: "Country code")
    region_code: String @doc(description: "Region code")
    min_order_amount: Float = 0 @doc(description: "Minimum order amount")
    shipping_cost: Float! @doc(description: "Shipping cost")
    delivery_days: Int = 5 @doc(description: "Delivery days")
    is_active: Boolean = true @doc(description: "Is active")
}

input UpdateShippingPolicyInput @doc(description: "Update shipping policy input") {
    name: String @doc(description: "Policy name")
    country_code: String @doc(description: "Country code")
    region_code: String @doc(description: "Region code")
    min_order_amount: Float @doc(description: "Minimum order amount")
    shipping_cost: Float @doc(description: "Shipping cost")
    delivery_days: Int @doc(description: "Delivery days")
    is_active: Boolean @doc(description: "Is active")
}

type CreateShippingPolicyOutput @doc(description: "Create shipping policy output") {
    policy: ShippingPolicy @doc(description: "Created policy")
    success: Boolean! @doc(description: "Success status")
    message: String @doc(description: "Result message")
}

type UpdateShippingPolicyOutput @doc(description: "Update shipping policy output") {
    policy: ShippingPolicy @doc(description: "Updated policy")
    success: Boolean! @doc(description: "Success status")
    message: String @doc(description: "Result message")
}

type DeleteShippingPolicyOutput @doc(description: "Delete shipping policy output") {
    success: Boolean! @doc(description: "Success status")
    message: String @doc(description: "Result message")
}
```

**Data Interface:**
```php
<?php
// Api/Data/ShippingPolicyInterface.php
namespace Tutorial\GraphQLDemo\Api\Data;

interface ShippingPolicyInterface
{
    const ENTITY_ID = 'entity_id';
    const NAME = 'name';
    const COUNTRY_CODE = 'country_code';
    const REGION_CODE = 'region_code';
    const MIN_ORDER_AMOUNT = 'min_order_amount';
    const SHIPPING_COST = 'shipping_cost';
    const DELIVERY_DAYS = 'delivery_days';
    const IS_ACTIVE = 'is_active';
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';

    public function getId();
    public function setId($id);
    public function getName();
    public function setName($name);
    public function getCountryCode();
    public function setCountryCode($countryCode);
    public function getRegionCode();
    public function setRegionCode($regionCode);
    public function getMinOrderAmount();
    public function setMinOrderAmount($amount);
    public function getShippingCost();
    public function setShippingCost($cost);
    public function getDeliveryDays();
    public function setDeliveryDays($days);
    public function getIsActive();
    public function setIsActive($isActive);
    public function getCreatedAt();
    public function setCreatedAt($createdAt);
    public function getUpdatedAt();
    public function setUpdatedAt($updatedAt);
}
```

### 2.3 Resolvers

**2.3.1 Resolvers**

Resolvers are the heart of GraphQL - they fetch the actual data.

**Base Query Resolver:**
```php
<?php
// Model/Resolver/ShippingPolicies.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Tutorial\GraphQLDemo\Api\ShippingPolicyRepositoryInterface;
use Magento\Framework\Api\SearchCriteriaBuilder;
use Magento\Framework\Api\FilterBuilder;
use Magento\Framework\Api\Search\FilterGroupBuilder;

class ShippingPolicies implements ResolverInterface
{
    private ShippingPolicyRepositoryInterface $shippingPolicyRepository;
    private SearchCriteriaBuilder $searchCriteriaBuilder;
    private FilterBuilder $filterBuilder;
    private FilterGroupBuilder $filterGroupBuilder;

    public function __construct(
        ShippingPolicyRepositoryInterface $shippingPolicyRepository,
        SearchCriteriaBuilder $searchCriteriaBuilder,
        FilterBuilder $filterBuilder,
        FilterGroupBuilder $filterGroupBuilder
    ) {
        $this->shippingPolicyRepository = $shippingPolicyRepository;
        $this->searchCriteriaBuilder = $searchCriteriaBuilder;
        $this->filterBuilder = $filterBuilder;
        $this->filterGroupBuilder = $filterGroupBuilder;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $this->validateArgs($args);
        
        $searchCriteria = $this->buildSearchCriteria($args);
        $searchResult = $this->shippingPolicyRepository->getList($searchCriteria);
        
        $items = [];
        foreach ($searchResult->getItems() as $policy) {
            $items[] = [
                'id' => $policy->getId(),
                'name' => $policy->getName(),
                'country_code' => $policy->getCountryCode(),
                'region_code' => $policy->getRegionCode(),
                'min_order_amount' => $policy->getMinOrderAmount(),
                'shipping_cost' => $policy->getShippingCost(),
                'delivery_days' => $policy->getDeliveryDays(),
                'is_active' => (bool)$policy->getIsActive(),
                'created_at' => $policy->getCreatedAt(),
                'updated_at' => $policy->getUpdatedAt(),
                'model' => $policy
            ];
        }

        $pageSize = $args['pageSize'] ?? 20;
        $currentPage = $args['currentPage'] ?? 1;
        $totalCount = $searchResult->getTotalCount();
        $totalPages = ceil($totalCount / $pageSize);

        return [
            'total_count' => $totalCount,
            'items' => $items,
            'page_info' => [
                'page_size' => $pageSize,
                'current_page' => $currentPage,
                'total_pages' => $totalPages
            ]
        ];
    }

    private function validateArgs(array $args): void
    {
        if (isset($args['pageSize']) && ($args['pageSize'] < 1 || $args['pageSize'] > 100)) {
            throw new GraphQlInputException(__('pageSize must be between 1 and 100'));
        }
        
        if (isset($args['currentPage']) && $args['currentPage'] < 1) {
            throw new GraphQlInputException(__('currentPage must be greater than 0'));
        }
    }

    private function buildSearchCriteria(array $args): \Magento\Framework\Api\SearchCriteriaInterface
    {
        $this->searchCriteriaBuilder->setPageSize($args['pageSize'] ?? 20);
        $this->searchCriteriaBuilder->setCurrentPage($args['currentPage'] ?? 1);

        if (isset($args['filter'])) {
            $this->applyFilters($args['filter']);
        }

        return $this->searchCriteriaBuilder->create();
    }

    private function applyFilters(array $filters): void
    {
        $filterGroups = [];

        foreach ($filters as $field => $condition) {
            $filters = [];
            
            if (isset($condition['eq'])) {
                $filters[] = $this->filterBuilder
                    ->setField($field)
                    ->setValue($condition['eq'])
                    ->setConditionType('eq')
                    ->create();
            }
            
            if (isset($condition['like'])) {
                $filters[] = $this->filterBuilder
                    ->setField($field)
                    ->setValue('%' . $condition['like'] . '%')
                    ->setConditionType('like')
                    ->create();
            }
            
            if (isset($condition['in'])) {
                $filters[] = $this->filterBuilder
                    ->setField($field)
                    ->setValue($condition['in'])
                    ->setConditionType('in')
                    ->create();
            }

            if (isset($condition['from']) || isset($condition['to'])) {
                if (isset($condition['from'])) {
                    $filters[] = $this->filterBuilder
                        ->setField($field)
                        ->setValue($condition['from'])
                        ->setConditionType('gteq')
                        ->create();
                }
                if (isset($condition['to'])) {
                    $filters[] = $this->filterBuilder
                        ->setField($field)
                        ->setValue($condition['to'])
                        ->setConditionType('lteq')
                        ->create();
                }
            }

            if (!empty($filters)) {
                $filterGroups[] = $this->filterGroupBuilder
                    ->setFilters($filters)
                    ->create();
            }
        }

        if (!empty($filterGroups)) {
            $this->searchCriteriaBuilder->setFilterGroups($filterGroups);
        }
    }
}
```

**Mutation Resolver:**
```php
<?php
// Model/Resolver/CreateShippingPolicy.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException;
use Magento\Framework\GraphQl\Exception\GraphQlInputException;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Tutorial\GraphQLDemo\Api\ShippingPolicyRepositoryInterface;
use Tutorial\GraphQLDemo\Api\Data\ShippingPolicyInterfaceFactory;
use Magento\CustomerGraphQl\Model\Customer\GetCustomer;

class CreateShippingPolicy implements ResolverInterface
{
    private ShippingPolicyRepositoryInterface $shippingPolicyRepository;
    private ShippingPolicyInterfaceFactory $shippingPolicyFactory;
    private GetCustomer $getCustomer;

    public function __construct(
        ShippingPolicyRepositoryInterface $shippingPolicyRepository,
        ShippingPolicyInterfaceFactory $shippingPolicyFactory,
        GetCustomer $getCustomer
    ) {
        $this->shippingPolicyRepository = $shippingPolicyRepository;
        $this->shippingPolicyFactory = $shippingPolicyFactory;
        $this->getCustomer = $getCustomer;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $this->validateAuthorization($context);
        $this->validateInput($args['input']);

        try {
            $policy = $this->shippingPolicyFactory->create();
            $input = $args['input'];
            
            $policy->setName($input['name']);
            $policy->setCountryCode($input['country_code']);
            $policy->setRegionCode($input['region_code'] ?? null);
            $policy->setMinOrderAmount($input['min_order_amount'] ?? 0);
            $policy->setShippingCost($input['shipping_cost']);
            $policy->setDeliveryDays($input['delivery_days'] ?? 5);
            $policy->setIsActive($input['is_active'] ?? true);

            $savedPolicy = $this->shippingPolicyRepository->save($policy);

            return [
                'policy' => [
                    'id' => $savedPolicy->getId(),
                    'name' => $savedPolicy->getName(),
                    'country_code' => $savedPolicy->getCountryCode(),
                    'region_code' => $savedPolicy->getRegionCode(),
                    'min_order_amount' => $savedPolicy->getMinOrderAmount(),
                    'shipping_cost' => $savedPolicy->getShippingCost(),
                    'delivery_days' => $savedPolicy->getDeliveryDays(),
                    'is_active' => (bool)$savedPolicy->getIsActive(),
                    'created_at' => $savedPolicy->getCreatedAt(),
                    'updated_at' => $savedPolicy->getUpdatedAt(),
                ],
                'success' => true,
                'message' => __('Shipping policy created successfully.')
            ];

        } catch (\Exception $e) {
            return [
                'policy' => null,
                'success' => false,
                'message' => __('Failed to create shipping policy: %1', $e->getMessage())
            ];
        }
    }

    private function validateAuthorization($context): void
    {
        if (!$context->getExtensionAttributes()->getIsCustomer()) {
            throw new GraphQlAuthorizationException(__('Current customer does not have access to this resource'));
        }

        // Additional admin check if needed
        $customer = $this->getCustomer->execute($context);
        // Add your admin validation logic here
    }

    private function validateInput(array $input): void
    {
        if (empty($input['name'])) {
            throw new GraphQlInputException(__('Name is required'));
        }

        if (empty($input['country_code']) || strlen($input['country_code']) !== 2) {
            throw new GraphQlInputException(__('Valid country code is required'));
        }

        if (!isset($input['shipping_cost']) || $input['shipping_cost'] < 0) {
            throw new GraphQlInputException(__('Valid shipping cost is required'));
        }

        if (isset($input['min_order_amount']) && $input['min_order_amount'] < 0) {
            throw new GraphQlInputException(__('Minimum order amount cannot be negative'));
        }

        if (isset($input['delivery_days']) && $input['delivery_days'] < 1) {
            throw new GraphQlInputException(__('Delivery days must be at least 1'));
        }
    }
}
```

**2.3.2 Asynchronous Resolvers**

For heavy operations, implement asynchronous resolvers:

```php
<?php
// Model/Resolver/AsyncShippingPolicies.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\Promise\Promise;
use Magento\Framework\Promise\PromiseFactory;

class AsyncShippingPolicies implements ResolverInterface
{
    private PromiseFactory $promiseFactory;
    private ShippingPolicyRepositoryInterface $repository;

    public function __construct(
        PromiseFactory $promiseFactory,
        ShippingPolicyRepositoryInterface $repository
    ) {
        $this->promiseFactory = $promiseFactory;
        $this->repository = $repository;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        return $this->promiseFactory->create(function() use ($args) {
            // Heavy operation that can be done asynchronously
            $searchCriteria = $this->buildSearchCriteria($args);
            $result = $this->repository->getList($searchCriteria);
            
            // Additional processing
            $enrichedItems = $this->enrichWithExternalData($result->getItems());
            
            return [
                'items' => $enrichedItems,
                'total_count' => $result->getTotalCount()
            ];
        });
    }

    private function enrichWithExternalData(array $items): array
    {
        // Simulate external API calls or heavy processing
        foreach ($items as &$item) {
            $item['external_data'] = $this->fetchExternalData($item['country_code']);
        }
        return $items;
    }

    private function fetchExternalData(string $countryCode): array
    {
        // Simulate external service call
        sleep(1); // This would be async in real implementation
        return ['country_name' => $this->getCountryName($countryCode)];
    }
}
```

**2.3.3 Creating the Countries Resolver**

Enhanced countries resolver with additional data:

```php
<?php
// Model/Resolver/Countries.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Directory\Api\CountryInformationAcquirerInterface;
use Tutorial\GraphQLDemo\Api\ShippingPolicyRepositoryInterface;

class Countries implements ResolverInterface
{
    private CountryInformationAcquirerInterface $countryInformationAcquirer;
    private ShippingPolicyRepositoryInterface $shippingPolicyRepository;

    public function __construct(
        CountryInformationAcquirerInterface $countryInformationAcquirer,
        ShippingPolicyRepositoryInterface $shippingPolicyRepository
    ) {
        $this->countryInformationAcquirer = $countryInformationAcquirer;
        $this->shippingPolicyRepository = $shippingPolicyRepository;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $countries = $this->countryInformationAcquirer->getCountriesInfo();
        $result = [];

        foreach ($countries as $country) {
            $countryData = [
                'id' => $country->getId(),
                'two_letter_abbreviation' => $country->getTwoLetterAbbreviation(),
                'three_letter_abbreviation' => $country->getThreeLetterAbbreviation(),
                'full_name_locale' => $country->getFullNameLocale(),
                'full_name_english' => $country->getFullNameEnglish(),
                'available_regions' => []
            ];

            // Add regions if available
            if ($country->getAvailableRegions()) {
                foreach ($country->getAvailableRegions() as $region) {
                    $countryData['available_regions'][] = [
                        'id' => $region->getId(),
                        'code' => $region->getCode(),
                        'name' => $region->getName()
                    ];
                }
            }

            // Add shipping policies count for this country
            $countryData['shipping_policies_count'] = $this->getShippingPoliciesCount(
                $country->getTwoLetterAbbreviation()
            );

            $result[] = $countryData;
        }

        return $result;
    }

    private function getShippingPoliciesCount(string $countryCode): int
    {
        try {
            $searchCriteria = $this->searchCriteriaBuilder
                ->addFilter('country_code', $countryCode)
                ->addFilter('is_active', 1)
                ->create();
            
            $result = $this->shippingPolicyRepository->getList($searchCriteria);
            return $result->getTotalCount();
        } catch (\Exception $e) {
            return 0;
        }
    }
}
```

### 2.4 Context & Authorization

**2.4.1 Context**

Context provides request-scoped data to resolvers:

```php
<?php
// Model/Context/CustomContext.php
namespace Tutorial\GraphQLDemo\Model\Context;

use Magento\GraphQl\Model\Query\ContextInterface;
use Magento\GraphQl\Model\Query\ContextExtensionInterface;

class CustomContext implements ContextInterface
{
    private ?int $userId;
    private ?int $userType;
    private ?string $userRole;
    private ContextExtensionInterface $extensionAttributes;

    public function __construct(
        ContextExtensionInterface $extensionAttributes,
        ?int $userId = null,
        ?int $userType = null,
        ?string $userRole = null
    ) {
        $this->extensionAttributes = $extensionAttributes;
        $this->userId = $userId;
        $this->userType = $userType;
        $this->userRole = $userRole;
    }

    public function getUserId(): ?int
    {
        return $this->userId;
    }

    public function getUserType(): ?int
    {
        return $this->userType;
    }

    public function getUserRole(): ?string
    {
        return $this->userRole;
    }

    public function getExtensionAttributes(): ContextExtensionInterface
    {
        return $this->extensionAttributes;
    }

    public function setExtensionAttributes(ContextExtensionInterface $extensionAttributes): void
    {
        $this->extensionAttributes = $extensionAttributes;
    }
}
```

**Context Factory:**
```php
<?php
// Model/Context/ContextFactory.php
namespace Tutorial\GraphQLDemo\Model\Context;

use Magento\Framework\GraphQl\Query\ContextFactoryInterface;
use Magento\Framework\App\Http\Context as HttpContext;
use Magento\Customer\Model\Context as CustomerContext;
use Magento\Authorization\Model\UserContextInterface;

class ContextFactory implements ContextFactoryInterface
{
    private HttpContext $httpContext;
    private UserContextInterface $userContext;

    public function __construct(
        HttpContext $httpContext,
        UserContextInterface $userContext
    ) {
        $this->httpContext = $httpContext;
        $this->userContext = $userContext;
    }

    public function create(): ContextInterface
    {
        $userId = $this->userContext->getUserId();
        $userType = $this->userContext->getUserType();
        
        // Determine user role
        $userRole = 'guest';
        if ($userType === UserContextInterface::USER_TYPE_CUSTOMER) {
            $userRole = 'customer';
        } elseif ($userType === UserContextInterface::USER_TYPE_ADMIN) {
            $userRole = 'admin';
        }

        return new CustomContext(
            $this->createExtensionAttributes(),
            $userId,
            $userType,
            $userRole
        );
    }

    private function createExtensionAttributes(): ContextExtensionInterface
    {
        // Create and return extension attributes
        return new ContextExtension();
    }
}
```

**2.4.2 Authorization**

Implement authorization for different user types:

```php
<?php
// Model/Authorization/PolicyAuthorization.php
namespace Tutorial\GraphQLDemo\Model\Authorization;

use Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException;
use Magento\Framework\GraphQl\Query\ContextInterface;
use Magento\Authorization\Model\UserContextInterface;

class PolicyAuthorization
{
    public function authorizeQuery(ContextInterface $context, string $operation): void
    {
        switch ($operation) {
            case 'read':
                $this->authorizeRead($context);
                break;
            case 'create':
            case 'update':
            case 'delete':
                $this->authorizeWrite($context);
                break;
            default:
                throw new GraphQlAuthorizationException(__('Unknown operation'));
        }
    }

    private function authorizeRead(ContextInterface $context): void
    {
        // Read operations allowed for all authenticated users
        if (!$this->isAuthenticated($context)) {
            throw new GraphQlAuthorizationException(
                __('Current user does not have access to this resource')
            );
        }
    }

    private function authorizeWrite(ContextInterface $context): void
    {
        // Write operations only for admin users
        if (!$this->isAdmin($context)) {
            throw new GraphQlAuthorizationException(
                __('Current user does not have permission to perform this operation')
            );
        }
    }

    private function isAuthenticated(ContextInterface $context): bool
    {
        $userType = $context->getUserType();
        return in_array($userType, [
            UserContextInterface::USER_TYPE_CUSTOMER,
            UserContextInterface::USER_TYPE_ADMIN
        ]);
    }

    private function isAdmin(ContextInterface $context): bool
    {
        return $context->getUserType() === UserContextInterface::USER_TYPE_ADMIN;
    }

    public function authorizeCustomerData(ContextInterface $context, int $customerId): void
    {
        $currentUserId = $context->getUserId();
        $userType = $context->getUserType();

        // Admin can access any customer data
        if ($userType === UserContextInterface::USER_TYPE_ADMIN) {
            return;
        }

        // Customer can only access their own data
        if ($userType === UserContextInterface::USER_TYPE_CUSTOMER && $currentUserId === $customerId) {
            return;
        }

        throw new GraphQlAuthorizationException(
            __('Current user does not have access to this customer data')
        );
    }
}
```

**2.4.3 User Context for Shipping Policies**

Customer-specific shipping policies:

```php
<?php
// Model/Resolver/CustomerShippingPolicies.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Customer\Api\CustomerRepositoryInterface;
use Tutorial\GraphQLDemo\Model\Authorization\PolicyAuthorization;

class CustomerShippingPolicies implements ResolverInterface
{
    private CustomerRepositoryInterface $customerRepository;
    private PolicyAuthorization $policyAuthorization;
    private ShippingPolicyRepositoryInterface $shippingPolicyRepository;

    public function __construct(
        CustomerRepositoryInterface $customerRepository,
        PolicyAuthorization $policyAuthorization,
        ShippingPolicyRepositoryInterface $shippingPolicyRepository
    ) {
        $this->customerRepository = $customerRepository;
        $this->policyAuthorization = $policyAuthorization;
        $this->shippingPolicyRepository = $shippingPolicyRepository;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $this->policyAuthorization->authorizeQuery($context, 'read');
        
        $customerId = $context->getUserId();
        $customer = $this->customerRepository->getById($customerId);
        
        // Get customer's addresses to determine relevant shipping policies
        $addresses = $customer->getAddresses();
        $relevantPolicies = [];
        
        foreach ($addresses as $address) {
            $countryId = $address->getCountryId();
            $regionCode = $address->getRegion() ? $address->getRegion()->getRegionCode() : null;
            
            $policies = $this->getShippingPoliciesForLocation($countryId, $regionCode);
            $relevantPolicies = array_merge($relevantPolicies, $policies);
        }

        // Remove duplicates and format for GraphQL
        $uniquePolicies = $this->removeDuplicatePolicies($relevantPolicies);
        
        return [
            'items' => $this->formatPoliciesForGraphQL($uniquePolicies),
            'total_count' => count($uniquePolicies)
        ];
    }

    private function getShippingPoliciesForLocation(string $countryCode, ?string $regionCode): array
    {
        $searchCriteria = $this->searchCriteriaBuilder
            ->addFilter('country_code', $countryCode)
            ->addFilter('is_active', 1);
            
        if ($regionCode) {
            $searchCriteria->addFilter('region_code', $regionCode);
        }
        
        $result = $this->shippingPolicyRepository->getList($searchCriteria->create());
        return $result->getItems();
    }

    private function removeDuplicatePolicies(array $policies): array
    {
        $unique = [];
        $seen = [];
        
        foreach ($policies as $policy) {
            $key = $policy->getId();
            if (!isset($seen[$key])) {
                $unique[] = $policy;
                $seen[$key] = true;
            }
        }
        
        return $unique;
    }

    private function formatPoliciesForGraphQL(array $policies): array
    {
        $result = [];
        foreach ($policies as $policy) {
            $result[] = [
                'id' => $policy->getId(),
                'name' => $policy->getName(),
                'country_code' => $policy->getCountryCode(),
                'region_code' => $policy->getRegionCode(),
                'min_order_amount' => $policy->getMinOrderAmount(),
                'shipping_cost' => $policy->getShippingCost(),
                'delivery_days' => $policy->getDeliveryDays(),
                'is_active' => (bool)$policy->getIsActive(),
                'model' => $policy
            ];
        }
        return $result;
    }
}
```

**2.4.4 Shipping Policy Callback Mutation**

Mutation with callback functionality:

```php
<?php
// Model/Resolver/ShippingPolicyCallback.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Framework\Event\ManagerInterface as EventManager;

class ShippingPolicyCallback implements ResolverInterface
{
    private EventManager $eventManager;
    private PolicyAuthorization $policyAuthorization;

    public function __construct(
        EventManager $eventManager,
        PolicyAuthorization $policyAuthorization
    ) {
        $this->eventManager = $eventManager;
        $this->policyAuthorization = $policyAuthorization;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $this->policyAuthorization->authorizeQuery($context, 'update');
        
        $policyId = $args['policy_id'];
        $callbackData = $args['callback_data'];
        
        try {
            // Process the callback
            $result = $this->processCallback($policyId, $callbackData, $context);
            
            // Dispatch event for other modules to listen
            $this->eventManager->dispatch('shipping_policy_callback_processed', [
                'policy_id' => $policyId,
                'callback_data' => $callbackData,
                'result' => $result,
                'context' => $context
            ]);
            
            return [
                'success' => true,
                'message' => __('Callback processed successfully'),
                'data' => $result
            ];
            
        } catch (\Exception $e) {
            return [
                'success' => false,
                'message' => __('Callback processing failed: %1', $e->getMessage()),
                'data' => null
            ];
        }
    }

    private function processCallback(int $policyId, array $callbackData, $context): array
    {
        // Implement your callback logic here
        // This could be webhook processing, external API calls, etc.
        
        $policy = $this->shippingPolicyRepository->getById($policyId);
        
        // Example: Update policy based on callback data
        if (isset($callbackData['status'])) {
            $policy->setIsActive($callbackData['status'] === 'active');
            $this->shippingPolicyRepository->save($policy);
        }
        
        return [
            'policy_id' => $policyId,
            'updated_fields' => array_keys($callbackData),
            'timestamp' => date('Y-m-d H:i:s')
        ];
    }
}
```

---

## Chapter 3: Caching

### 3.1 Caching Basics

**3.1.1 Caching Basics**

GraphQL caching in Magento works at multiple levels:

**1. Query Result Caching:**
- Full query results cached by Varnish/Redis
- Based on query string, variables, and context

**2. Resolver-Level Caching:**
- Individual resolver results cached
- Shared across similar queries

**3. Data Source Caching:**
- Model/collection caching
- Repository-level caching

**Cache Tags:**
```php
<?php
// Example cache tags implementation
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\GraphQlCache\Model\CacheableQueryInterface;

class CacheableShippingPolicies implements ResolverInterface, CacheableQueryInterface
{
    public function resolve(Field $field, $context, ResolveInfo $info, array $value = null, array $args = null)
    {
        // Resolution logic
        $result = $this->getPolicies($args);
        
        // Set cache tags for invalidation
        $this->addCacheTags($result);
        
        return $result;
    }

    public function getCacheTags(array $resolvedValue, array $args, ResolveInfo $info, array $value = null): array
    {
        $tags = ['shipping_policies'];
        
        // Add specific policy tags
        if (isset($resolvedValue['items'])) {
            foreach ($resolvedValue['items'] as $item) {
                $tags[] = sprintf('shipping_policy_%s', $item['id']);
                $tags[] = sprintf('shipping_policy_country_%s', $item['country_code']);
            }
        }
        
        return $tags;
    }
}
```

**3.1.2 Cache ID Factors**

Cache ID is built from multiple factors:

```php
<?php
// Model/Cache/IdentityFactory.php
namespace Tutorial\GraphQLDemo\Model\Cache;

use Magento\Framework\GraphQl\Query\ContextInterface;
use Magento\Framework\App\Http\Context as HttpContext;

class IdentityFactory
{
    private HttpContext $httpContext;

    public function __construct(HttpContext $httpContext)
    {
        $this->httpContext = $httpContext;
    }

    public function create(
        string $query,
        array $variables,
        ContextInterface $context
    ): string {
        $factors = [
            'query' => md5($query),
            'variables' => md5(serialize($variables)),
            'store_id' => $context->getExtensionAttributes()->getStore()->getId(),
            'customer_group' => $this->httpContext->getValue(\Magento\Customer\Model\Context::CONTEXT_GROUP),
            'currency' => $this->httpContext->getValue(\Magento\Store\Model\Store::ENTITY),
        ];

        // Add custom factors
        if ($context->getUserId()) {
            $factors['customer_id'] = $context->getUserId();
        }

        return md5(serialize($factors));
    }
}
```

**Cache Context:**
```php
<?php
// Model/Cache/Context.php
namespace Tutorial\GraphQLDemo\Model\Cache;

class Context
{
    const CACHE_TAG_SHIPPING_POLICIES = 'shipping_policies';
    const CACHE_TAG_SHIPPING_POLICY = 'shipping_policy_%s';
    const CACHE_TAG_COUNTRY = 'shipping_policy_country_%s';
    const CACHE_TAG_CUSTOMER = 'shipping_policy_customer_%s';

    public static function getPolicyTags(array $policies): array
    {
        $tags = [self::CACHE_TAG_SHIPPING_POLICIES];
        
        foreach ($policies as $policy) {
            $tags[] = sprintf(self::CACHE_TAG_SHIPPING_POLICY, $policy['id']);
            $tags[] = sprintf(self::CACHE_TAG_COUNTRY, $policy['country_code']);
        }
        
        return array_unique($tags);
    }

    public static function getCustomerTags(int $customerId): array
    {
        return [
            sprintf(self::CACHE_TAG_CUSTOMER, $customerId),
            self::CACHE_TAG_SHIPPING_POLICIES
        ];
    }
}
```

### 3.2 Varnish Configuration

**3.2.1 Hands-on - Prerequisites and Caveats**

**Prerequisites:**
- Varnish 6.0+ installed
- Magento configured for Varnish
- GraphQL queries properly tagged

**Caveats:**
- Customer-specific data should not be cached in Varnish
- Mutations should never be cached
- Sensitive data must be excluded

**3.2.2 Varnish Configuration**

**VCL Configuration for GraphQL:**
```vcl
# /etc/varnish/default.vcl

vcl 4.0;

import std;
import directors;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .first_byte_timeout = 600s;
    .between_bytes_timeout = 600s;
}

# GraphQL-specific ACL
acl purge {
    "localhost";
    "127.0.0.1";
    "::1";
}

sub vcl_recv {
    # Handle GraphQL requests
    if (req.url ~ "^/graphql") {
        # Only cache GET requests (queries, not mutations)
        if (req.method != "GET") {
            return (pass);
        }
        
        # Don't cache if customer is logged in
        if (req.http.Authorization) {
            return (pass);
        }
        
        # Don't cache if contains customer-specific data
        if (req.url ~ "customer|cart|wishlist") {
            return (pass);
        }
        
        # Normalize GraphQL query parameters
        call normalize_graphql_query;
        
        # Remove problematic headers
        unset req.http.Cookie;
        unset req.http.X-Forwarded-For;
        
        return (hash);
    }
    
    # Handle cache purge requests
    if (req.method == "PURGE") {
        if (!client.ip ~ purge) {
            return (synth(405, "Purging not allowed"));
        }
        return (purge);
    }
}

sub normalize_graphql_query {
    # Sort query parameters for consistent caching
    set req.url = std.querysort(req.url);
    
    # Remove timestamp and other varying parameters
    set req.url = regsuball(req.url, "&?_=[0-9]+", "");
}

sub vcl_backend_response {
    # Cache GraphQL responses
    if (bereq.url ~ "^/graphql" && bereq.method == "GET") {
        # Check if response is cacheable
        if (beresp.http.X-Magento-Tags) {
            set beresp.ttl = 1h;  # Cache for 1 hour
            set beresp.grace = 30s;
            
            # Store cache tags for selective purging
            set beresp.http.X-Cache-Tags = beresp.http.X-Magento-Tags;
        }
    }
    
    # Remove sensitive headers
    unset beresp.http.Set-Cookie;
    unset beresp.http.X-Magento-Debug;
}

sub vcl_deliver {
    # Add cache hit/miss header
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }
    
    # Remove internal headers
    unset resp.http.X-Cache-Tags;
    unset resp.http.X-Magento-Tags;
}

sub vcl_purge {
    # Handle tag-based purging
    if (req.http.X-Purge-Tags) {
        ban("beresp.http.X-Cache-Tags ~ " + req.http.X-Purge-Tags);
        return (synth(200, "Purged by tags"));
    }
    
    return (synth(200, "Purged"));
}
```

**3.2.3 Trying a Cacheable Query**

**Cacheable Query Example:**
```graphql
query GetCacheableShippingPolicies {
  shippingPolicies(
    filter: {
      country_code: {eq: "US"}
      is_active: {eq: "1"}
    }
    pageSize: 10
  ) {
    items {
      id
      name
      country_code
      shipping_cost
      delivery_days
    }
    total_count
  }
}
```

**Testing Cache Behavior:**
```bash
# First request (MISS)
curl -X GET \
  "https://your-site.com/graphql?query=query{shippingPolicies{items{id,name}}}" \
  -H "Accept: application/json"

# Second request (HIT)
curl -X GET \
  "https://your-site.com/graphql?query=query{shippingPolicies{items{id,name}}}" \
  -H "Accept: application/json"

# Check cache headers
curl -I -X GET \
  "https://your-site.com/graphql?query=query{shippingPolicies{items{id,name}}}" \
  -H "Accept: application/json"
```

### 3.3 Making Queries Cacheable

**3.3.1 Making a Query Cacheable**

Implement the `CacheableQueryInterface`:

```php
<?php
// Model/Resolver/CacheableShippingPolicies.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\GraphQlCache\Model\CacheableQueryInterface;

class CacheableShippingPolicies implements ResolverInterface, CacheableQueryInterface
{
    private ShippingPolicyRepositoryInterface $repository;

    public function __construct(ShippingPolicyRepositoryInterface $repository)
    {
        $this->repository = $repository;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $searchCriteria = $this->buildSearchCriteria($args);
        $result = $this->repository->getList($searchCriteria);
        
        $items = [];
        foreach ($result->getItems() as $policy) {
            $items[] = [
                'id' => $policy->getId(),
                'name' => $policy->getName(),
                'country_code' => $policy->getCountryCode(),
                'region_code' => $policy->getRegionCode(),
                'shipping_cost' => $policy->getShippingCost(),
                'delivery_days' => $policy->getDeliveryDays(),
                'is_active' => (bool)$policy->getIsActive(),
                'model' => $policy
            ];
        }

        return [
            'items' => $items,
            'total_count' => $result->getTotalCount(),
            'page_info' => $this->buildPageInfo($args, $result->getTotalCount())
        ];
    }

    public function getCacheTags(
        array $resolvedValue,
        array $args,
        ResolveInfo $info,
        array $value = null
    ): array {
        $tags = ['shipping_policies'];
        
        // Add tags for each policy
        foreach ($resolvedValue['items'] as $item) {
            $tags[] = sprintf('shipping_policy_%s', $item['id']);
            $tags[] = sprintf('country_%s', $item['country_code']);
            
            if ($item['region_code']) {
                $tags[] = sprintf('region_%s_%s', $item['country_code'], $item['region_code']);
            }
        }

        // Add filter-based tags
        if (isset($args['filter']['country_code']['eq'])) {
            $tags[] = sprintf('country_%s', $args['filter']['country_code']['eq']);
        }

        return array_unique($tags);
    }
}
```

**Cache Tag Manager:**
```php
<?php
// Model/Cache/TagManager.php
namespace Tutorial\GraphQLDemo\Model\Cache;

use Magento\Framework\App\Cache\TagInterface;
use Magento\Framework\App\CacheInterface;

class TagManager
{
    private CacheInterface $cache;

    public function __construct(CacheInterface $cache)
    {
        $this->cache = $cache;
    }

    public function invalidateShippingPolicy(int $policyId): void
    {
        $tags = [
            sprintf('shipping_policy_%s', $policyId),
            'shipping_policies'
        ];
        
        $this->cache->clean($tags);
    }

    public function invalidateCountryPolicies(string $countryCode): void
    {
        $tags = [
            sprintf('country_%s', $countryCode),
            'shipping_policies'
        ];
        
        $this->cache->clean($tags);
    }

    public function invalidateRegionPolicies(string $countryCode, string $regionCode): void
    {
        $tags = [
            sprintf('region_%s_%s', $countryCode, $regionCode),
            sprintf('country_%s', $countryCode),
            'shipping_policies'
        ];
        
        $this->cache->clean($tags);
    }

    public function invalidateAllPolicies(): void
    {
        $this->cache->clean(['shipping_policies']);
    }
}
```

**3.3.2 Caching Shipping Policies**

**Observer for Cache Invalidation:**
```php
<?php
// Observer/InvalidateShippingPolicyCache.php
namespace Tutorial\GraphQLDemo\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;
use Tutorial\GraphQLDemo\Model\Cache\TagManager;

class InvalidateShippingPolicyCache implements ObserverInterface
{
    private TagManager $tagManager;

    public function __construct(TagManager $tagManager)
    {
        $this->tagManager = $tagManager;
    }

    public function execute(Observer $observer): void
    {
        $policy = $observer->getEvent()->getShippingPolicy();
        
        if ($policy && $policy->getId()) {
            // Invalidate specific policy cache
            $this->tagManager->invalidateShippingPolicy($policy->getId());
            
            // Invalidate country-specific cache
            $this->tagManager->invalidateCountryPolicies($policy->getCountryCode());
            
            // Invalidate region-specific cache if applicable
            if ($policy->getRegionCode()) {
                $this->tagManager->invalidateRegionPolicies(
                    $policy->getCountryCode(),
                    $policy->getRegionCode()
                );
            }
        }
    }
}
```

**Register Observer:**
```xml
<!-- etc/events.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="shipping_policy_save_after">
        <observer name="invalidate_shipping_policy_cache" 
                  instance="Tutorial\GraphQLDemo\Observer\InvalidateShippingPolicyCache"/>
    </event>
    <event name="shipping_policy_delete_after">
        <observer name="invalidate_shipping_policy_cache" 
                  instance="Tutorial\GraphQLDemo\Observer\InvalidateShippingPolicyCache"/>
    </event>
</config>
```

**Cache Identity Provider:**
```php
<?php
// Model/Cache/IdentityProvider.php
namespace Tutorial\GraphQLDemo\Model\Cache;

use Magento\Framework\GraphQl\Query\ContextInterface;

class IdentityProvider
{
    public function getIdentityForShippingPolicies(
        array $args,
        ContextInterface $context
    ): string {
        $factors = [
            'query_type' => 'shipping_policies',
            'store_id' => $context->getExtensionAttributes()->getStore()->getId(),
            'filters' => $args['filter'] ?? [],
            'page_size' => $args['pageSize'] ?? 20,
            'current_page' => $args['currentPage'] ?? 1
        ];

        // Add customer group if relevant
        $customerGroupId = $context->getExtensionAttributes()->getCustomerGroupId();
        if ($customerGroupId) {
            $factors['customer_group_id'] = $customerGroupId;
        }

        return md5(serialize($factors));
    }

    public function getVaryData(ContextInterface $context): array
    {
        return [
            'store' => $context->getExtensionAttributes()->getStore()->getCode(),
            'customer_group' => $context->getExtensionAttributes()->getCustomerGroupId() ?? 0,
            'currency' => $context->getExtensionAttributes()->getStore()->getCurrentCurrencyCode()
        ];
    }
}
```

---

## Chapter 4: Advanced Concepts

### 4.1 Deep Dives

**4.1.1 Deep Dives - Intro**

Advanced GraphQL concepts in Magento:

1. **Schema Building Process** - How Magento builds the GraphQL schema
2. **Query Execution Pipeline** - Step-by-step query processing
3. **Type Resolution** - Dynamic type resolution for interfaces/unions
4. **Performance Optimization** - N+1 queries, batching, and optimization
5. **Security Considerations** - Query complexity, depth limiting, introspection

**4.1.2 Deep Dive - Config Reading**

Understanding how Magento reads GraphQL configuration:

```php
<?php
// Deep dive into schema configuration reading
namespace Tutorial\GraphQLDemo\Model\Config;

use Magento\Framework\Config\ReaderInterface;
use Magento\Framework\GraphQl\Schema\Type\TypeRegistry;

class SchemaReader implements ReaderInterface
{
    private TypeRegistry $typeRegistry;
    private array $schemaFiles;

    public function __construct(
        TypeRegistry $typeRegistry,
        array $schemaFiles = []
    ) {
        $this->typeRegistry = $typeRegistry;
        $this->schemaFiles = $schemaFiles;
    }

    public function read($scope = null): array
    {
        $schema = [];
        
        // Read all .graphqls files
        foreach ($this->schemaFiles as $file) {
            $fileContent = file_get_contents($file);
            $parsedSchema = $this->parseGraphQLSchema($fileContent);
            $schema = array_merge_recursive($schema, $parsedSchema);
        }

        return $this->processSchema($schema);
    }

    private function parseGraphQLSchema(string $content): array
    {
        // Parse GraphQL schema definition
        $parser = new \GraphQL\Language\Parser();
        $ast = $parser->parse($content);
        
        return $this->convertAstToArray($ast);
    }

    private function processSchema(array $schema): array
    {
        // Process type definitions
        $processedSchema = [
            'types' => [],
            'queries' => [],
            'mutations' => [],
            'subscriptions' => []
        ];

        foreach ($schema['types'] as $typeName => $typeConfig) {
            $processedSchema['types'][$typeName] = $this->processTypeDefinition($typeConfig);
        }

        return $processedSchema;
    }

    private function processTypeDefinition(array $typeConfig): array
    {
        // Process fields, resolvers, and other type metadata
        $processed = [
            'name' => $typeConfig['name'],
            'fields' => [],
            'interfaces' => $typeConfig['interfaces'] ?? [],
            'resolver' => $typeConfig['resolver'] ?? null
        ];

        foreach ($typeConfig['fields'] as $fieldName => $fieldConfig) {
            $processed['fields'][$fieldName] = $this->processFieldDefinition($fieldConfig);
        }

        return $processed;
    }

    private function processFieldDefinition(array $fieldConfig): array
    {
        return [
            'type' => $fieldConfig['type'],
            'args' => $fieldConfig['args'] ?? [],
            'resolver' => $fieldConfig['resolver'] ?? null,
            'description' => $fieldConfig['description'] ?? null,
            'deprecationReason' => $fieldConfig['deprecationReason'] ?? null
        ];
    }
}
```

**Configuration Merger:**
```php
<?php
// Model/Config/SchemaMerger.php
namespace Tutorial\GraphQLDemo\Model\Config;

class SchemaMerger
{
    public function merge(array $schemas): array
    {
        $merged = [
            'types' => [],
            'queries' => [],
            'mutations' => [],
            'subscriptions' => []
        ];

        foreach ($schemas as $schema) {
            // Merge types
            foreach ($schema['types'] ?? [] as $typeName => $typeConfig) {
                if (isset($merged['types'][$typeName])) {
                    $merged['types'][$typeName] = $this->mergeTypes(
                        $merged['types'][$typeName],
                        $typeConfig
                    );
                } else {
                    $merged['types'][$typeName] = $typeConfig;
                }
            }

            // Merge root types
            foreach (['queries', 'mutations', 'subscriptions'] as $rootType) {
                if (isset($schema[$rootType])) {
                    $merged[$rootType] = array_merge(
                        $merged[$rootType],
                        $schema[$rootType]
                    );
                }
            }
        }

        return $merged;
    }

    private function mergeTypes(array $existing, array $new): array
    {
        // Merge type fields
        $existing['fields'] = array_merge($existing['fields'], $new['fields']);
        
        // Merge interfaces
        if (isset($new['interfaces'])) {
            $existing['interfaces'] = array_unique(
                array_merge($existing['interfaces'] ?? [], $new['interfaces'])
            );
        }

        return $existing;
    }
}
```

**4.1.3 Deep Dive - Schema Building**

Schema building process in detail:

```php
<?php
// Model/Schema/Builder.php
namespace Tutorial\GraphQLDemo\Model\Schema;

use GraphQL\Type\Schema;
use GraphQL\Type\Definition\ObjectType;
use GraphQL\Type\Definition\Type;
use Magento\Framework\GraphQl\Schema\Type\TypeRegistry;

class Builder
{
    private TypeRegistry $typeRegistry;
    private array $config;

    public function __construct(
        TypeRegistry $typeRegistry,
        array $config
    ) {
        $this->typeRegistry = $typeRegistry;
        $this->config = $config;
    }

    public function build(): Schema
    {
        return new Schema([
            'query' => $this->buildQueryType(),
            'mutation' => $this->buildMutationType(),
            'subscription' => $this->buildSubscriptionType(),
            'types' => $this->buildTypes(),
            'typeLoader' => [$this, 'loadType']
        ]);
    }

    private function buildQueryType(): ObjectType
    {
        return new ObjectType([
            'name' => 'Query',
            'fields' => $this->buildQueryFields()
        ]);
    }

    private function buildMutationType(): ?ObjectType
    {
        $mutationFields = $this->buildMutationFields();
        
        if (empty($mutationFields)) {
            return null;
        }

        return new ObjectType([
            'name' => 'Mutation',
            'fields' => $mutationFields
        ]);
    }

    private function buildSubscriptionType(): ?ObjectType
    {
        $subscriptionFields = $this->buildSubscriptionFields();
        
        if (empty($subscriptionFields)) {
            return null;
        }

        return new ObjectType([
            'name' => 'Subscription',
            'fields' => $subscriptionFields
        ]);
    }

    private function buildQueryFields(): array
    {
        $fields = [];
        
        foreach ($this->config['queries'] as $fieldName => $fieldConfig) {
            $fields[$fieldName] = [
                'type' => $this->resolveType($fieldConfig['type']),
                'args' => $this->buildArgs($fieldConfig['args'] ?? []),
                'resolve' => $this->createResolver($fieldConfig['resolver'] ?? null),
                'description' => $fieldConfig['description'] ?? null
            ];
        }
        
        return $fields;
    }

    private function buildMutationFields(): array
    {
        $fields = [];
        
        foreach ($this->config['mutations'] as $fieldName => $fieldConfig) {
            $fields[$fieldName] = [
                'type' => $this->resolveType($fieldConfig['type']),
                'args' => $this->buildArgs($fieldConfig['args'] ?? []),
                'resolve' => $this->createResolver($fieldConfig['resolver'] ?? null),
                'description' => $fieldConfig['description'] ?? null
            ];
        }
        
        return $fields;
    }

    private function buildTypes(): array
    {
        $types = [];
        
        foreach ($this->config['types'] as $typeName => $typeConfig) {
            $types[] = $this->buildType($typeName, $typeConfig);
        }
        
        return $types;
    }

    private function buildType(string $typeName, array $typeConfig): Type
    {
        return new ObjectType([
            'name' => $typeName,
            'fields' => function() use ($typeConfig) {
                return $this->buildTypeFields($typeConfig['fields']);
            },
            'interfaces' => $this->resolveInterfaces($typeConfig['interfaces'] ?? []),
            'description' => $typeConfig['description'] ?? null
        ]);
    }

    private function buildTypeFields(array $fieldsConfig): array
    {
        $fields = [];
        
        foreach ($fieldsConfig as $fieldName => $fieldConfig) {
            $fields[$fieldName] = [
                'type' => $this->resolveType($fieldConfig['type']),
                'args' => $this->buildArgs($fieldConfig['args'] ?? []),
                'resolve' => $this->createFieldResolver($fieldConfig),
                'description' => $fieldConfig['description'] ?? null
            ];
        }
        
        return $fields;
    }

    private function createResolver(?string $resolverClass): ?callable
    {
        if (!$resolverClass) {
            return null;
        }

        return function($root, $args, $context, $info) use ($resolverClass) {
            $resolver = $this->objectManager->get($resolverClass);
            return $resolver->resolve($info->fieldDefinition, $context, $info, $root, $args);
        };
    }

    private function createFieldResolver(array $fieldConfig): ?callable
    {
        if (isset($fieldConfig['resolver'])) {
            return $this->createResolver($fieldConfig['resolver']);
        }

        // Default field resolver
        return function($root, $args, $context, $info) {
            $fieldName = $info->fieldName;
            
            if (is_array($root) && isset($root[$fieldName])) {
                return $root[$fieldName];
            }
            
            if (is_object($root) && method_exists($root, 'getData')) {
                return $root->getData($fieldName);
            }
            
            if (is_object($root) && property_exists($root, $fieldName)) {
                return $root->$fieldName;
            }
            
            return null;
        };
    }

    public function loadType(string $typeName): ?Type
    {
        return $this->typeRegistry->get($typeName);
    }
}
```

**4.1.4 Deep Dive - Query Execution**

Query execution pipeline in Magento GraphQL:

```php
<?php
// Model/Execution/QueryProcessor.php
namespace Tutorial\GraphQLDemo\Model\Execution;

use GraphQL\GraphQL;
use GraphQL\Type\Schema;
use GraphQL\Execution\ExecutionResult;
use Magento\Framework\GraphQl\Query\ContextInterface;

class QueryProcessor
{
    private Schema $schema;
    private SecurityValidator $securityValidator;
    private PerformanceMonitor $performanceMonitor;

    public function __construct(
        Schema $schema,
        SecurityValidator $securityValidator,
        PerformanceMonitor $performanceMonitor
    ) {
        $this->schema = $schema;
        $this->securityValidator = $securityValidator;
        $this->performanceMonitor = $performanceMonitor;
    }

    public function execute(
        string $query,
        array $variables,
        ContextInterface $context,
        ?string $operationName = null
    ): ExecutionResult {
        
        // 1. Parse and validate query
        $this->securityValidator->validateQuery($query, $variables, $context);
        
        // 2. Start performance monitoring
        $this->performanceMonitor->start($query, $variables);
        
        try {
            // 3. Execute query
            $result = GraphQL::executeQuery(
                $this->schema,
                $query,
                null, // rootValue
                $context, // contextValue
                $variables,
                $operationName,
                null, // fieldResolver
                $this->createValidationRules($context)
            );
            
            // 4. Post-process result
            $result = $this->postProcessResult($result, $context);
            
        } finally {
            // 5. End performance monitoring
            $this->performanceMonitor->end();
        }
        
        return $result;
    }

    private function createValidationRules(ContextInterface $context): array
    {
        return [
            new \GraphQL\Validation\Rules\QueryDepth(15), // Max depth 15
            new \GraphQL\Validation\Rules\QueryComplexity(1000), // Max complexity 1000
            new \GraphQL\Validation\Rules\DisableIntrospection() // Disable in production
        ];
    }

    private function postProcessResult(ExecutionResult $result, ContextInterface $context): ExecutionResult
    {
        // Add cache headers
        if ($result->data && !$result->errors) {
            $this->addCacheHeaders($result, $context);
        }
        
        // Add debug information in development
        if ($this->isDevelopmentMode()) {
            $this->addDebugInfo($result);
        }
        
        return $result;
    }

    private function addCacheHeaders(ExecutionResult $result, ContextInterface $context): void
    {
        // Implementation for cache headers
        $headers = $context->getExtensionAttributes()->getHeaders();
        
        if ($this->isCacheable($result)) {
            $headers->set('Cache-Control', 'public, max-age=3600');
            $headers->set('X-Magento-Tags', $this->getCacheTags($result));
        } else {
            $headers->set('Cache-Control', 'no-cache, no-store, must-revalidate');
        }
    }
}
```

**Query Execution Steps:**
```php
<?php
// Model/Execution/ExecutionSteps.php
namespace Tutorial\GraphQLDemo\Model\Execution;

class ExecutionSteps
{
    /**
     * Step 1: Query Parsing
     */
    public function parseQuery(string $query): \GraphQL\Language\AST\DocumentNode
    {
        try {
            return \GraphQL\Language\Parser::parse($query);
        } catch (\GraphQL\Error\SyntaxError $e) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('GraphQL syntax error: %1', $e->getMessage())
            );
        }
    }

    /**
     * Step 2: Query Validation
     */
    public function validateQuery(
        \GraphQL\Language\AST\DocumentNode $document,
        \GraphQL\Type\Schema $schema,
        array $validationRules
    ): array {
        return \GraphQL\Validation\DocumentValidator::validate(
            $schema,
            $document,
            $validationRules
        );
    }

    /**
     * Step 3: Query Execution Planning
     */
    public function createExecutionPlan(
        \GraphQL\Language\AST\DocumentNode $document,
        \GraphQL\Type\Schema $schema,
        array $variables
    ): array {
        $plan = [];
        
        foreach ($document->definitions as $definition) {
            if ($definition instanceof \GraphQL\Language\AST\OperationDefinitionNode) {
                $plan[] = $this->planOperation($definition, $schema, $variables);
            }
        }
        
        return $plan;
    }

    /**
     * Step 4: Field Resolution
     */
    public function resolveField(
        string $fieldName,
        $rootValue,
        array $args,
        $context,
        \GraphQL\Type\Definition\ResolveInfo $info
    ) {
        $resolver = $this->getFieldResolver($info->parentType, $fieldName);
        
        if ($resolver) {
            return $resolver($rootValue, $args, $context, $info);
        }
        
        return $this->defaultFieldResolver($rootValue, $args, $context, $info);
    }

    /**
     * Step 5: Result Serialization
     */
    public function serializeResult($result, \GraphQL\Type\Definition\Type $type): mixed
    {
        if ($type instanceof \GraphQL\Type\Definition\ScalarType) {
            return $type->serialize($result);
        }
        
        if ($type instanceof \GraphQL\Type\Definition\ObjectType) {
            return $this->serializeObject($result, $type);
        }
        
        if ($type instanceof \GraphQL\Type\Definition\ListOfType) {
            return $this->serializeList($result, $type);
        }
        
        return $result;
    }

    private function planOperation(
        \GraphQL\Language\AST\OperationDefinitionNode $operation,
        \GraphQL\Type\Schema $schema,
        array $variables
    ): array {
        return [
            'operation' => $operation->operation,
            'name' => $operation->name ? $operation->name->value : null,
            'fields' => $this->planSelectionSet($operation->selectionSet, $schema, $variables),
            'variables' => $this->resolveVariables($operation->variableDefinitions ?? [], $variables)
        ];
    }

    private function planSelectionSet(
        \GraphQL\Language\AST\SelectionSetNode $selectionSet,
        \GraphQL\Type\Schema $schema,
        array $variables
    ): array {
        $fields = [];
        
        foreach ($selectionSet->selections as $selection) {
            if ($selection instanceof \GraphQL\Language\AST\FieldNode) {
                $fields[] = $this->planField($selection, $schema, $variables);
            }
        }
        
        return $fields;
    }
}
```

### 4.2 Product Attributes & Types

**4.2.1 Product Attribute Readers**

Product attribute system integration:

```php
<?php
// Model/Resolver/ProductAttributeReader.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Catalog\Api\ProductAttributeRepositoryInterface;
use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Eav\Api\AttributeRepositoryInterface;

class ProductAttributeReader implements ResolverInterface
{
    private ProductAttributeRepositoryInterface $attributeRepository;
    private AttributeRepositoryInterface $eavAttributeRepository;

    public function __construct(
        ProductAttributeRepositoryInterface $attributeRepository,
        AttributeRepositoryInterface $eavAttributeRepository
    ) {
        $this->attributeRepository = $attributeRepository;
        $this->eavAttributeRepository = $eavAttributeRepository;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $attributeCode = $args['attribute_code'] ?? null;
        
        if ($attributeCode) {
            return $this->getSingleAttribute($attributeCode);
        }
        
        return $this->getAllProductAttributes();
    }

    private function getSingleAttribute(string $attributeCode): array
    {
        try {
            $attribute = $this->attributeRepository->get($attributeCode);
            return $this->formatAttributeData($attribute);
        } catch (\Exception $e) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlNoSuchEntityException(
                __('Attribute with code "%1" not found', $attributeCode)
            );
        }
    }

    private function getAllProductAttributes(): array
    {
        $searchCriteria = $this->searchCriteriaBuilder->create();
        $searchResult = $this->eavAttributeRepository->getList(
            \Magento\Catalog\Api\Data\ProductAttributeInterface::ENTITY_TYPE_CODE,
            $searchCriteria
        );

        $attributes = [];
        foreach ($searchResult->getItems() as $attribute) {
            $attributes[] = $this->formatAttributeData($attribute);
        }

        return [
            'items' => $attributes,
            'total_count' => $searchResult->getTotalCount()
        ];
    }

    private function formatAttributeData($attribute): array
    {
        return [
            'attribute_id' => $attribute->getAttributeId(),
            'attribute_code' => $attribute->getAttributeCode(),
            'frontend_label' => $attribute->getFrontendLabel(),
            'frontend_input' => $attribute->getFrontendInput(),
            'is_required' => $attribute->getIsRequired(),
            'is_unique' => $attribute->getIsUnique(),
            'is_searchable' => $attribute->getIsSearchable(),
            'is_filterable' => $attribute->getIsFilterable(),
            'is_filterable_in_search' => $attribute->getIsFilterableInSearch(),
            'is_used_for_promo_rules' => $attribute->getIsUsedForPromoRules(),
            'is_visible_on_front' => $attribute->getIsVisibleOnFront(),
            'used_in_product_listing' => $attribute->getUsedInProductListing(),
            'used_for_sort_by' => $attribute->getUsedForSortBy(),
            'default_value' => $attribute->getDefaultValue(),
            'backend_type' => $attribute->getBackendType(),
            'backend_model' => $attribute->getBackendModel(),
            'source_model' => $attribute->getSourceModel(),
            'options' => $this->getAttributeOptions($attribute),
            'validation_rules' => $this->getValidationRules($attribute),
            'apply_to' => $attribute->getApplyTo()
        ];
    }

    private function getAttributeOptions($attribute): array
    {
        $options = [];
        
        if ($attribute->getSource()) {
            foreach ($attribute->getSource()->getAllOptions() as $option) {
                if ($option['value'] !== '') {
                    $options[] = [
                        'value' => $option['value'],
                        'label' => $option['label'],
                        'is_default' => false // Would need additional logic
                    ];
                }
            }
        }
        
        return $options;
    }

    private function getValidationRules($attribute): array
    {
        $rules = [];
        
        if ($attribute->getFrontendClass()) {
            $rules['frontend_class'] = $attribute->getFrontendClass();
        }
        
        if ($attribute->getIsRequired()) {
            $rules['required'] = true;
        }
        
        // Add specific validation rules based on frontend input
        switch ($attribute->getFrontendInput()) {
            case 'text':
                if ($attribute->getData('input_validation')) {
                    $rules['input_validation'] = $attribute->getData('input_validation');
                }
                break;
            case 'textarea':
                if ($attribute->getData('wysiwyg_enabled')) {
                    $rules['wysiwyg_enabled'] = (bool)$attribute->getData('wysiwyg_enabled');
                }
                break;
            case 'date':
                $rules['date_format'] = 'Y-m-d';
                break;
        }
        
        return $rules;
    }
}
```

**Enhanced Product Attribute Schema:**
```graphql
# etc/schema.graphqls - Product Attributes
extend type Query {
    productAttributes(
        filter: ProductAttributeFilterInput @doc(description: "Filter criteria")
        pageSize: Int = 20 @doc(description: "Number of items per page")
        currentPage: Int = 1 @doc(description: "Current page number")
    ): ProductAttributesOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\ProductAttributeReader") @doc(description: "Get product attributes")
    
    productAttribute(
        attribute_code: String! @doc(description: "Attribute code")
    ): ProductAttributeDetails @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\ProductAttributeReader") @doc(description: "Get single product attribute")
}

type ProductAttributesOutput @doc(description: "Product attributes output") {
    items: [ProductAttributeDetails] @doc(description: "Attribute items")
    total_count: Int @doc(description: "Total count")
    page_info: SearchResultPageInfo @doc(description: "Page info")
}

type ProductAttributeDetails @doc(description: "Product attribute details") {
    attribute_id: Int @doc(description: "Attribute ID")
    attribute_code: String @doc(description: "Attribute code")
    frontend_label: String @doc(description: "Frontend label")
    frontend_input: String @doc(description: "Frontend input type")
    is_required: Boolean @doc(description: "Is required")
    is_unique: Boolean @doc(description: "Is unique")
    is_searchable: Boolean @doc(description: "Is searchable")
    is_filterable: Boolean @doc(description: "Is filterable")
    is_filterable_in_search: Boolean @doc(description: "Is filterable in search")
    is_used_for_promo_rules: Boolean @doc(description: "Is used for promo rules")
    is_visible_on_front: Boolean @doc(description: "Is visible on front")
    used_in_product_listing: Boolean @doc(description: "Used in product listing")
    used_for_sort_by: Boolean @doc(description: "Used for sort by")
    default_value: String @doc(description: "Default value")
    backend_type: String @doc(description: "Backend type")
    backend_model: String @doc(description: "Backend model")
    source_model: String @doc(description: "Source model")
    options: [AttributeOption] @doc(description: "Attribute options")
    validation_rules: AttributeValidationRules @doc(description: "Validation rules")
    apply_to: [String] @doc(description: "Apply to product types")
}

type AttributeOption @doc(description: "Attribute option") {
    value: String @doc(description: "Option value")
    label: String @doc(description: "Option label")
    is_default: Boolean @doc(description: "Is default option")
}

type AttributeValidationRules @doc(description: "Attribute validation rules") {
    frontend_class: String @doc(description: "Frontend class")
    required: Boolean @doc(description: "Is required")
    input_validation: String @doc(description: "Input validation")
    wysiwyg_enabled: Boolean @doc(description: "WYSIWYG enabled")
    date_format: String @doc(description: "Date format")
}

input ProductAttributeFilterInput @doc(description: "Product attribute filter") {
    attribute_code: FilterTypeInput @doc(description: "Attribute code filter")
    frontend_input: FilterTypeInput @doc(description: "Frontend input filter")
    is_searchable: FilterTypeInput @doc(description: "Is searchable filter")
    is_filterable: FilterTypeInput @doc(description: "Is filterable filter")
    used_in_product_listing: FilterTypeInput @doc(description: "Used in listing filter")
}
```

**4.2.2 Product Types**

Enhanced product type handling:

```php
<?php
// Model/Resolver/ProductTypeDetails.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Catalog\Model\Product\Type as ProductType;

class ProductTypeDetails implements ResolverInterface
{
    private ProductType $productType;
    private array $productTypeInstances;

    public function __construct(
        ProductType $productType,
        array $productTypeInstances = []
    ) {
        $this->productType = $productType;
        $this->productTypeInstances = $productTypeInstances;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        $typeId = $args['type_id'] ?? null;
        
        if ($typeId) {
            return $this->getSingleProductType($typeId);
        }
        
        return $this->getAllProductTypes();
    }

    private function getSingleProductType(string $typeId): array
    {
        $types = $this->productType->getTypes();
        
        if (!isset($types[$typeId])) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlNoSuchEntityException(
                __('Product type "%1" not found', $typeId)
            );
        }
        
        return $this->formatProductTypeData($typeId, $types[$typeId]);
    }

    private function getAllProductTypes(): array
    {
        $types = $this->productType->getTypes();
        $formattedTypes = [];
        
        foreach ($types as $typeId => $typeConfig) {
            $formattedTypes[] = $this->formatProductTypeData($typeId, $typeConfig);
        }
        
        return [
            'items' => $formattedTypes,
            'total_count' => count($formattedTypes)
        ];
    }

    private function formatProductTypeData(string $typeId, array $typeConfig): array
    {
        $typeInstance = $this->getProductTypeInstance($typeId);
        
        return [
            'type_id' => $typeId,
            'label' => $typeConfig['label'] ?? $typeId,
            'model_instance' => $typeConfig['model'] ?? null,
            'composite' => $typeInstance ? $typeInstance->isComposite() : false,
            'can_use_qty_decimals' => $typeInstance ? $typeInstance->canUseQtyDecimals() : false,
            'is_qty' => $typeInstance ? $typeInstance->isQty() : true,
            'delete_type_restricted' => $typeInstance ? $typeInstance->deleteTypeRestrictedAttribute() : false,
            'allowed_attributes' => $this->getAllowedAttributes($typeId),
            'required_attributes' => $this->getRequiredAttributes($typeId),
            'sellable_attributes' => $this->getSellableAttributes($typeId),
            'price_model' => $this->getPriceModel($typeId),
            'index_priority' => $typeConfig['index_priority'] ?? 0,
            'sort_order' => $typeConfig['sort_order'] ?? 0
        ];
    }

    private function getProductTypeInstance(string $typeId): ?\Magento\Catalog\Model\Product\Type\AbstractType
    {
        if (isset($this->productTypeInstances[$typeId])) {
            return $this->productTypeInstances[$typeId];
        }
        
        try {
            return $this->productType->factory($typeId);
        } catch (\Exception $e) {
            return null;
        }
    }

    private function getAllowedAttributes(string $typeId): array
    {
        $typeInstance = $this->getProductTypeInstance($typeId);
        
        if (!$typeInstance) {
            return [];
        }
        
        $attributes = [];
        foreach ($typeInstance->getEditableAttributes() as $attribute) {
            $attributes[] = [
                'attribute_code' => $attribute->getAttributeCode(),
                'frontend_label' => $attribute->getFrontendLabel(),
                'is_required' => $attribute->getIsRequired(),
                'is_user_defined' => $attribute->getIsUserDefined()
            ];
        }
        
        return $attributes;
    }

    private function getRequiredAttributes(string $typeId): array
    {
        $typeInstance = $this->getProductTypeInstance($typeId);
        
        if (!$typeInstance) {
            return [];
        }
        
        $attributes = [];
        foreach ($typeInstance->getEditableAttributes() as $attribute) {
            if ($attribute->getIsRequired()) {
                $attributes[] = [
                    'attribute_code' => $attribute->getAttributeCode(),
                    'frontend_label' => $attribute->getFrontendLabel(),
                    'frontend_input' => $attribute->getFrontendInput()
                ];
            }
        }
        
        return $attributes;
    }

    private function getSellableAttributes(string $typeId): array
    {
        $typeInstance = $this->getProductTypeInstance($typeId);
        
        if (!$typeInstance) {
            return [];
        }
        
        $sellableAttributes = $typeInstance->getSellableAttributes();
        $attributes = [];
        
        foreach ($sellableAttributes as $attribute) {
            $attributes[] = [
                'attribute_code' => $attribute->getAttributeCode(),
                'frontend_label' => $attribute->getFrontendLabel()
            ];
        }
        
        return $attributes;
    }

    private function getPriceModel(string $typeId): ?array
    {
        $typeInstance = $this->getProductTypeInstance($typeId);
        
        if (!$typeInstance) {
            return null;
        }
        
        $priceModel = $typeInstance->getPriceModel();
        
        return [
            'class' => get_class($priceModel),
            'supports_tier_pricing' => method_exists($priceModel, 'getTierPrice'),
            'supports_special_pricing' => method_exists($priceModel, 'getSpecialPrice'),
            'supports_group_pricing' => method_exists($priceModel, 'getGroupPrice')
        ];
    }
}
```

**Product Type Schema:**
```graphql
# Product Types Schema Extension
extend type Query {
    productTypes(
        filter: ProductTypeFilterInput @doc(description: "Filter criteria")
    ): ProductTypesOutput @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\ProductTypeDetails") @doc(description: "Get product types")
    
    productType(
        type_id: String! @doc(description: "Product type ID")
    ): ProductTypeDetails @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\ProductTypeDetails") @doc(description: "Get single product type")
}

type ProductTypesOutput @doc(description: "Product types output") {
    items: [ProductTypeDetails] @doc(description: "Product type items")
    total_count: Int @doc(description: "Total count")
}

type ProductTypeDetails @doc(description: "Product type details") {
    type_id: String @doc(description: "Type ID")
    label: String @doc(description: "Type label")
    model_instance: String @doc(description: "Model instance class")
    composite: Boolean @doc(description: "Is composite product")
    can_use_qty_decimals: Boolean @doc(description: "Can use quantity decimals")
    is_qty: Boolean @doc(description: "Uses quantity")
    delete_type_restricted: Boolean @doc(description: "Delete type restricted")
    allowed_attributes: [ProductTypeAttribute] @doc(description: "Allowed attributes")
    required_attributes: [ProductTypeAttribute] @doc(description: "Required attributes")
    sellable_attributes: [ProductTypeAttribute] @doc(description: "Sellable attributes")
    price_model: ProductTypePriceModel @doc(description: "Price model details")
    index_priority: Int @doc(description: "Index priority")
    sort_order: Int @doc(description: "Sort order")
}

type ProductTypeAttribute @doc(description: "Product type attribute") {
    attribute_code: String @doc(description: "Attribute code")
    frontend_label: String @doc(description: "Frontend label")
    is_required: Boolean @doc(description: "Is required")
    is_user_defined: Boolean @doc(description: "Is user defined")
    frontend_input: String @doc(description: "Frontend input type")
}

type ProductTypePriceModel @doc(description: "Product type price model") {
    class: String @doc(description: "Price model class")
    supports_tier_pricing: Boolean @doc(description: "Supports tier pricing")
    supports_special_pricing: Boolean @doc(description: "Supports special pricing")
    supports_group_pricing: Boolean @doc(description: "Supports group pricing")
}

input ProductTypeFilterInput @doc(description: "Product type filter") {
    type_id: FilterTypeInput @doc(description: "Type ID filter")
    composite: FilterTypeInput @doc(description: "Composite filter")
    can_use_qty_decimals: FilterTypeInput @doc(description: "Quantity decimals filter")
}
```

### 4.3 Type Resolvers & Staging

**4.3.1 Type Resolvers**

Advanced type resolution for interfaces and unions:

```php
<?php
// Model/TypeResolver/ProductTypeResolver.php
namespace Tutorial\GraphQLDemo\Model\TypeResolver;

use Magento\Framework\GraphQl\Query\Resolver\TypeResolverInterface;

class ProductTypeResolver implements TypeResolverInterface
{
    private array $supportedTypes = [
        'simple' => 'SimpleProduct',
        'configurable' => 'ConfigurableProduct',
        'grouped' => 'GroupedProduct',
        'bundle' => 'BundleProduct',
        'virtual' => 'VirtualProduct',
        'downloadable' => 'DownloadableProduct'
    ];

    public function resolveType(array $data): string
    {
        if (!isset($data['type_id'])) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Missing product type_id for type resolution')
            );
        }

        $typeId = $data['type_id'];
        
        if (!isset($this->supportedTypes[$typeId])) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Unsupported product type: %1', $typeId)
            );
        }

        return $this->supportedTypes[$typeId];
    }
}
```

**Custom Union Type Resolver:**
```php
<?php
// Model/TypeResolver/SearchResultTypeResolver.php
namespace Tutorial\GraphQLDemo\Model\TypeResolver;

use Magento\Framework\GraphQl\Query\Resolver\TypeResolverInterface;

class SearchResultTypeResolver implements TypeResolverInterface
{
    public function resolveType(array $data): string
    {
        if (isset($data['entity_type'])) {
            switch ($data['entity_type']) {
                case 'product':
                    return $this->resolveProductType($data);
                case 'category':
                    return 'CategoryTree';
                case 'cms_page':
                    return 'CmsPage';
                case 'cms_block':
                    return 'CmsBlock';
                case 'shipping_policy':
                    return 'ShippingPolicy';
                default:
                    break;
            }
        }

        // Fallback resolution based on data structure
        if (isset($data['sku'])) {
            return $this->resolveProductType($data);
        }
        
        if (isset($data['children_count'])) {
            return 'CategoryTree';
        }
        
        if (isset($data['page_layout'])) {
            return 'CmsPage';
        }

        throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
            __('Cannot resolve type for search result')
        );
    }

    private function resolveProductType(array $data): string
    {
        $productTypeResolver = new ProductTypeResolver();
        return $productTypeResolver->resolveType($data);
    }
}
```

**Interface Type Resolver:**
```php
<?php
// Model/TypeResolver/CustomizableProductInterface.php
namespace Tutorial\GraphQLDemo\Model\TypeResolver;

use Magento\Framework\GraphQl\Query\Resolver\TypeResolverInterface;

class CustomizableProductInterface implements TypeResolverInterface
{
    public function resolveType(array $data): string
    {
        if (!isset($data['type_id'])) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Missing product type_id for customizable product resolution')
            );
        }

        // Only certain product types support customization
        $customizableTypes = [
            'simple' => 'SimpleProduct',
            'virtual' => 'VirtualProduct',
            'downloadable' => 'DownloadableProduct'
        ];

        $typeId = $data['type_id'];
        
        if (isset($customizableTypes[$typeId])) {
            return $customizableTypes[$typeId];
        }

        throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
            __('Product type %1 does not support customization', $typeId)
        );
    }
}
```

**Union and Interface Schema:**
```graphql
# Advanced Type System Schema
union SearchResult = ProductInterface | CategoryTree | CmsPage | CmsBlock | ShippingPolicy

interface CustomizableProductInterface {
    id: Int
    sku: String
    name: String
    options: [CustomizableOptionInterface]
}

interface PhysicalProductInterface {
    weight: Float
    dimensions: ProductDimensions
    shipping_class: String
}

type ProductDimensions @doc(description: "Product dimensions") {
    length: Float @doc(description: "Length")
    width: Float @doc(description: "Width")
    height: Float @doc(description: "Height")
    unit: String @doc(description: "Measurement unit")
}

# Type resolver registration
type SimpleProduct implements ProductInterface & CustomizableProductInterface & PhysicalProductInterface {
    # All fields from interfaces plus simple-specific fields
    weight: Float
    dimensions: ProductDimensions
    shipping_class: String
    options: [CustomizableOptionInterface]
}

type ConfigurableProduct implements ProductInterface & PhysicalProductInterface {
    # Configurable products don't implement CustomizableProductInterface
    weight: Float
    dimensions: ProductDimensions
    shipping_class: String
    configurable_options: [ConfigurableProductOptions]
    variants: [ConfigurableVariant]
}
```

**4.3.2 Staging**

Enterprise staging integration for scheduled content:

```php
<?php
// Model/Resolver/StagingAwareShippingPolicies.php
namespace Tutorial\GraphQLDemo\Model\Resolver;

use Magento\Framework\GraphQl\Config\Element\Field;
use Magento\Framework\GraphQl\Query\ResolverInterface;
use Magento\Framework\GraphQl\Schema\Type\ResolveInfo;
use Magento\Staging\Model\VersionManager;

class StagingAwareShippingPolicies implements ResolverInterface
{
    private VersionManager $versionManager;
    private ShippingPolicyRepositoryInterface $repository;

    public function __construct(
        VersionManager $versionManager,
        ShippingPolicyRepositoryInterface $repository
    ) {
        $this->versionManager = $versionManager;
        $this->repository = $repository;
    }

    public function resolve(
        Field $field,
        $context,
        ResolveInfo $info,
        array $value = null,
        array $args = null
    ) {
        // Check if preview mode is enabled
        $previewVersion = $this->getPreviewVersion($args);
        
        if ($previewVersion) {
            $this->versionManager->setCurrentVersionId($previewVersion);
        }

        try {
            $searchCriteria = $this->buildSearchCriteria($args);
            $result = $this->repository->getList($searchCriteria);
            
            $items = [];
            foreach ($result->getItems() as $policy) {
                $policyData = $this->formatPolicyData($policy);
                
                // Add staging information
                if ($this->isStagingEnabled()) {
                    $policyData['staging_info'] = $this->getStagingInfo($policy);
                }
                
                $items[] = $policyData;
            }

            return [
                'items' => $items,
                'total_count' => $result->getTotalCount(),
                'staging_version' => $this->versionManager->getCurrentVersionId()
            ];
            
        } finally {
            // Reset version manager
            if ($previewVersion) {
                $this->versionManager->setCurrentVersionId(1);
            }
        }
    }

    private function getPreviewVersion(array $args): ?int
    {
        return isset($args['preview_version']) ? (int)$args['preview_version'] : null;
    }

    private function getStagingInfo($policy): array
    {
        $stagingInfo = [
            'is_staged' => false,
            'start_time' => null,
            'end_time' => null,
            'update_name' => null,
            'rollback_id' => null
        ];

        if (!$this->isStagingEnabled()) {
            return $stagingInfo;
        }

        try {
            // Get staging metadata
            $versionInfo = $this->versionManager->getVersionMetaData($policy->getId());
            
            if ($versionInfo) {
                $stagingInfo = [
                    'is_staged' => true,
                    'start_time' => $versionInfo['start_time'] ?? null,
                    'end_time' => $versionInfo['end_time'] ?? null,
                    'update_name' => $versionInfo['name'] ?? null,
                    'rollback_id' => $versionInfo['rollback_id'] ?? null,
                    'created_in' => $versionInfo['created_in'] ?? null,
                    'updated_in' => $versionInfo['updated_in'] ?? null
                ];
            }
        } catch (\Exception $e) {
            // Staging info not available
        }

        return $stagingInfo;
    }

    private function isStagingEnabled(): bool
    {
        return class_exists('\Magento\Staging\Model\VersionManager');
    }
}
```

**Staging Schema Extensions:**
```graphql
# Staging-aware schema extensions
extend type ShippingPolicy {
    staging_info: StagingInfo @doc(description: "Staging information")
}

extend type ShippingPoliciesOutput {
    staging_version: Int @doc(description: "Current staging version")
}

extend input ShippingPolicyFilterInput {
    preview_version: Int @doc(description: "Preview version ID")
    include_staged: Boolean @doc(description: "Include staged content")
}

type StagingInfo @doc(description: "Content staging information") {
    is_staged: Boolean @doc(description: "Is content staged")
    start_time: String @doc(description: "Start time")
    end_time: String @doc(description: "End time")
    update_name: String @doc(description: "Update name")
    rollback_id: Int @doc(description: "Rollback ID")
    created_in: Int @doc(description: "Created in version")
    updated_in: Int @doc(description: "Updated in version")
}

extend type Query {
    stagingVersions: [StagingVersion] @resolver(class: "Tutorial\\GraphQLDemo\\Model\\Resolver\\StagingVersions") @doc(description: "Get staging versions")
}

type StagingVersion @doc(description: "Staging version") {
    id: Int @doc(description: "Version ID")
    name: String @doc(description: "Version name")
    start_time: String @doc(description: "Start time")
    end_time: String @doc(description: "End time")
    status: String @doc(description: "Version status")
    is_rollback: Boolean @doc(description: "Is rollback version")
}
```

### 4.4 Query Limits & Security

**4.4.1 Query Limits**

Implement query complexity and depth limiting:

```php
<?php
// Model/Security/QueryComplexityAnalyzer.php
namespace Tutorial\GraphQLDemo\Model\Security;

use GraphQL\Validation\Rules\QueryComplexity;
use GraphQL\Type\Definition\FieldDefinition;
use GraphQL\Type\Definition\Type;

class QueryComplexityAnalyzer extends QueryComplexity
{
    private int $maxComplexity;
    private array $fieldComplexityMap;

    public function __construct(
        int $maxComplexity = 1000,
        array $fieldComplexityMap = []
    ) {
        $this->maxComplexity = $maxComplexity;
        $this->fieldComplexityMap = $fieldComplexityMap;
        
        parent::__construct($maxComplexity, $this->createComplexityAnalyzer());
    }

    private function createComplexityAnalyzer(): callable
    {
        return function (int $childrenComplexity, array $args, FieldDefinition $field): int {
            // Get base complexity for this field
            $baseComplexity = $this->getFieldComplexity($field->name);
            
            // Calculate complexity based on arguments
            $argComplexity = $this->calculateArgumentComplexity($args);
            
            // Apply multipliers for list fields
            $listMultiplier = $this->getListMultiplier($field->getType(), $args);
            
            $totalComplexity = ($baseComplexity + $argComplexity + $childrenComplexity) * $listMultiplier;
            
            return $totalComplexity;
        };
    }

    private function getFieldComplexity(string $fieldName): int
    {
        // Define complexity for specific fields
        $complexityMap = array_merge([
            'products' => 10,
            'categories' => 5,
            'shippingPolicies' => 3,
            'storeConfig' => 1,
            'cart' => 8,
            'customer' => 5,
            'countries' => 2
        ], $this->fieldComplexityMap);

        return $complexityMap[$fieldName] ?? 1;
    }

    private function calculateArgumentComplexity(array $args): int
    {
        $complexity = 0;
        
        // Add complexity for filters
        if (isset($args['filter'])) {
            $complexity += count($args['filter']) * 2;
        }
        
        // Add complexity for sorting
        if (isset($args['sort'])) {
            $complexity += count($args['sort']);
        }
        
        // Add complexity for search terms
        if (isset($args['search'])) {
            $complexity += 5; // Search operations are expensive
        }
        
        return $complexity;
    }

    private function getListMultiplier(Type $type, array $args): int
    {
        if (!Type::isListType($type)) {
            return 1;
        }
        
        // Use pageSize as multiplier for list types
        $pageSize = $args['pageSize'] ?? 20;
        
        // Cap the multiplier to prevent abuse
        return min($pageSize, 100);
    }
}
```

**Query Depth Analyzer:**
```php
<?php
// Model/Security/QueryDepthAnalyzer.php
namespace Tutorial\GraphQLDemo\Model\Security;

use GraphQL\Validation\Rules\QueryDepth;
use GraphQL\Language\AST\FieldNode;
use GraphQL\Language\AST\NodeKind;

class QueryDepthAnalyzer extends QueryDepth
{
    private int $maxDepth;
    private array $depthExceptions;

    public function __construct(
        int $maxDepth = 15,
        array $depthExceptions = []
    ) {
        $this->maxDepth = $maxDepth;
        $this->depthExceptions = $depthExceptions;
        
        parent::__construct($maxDepth);
    }

    protected function getFieldDepth(FieldNode $node, int $depth = 0): int
    {
        // Check if this field has depth exceptions
        $fieldName = $node->name->value;
        
        if (isset($this->depthExceptions[$fieldName])) {
            $maxAllowedDepth = $this->depthExceptions[$fieldName];
            
            if ($depth > $maxAllowedDepth) {
                throw new \GraphQL\Error\Error(
                    sprintf('Field "%s" exceeds maximum allowed depth of %d', $fieldName, $maxAllowedDepth),
                    [$node]
                );
            }
        }

        // Calculate depth for selections
        $maxChildDepth = 0;
        
        if ($node->selectionSet) {
            foreach ($node->selectionSet->selections as $selection) {
                if ($selection->kind === NodeKind::FIELD) {
                    $childDepth = $this->getFieldDepth($selection, $depth + 1);
                    $maxChildDepth = max($maxChildDepth, $childDepth);
                }
            }
        }

        return $depth + $maxChildDepth;
    }
}
```

**Rate Limiting:**
```php
<?php
// Model/Security/RateLimiter.php
namespace Tutorial\GraphQLDemo\Model\Security;

use Magento\Framework\HTTP\PhpEnvironment\RemoteAddress;
use Magento\Framework\App\CacheInterface;

class RateLimiter
{
    private CacheInterface $cache;
    private RemoteAddress $remoteAddress;
    private int $maxRequestsPerMinute;
    private int $maxRequestsPerHour;

    public function __construct(
        CacheInterface $cache,
        RemoteAddress $remoteAddress,
        int $maxRequestsPerMinute = 60,
        int $maxRequestsPerHour = 1000
    ) {
        $this->cache = $cache;
        $this->remoteAddress = $remoteAddress;
        $this->maxRequestsPerMinute = $maxRequestsPerMinute;
        $this->maxRequestsPerHour = $maxRequestsPerHour;
    }

    public function checkRateLimit(string $query, array $variables): void
    {
        $clientIp = $this->remoteAddress->getRemoteAddress();
        $userId = $this->getCurrentUserId();
        
        // Create rate limit keys
        $keys = [
            sprintf('rate_limit_ip_minute_%s_%s', $clientIp, date('Y-m-d-H-i')),
            sprintf('rate_limit_ip_hour_%s_%s', $clientIp, date('Y-m-d-H')),
        ];
        
        if ($userId) {
            $keys[] = sprintf('rate_limit_user_minute_%s_%s', $userId, date('Y-m-d-H-i'));
            $keys[] = sprintf('rate_limit_user_hour_%s_%s', $userId, date('Y-m-d-H'));
        }

        // Check limits
        foreach ($keys as $key) {
            $this->checkLimit($key, $query, $variables);
        }
    }

    private function checkLimit(string $key, string $query, array $variables): void
    {
        $currentCount = (int)$this->cache->load($key) ?: 0;
        $isMinuteKey = strpos($key, '_minute_') !== false;
        $limit = $isMinuteKey ? $this->maxRequestsPerMinute : $this->maxRequestsPerHour;
        
        if ($currentCount >= $limit) {
            $timeWindow = $isMinuteKey ? 'minute' : 'hour';
            throw new \Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException(
                __('Rate limit exceeded: %1 requests per %2', $limit, $timeWindow)
            );
        }

        // Increment counter
        $ttl = $isMinuteKey ? 60 : 3600;
        $this->cache->save($currentCount + 1, $key, [], $ttl);
    }

    private function getCurrentUserId(): ?int
    {
        // Get current user ID from context
        // Implementation depends on your authentication system
        return null;
    }
}
```

**4.4.2 Disabling Introspection**

Disable introspection in production:

```php
<?php
// Model/Security/IntrospectionControl.php
namespace Tutorial\GraphQLDemo\Model\Security;

use GraphQL\Validation\Rules\DisableIntrospection;
use Magento\Framework\App\State;

class IntrospectionControl
{
    private State $appState;
    private bool $forceDisable;

    public function __construct(
        State $appState,
        bool $forceDisable = false
    ) {
        $this->appState = $appState;
        $this->forceDisable = $forceDisable;
    }

    public function getIntrospectionRule(): ?DisableIntrospection
    {
        // Always disable in production or if forced
        if ($this->shouldDisableIntrospection()) {
            return new DisableIntrospection(DisableIntrospection::ENABLED);
        }

        return null;
    }

    private function shouldDisableIntrospection(): bool
    {
        if ($this->forceDisable) {
            return true;
        }

        try {
            $mode = $this->appState->getMode();
            return $mode === State::MODE_PRODUCTION;
        } catch (\Exception $e) {
            // If we can't determine mode, err on the side of caution
            return true;
        }
    }

    public function validateIntrospectionQuery(string $query): void
    {
        if ($this->shouldDisableIntrospection() && $this->isIntrospectionQuery($query)) {
            throw new \Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException(
                __('GraphQL introspection is disabled')
            );
        }
    }

    private function isIntrospectionQuery(string $query): bool
    {
        $introspectionPatterns = [
            '/__schema/',
            '/__type/',
            '/__typename/',
            '/__directive/',
            '/__enumValue/',
            '/__field/',
            '/__inputValue/'
        ];

        foreach ($introspectionPatterns as $pattern) {
            if (preg_match($pattern, $query)) {
                return true;
            }
        }

        return false;
    }
}
```

**Security Middleware:**
```php
<?php
// Model/Security/SecurityMiddleware.php
namespace Tutorial\GraphQLDemo\Model\Security;

use Magento\Framework\GraphQl\Query\ContextInterface;

class SecurityMiddleware
{
    private QueryComplexityAnalyzer $complexityAnalyzer;
    private QueryDepthAnalyzer $depthAnalyzer;
    private RateLimiter $rateLimiter;
    private IntrospectionControl $introspectionControl;

    public function __construct(
        QueryComplexityAnalyzer $complexityAnalyzer,
        QueryDepthAnalyzer $depthAnalyzer,
        RateLimiter $rateLimiter,
        IntrospectionControl $introspectionControl
    ) {
        $this->complexityAnalyzer = $complexityAnalyzer;
        $this->depthAnalyzer = $depthAnalyzer;
        $this->rateLimiter = $rateLimiter;
        $this->introspectionControl = $introspectionControl;
    }

    public function processRequest(
        string $query,
        array $variables,
        ContextInterface $context
    ): void {
        // 1. Check rate limits
        $this->rateLimiter->checkRateLimit($query, $variables);
        
        // 2. Validate introspection
        $this->introspectionControl->validateIntrospectionQuery($query);
        
        // 3. Additional security checks can be added here
        $this->validateQuerySecurity($query, $variables, $context);
    }

    public function getValidationRules(): array
    {
        $rules = [
            $this->complexityAnalyzer,
            $this->depthAnalyzer
        ];

        $introspectionRule = $this->introspectionControl->getIntrospectionRule();
        if ($introspectionRule) {
            $rules[] = $introspectionRule;
        }

        return $rules;
    }

    private function validateQuerySecurity(
        string $query,
        array $variables,
        ContextInterface $context
    ): void {
        // Check for suspicious patterns
        $suspiciousPatterns = [
            '/union\s+select/i',  // SQL injection attempt
            '/script\s*>/i',      // XSS attempt
            '/javascript:/i',     // XSS attempt
        ];

        foreach ($suspiciousPatterns as $pattern) {
            if (preg_match($pattern, $query) || preg_match($pattern, serialize($variables))) {
                throw new \Magento\Framework\GraphQl\Exception\GraphQlAuthorizationException(
                    __('Suspicious query detected')
                );
            }
        }

        // Check for excessively large queries
        if (strlen($query) > 10000) { // 10KB limit
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Query too large')
            );
        }

        // Check variable size
        if (strlen(serialize($variables)) > 5000) { // 5KB limit for variables
            throw new \Magento\Framework\GraphQl\Exception\GraphQlInputException(
                __('Variables payload too large')
            );
        }
    }
}
```

---

## Conclusion

### Final Exercise

**Comprehensive GraphQL Module Exercise:**

Create a complete customer loyalty program management system using GraphQL:

**Requirements:**
1. **Customer Points Management**
   - Query customer points balance
   - Mutation to add/redeem points
   - Points history with pagination

2. **Reward Catalog**
   - Query available rewards
   - Filter by point cost and category
   - Redeem rewards mutation

3. **Tier System**
   - Query customer tier status
   - Tier benefits and requirements
   - Tier progression tracking

**Implementation Checklist:**
- [ ] Create complete schema with queries and mutations
- [ ] Implement all resolvers with proper error handling
- [ ] Add caching with appropriate cache tags
- [ ] Implement security and authorization
- [ ] Add rate limiting and query complexity analysis
- [ ] Support for PWA/GraphQL and traditional frontend
- [ ] Comprehensive logging and monitoring
- [ ] Unit and integration tests

**Sample Implementation Structure:**
```php
<?php
// Final exercise starter code
namespace YourCompany\LoyaltyProgram\Model\Resolver;

class CustomerLoyaltyProgram implements ResolverInterface, CacheableQueryInterface
{
    public function resolve(Field $field, $context, ResolveInfo $info, array $value = null, array $args = null)
    {
        // Implement comprehensive loyalty program logic
        $customerId = $this->getCustomerId($context);
        
        return [
            'points_balance' => $this->getPointsBalance($customerId),
            'tier' => $this->getCurrentTier($customerId),
            'available_rewards' => $this->getAvailableRewards($customerId, $args),
            'points_history' => $this->getPointsHistory($customerId, $args),
            'tier_progress' => $this->getTierProgress($customerId)
        ];
    }

    public function getCacheTags(array $resolvedValue, array $args, ResolveInfo $info, array $value = null): array
    {
        return [
            sprintf('customer_loyalty_%s', $resolvedValue['customer_id']),
            'loyalty_rewards',
            'loyalty_tiers'
        ];
    }
}
```

### Congratulations!

🎉 **Congratulations on completing the Complete GraphQL in Magento Tutorial!**

You have successfully learned:

✅ **GraphQL Fundamentals** - Queries, mutations, schema design  
✅ **Magento Integration** - Resolvers, context, caching  
✅ **Advanced Concepts** - Type resolution, staging, security  
✅ **Best Practices** - Performance, caching, authorization  
✅ **Real-World Examples** - Complete working implementations  

**What's Next?**
- Implement your own GraphQL modules
- Contribute to Magento's GraphQL ecosystem
- Explore PWA integrations
- Build headless commerce solutions

**Resources for Continued Learning:**
- Magento GraphQL Documentation
- GraphQL Specification
- Magento Community Forums
- PWA Studio Documentation

**Happy Coding! 🚀**

---

## Additional Resources

### Sample Queries Collection

**Store Information:**
```graphql
query GetStoreInfo {
  storeConfig {
    store_name
    base_url
    locale
    timezone
    base_currency_code
  }
}
```

**Product Search:**
```graphql
query SearchProducts($search: String!, $pageSize: Int = 12) {
  products(search: $search, pageSize: $pageSize) {
    total_count
    items {
      name
      sku
      price_range {
        minimum_price {
          regular_price {
            value
            currency
          }
        }
      }
      image {
        url
        label
      }
    }
  }
}
```

**Complete Cart Operations:**
```graphql
# Create cart
mutation CreateCart {
  createEmptyCart
}

# Add product
mutation AddToCart($cartId: String!, $sku: String!, $qty: Float!) {
  addSimpleProductsToCart(
    input: {
      cart_id: $cartId
      cart_items: [{ data: { quantity: $qty, sku: $sku } }]
    }
  ) {
    cart {
      id
      items {
        id
        quantity
        product {
          name
          sku
        }
      }
      prices {
        grand_total {
          value
          currency
        }
      }
    }
  }
}
```

This concludes our comprehensive GraphQL in Magento tutorial. You now have the knowledge and tools to build sophisticated GraphQL integrations in Magento 2!