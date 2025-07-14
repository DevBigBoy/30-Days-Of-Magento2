# Magento Configurable Products Complete Guide

## What is a Configurable Product?

A **Configurable Product** in Magento is a complex product type that allows customers to choose from different variations (like size, color, material) of the same base product. It's essentially a "parent" product that groups multiple "child" simple products together, where each child represents a specific combination of attributes.

### Real-World Example
Think of a T-shirt that comes in:
- **Colors**: Black, Blue, Red
- **Sizes**: XS, S, M, L

Instead of creating separate product listings for each combination (Black-XS, Black-S, Blue-M, etc.), you create:
- **1 Configurable Product** (the parent) - "Cassius Sparring Tank"
- **Multiple Simple Products** (the children) - one for each size/color combination

## Why Use Configurable Products?

### Benefits:
- **Better UX**: Customers see one product page with dropdown selections
- **Easier Management**: Manage one product listing instead of dozens
- **SEO Friendly**: One URL instead of multiple product pages
- **Inventory Control**: Track stock for each specific variant
- **Pricing Flexibility**: Different prices for different variants

### Use Cases:
- Clothing (size, color, material)
- Electronics (storage size, color, model)
- Furniture (color, material, size)
- Books (format: hardcover, paperback, digital)

---

## Breaking Down Your Example

Let's analyze each attribute in your configurable product example:

### Parent Product Attributes

```php
$configurableProduct['entity_id'] = 50;
$configurableProduct['sku'] = 'WS03-CONFIG';
$configurableProduct['name'] = 'Cassius Sparring Tank';
$configurableProduct['type_id'] = 'configurable';
$configurableProduct['price'] = 29.00;
```

#### Explanation:

| Attribute | Value | Purpose |
|-----------|-------|---------|
| `entity_id` | `50` | **Unique Database ID** - Internal Magento identifier for this product |
| `sku` | `'WS03-CONFIG'` | **Stock Keeping Unit** - Unique identifier for inventory management |
| `name` | `'Cassius Sparring Tank'` | **Product Name** - What customers see on the frontend |
| `type_id` | `'configurable'` | **Product Type** - Tells Magento this is a configurable product |
| `price` | `29.00` | **Base Price** - Starting price (can be overridden by variants) |

### Configurable Options Structure

```php
$configurableProduct['configurable_options'] = [
    'color' => [
        'attribute_id' => 93,
        'attribute_code' => 'color',
        'label' => 'Color',
        'values' => [
            ['value_index' => 49, 'label' => 'Black'],
            ['value_index' => 50, 'label' => 'Blue'],
            ['value_index' => 51, 'label' => 'Red']
        ]
    ],
    'size' => [
        'attribute_id' => 142,
        'attribute_code' => 'size', 
        'label' => 'Size',
        'values' => [
            ['value_index' => 166, 'label' => 'XS'],
            ['value_index' => 167, 'label' => 'S'],
            ['value_index' => 168, 'label' => 'M'],
            ['value_index' => 169, 'label' => 'L']
        ]
    ]
];
```

#### Color Attribute Breakdown:

| Attribute | Value | Purpose |
|-----------|-------|---------|
| `attribute_id` | `93` | **Database ID** of the color attribute in `eav_attribute` table |
| `attribute_code` | `'color'` | **Code Name** used in code and API calls |
| `label` | `'Color'` | **Frontend Label** shown to customers |
| `values` | `array` | **Available Options** for this attribute |

#### Color Values Breakdown:

| Field | Value | Purpose |
|-------|-------|---------|
| `value_index` | `49, 50, 51` | **Option ID** from `eav_attribute_option_value` table |
| `label` | `'Black', 'Blue', 'Red'` | **Display Text** shown in dropdown |

#### Size Attribute (Same Structure):
- Works identically to color but with size-specific values
- Each size option has its own `value_index` and `label`

### Child Products (Variants)

```php
$configurableProduct['variants'] = [
    [
        'entity_id' => 51,
        'sku' => 'WS03-BLACK-XS',
        'price' => 29.00,
        'color' => 'Black',
        'size' => 'XS',
        'qty' => 25
    ],
    [
        'entity_id' => 52,
        'sku' => 'WS03-BLACK-S', 
        'price' => 29.00,
        'color' => 'Black',
        'size' => 'S',
        'qty' => 30
    ],
    [
        'entity_id' => 53,
        'sku' => 'WS03-BLUE-M',
        'price' => 31.00,
        'color' => 'Blue', 
        'size' => 'M',
        'qty' => 15
    ]
];
```

#### Child Product Attributes:

| Attribute | Purpose | Example Values |
|-----------|---------|----------------|
| `entity_id` | **Unique ID** for each variant | `51, 52, 53` |
| `sku` | **Unique SKU** following naming convention | `'WS03-BLACK-XS'` |
| `price` | **Variant-specific price** (can differ from parent) | `29.00, 31.00` |
| `color` | **Actual color value** for this variant | `'Black', 'Blue'` |
| `size` | **Actual size value** for this variant | `'XS', 'S', 'M'` |
| `qty` | **Stock quantity** for this specific variant | `25, 30, 15` |

---

## Complete Configurable Product Attributes

Here's a comprehensive list of all possible configurable product attributes:

### Core Product Attributes
```php
$configurableProduct = [
    // Basic Information
    'entity_id' => 50,
    'sku' => 'WS03-CONFIG',
    'name' => 'Cassius Sparring Tank',
    'type_id' => 'configurable',
    'attribute_set_id' => 9, // Product attribute set
    'status' => 1, // 1 = Enabled, 2 = Disabled
    'visibility' => 4, // 1=Not Visible, 2=Catalog, 3=Search, 4=Catalog+Search
    
    // Pricing
    'price' => 29.00,
    'special_price' => null,
    'special_from_date' => null,
    'special_to_date' => null,
    'cost' => 15.00,
    'msrp' => 35.00, // Manufacturer's Suggested Retail Price
    
    // Inventory
    'manage_stock' => 0, // Configurable products don't manage stock directly
    'is_in_stock' => 1,
    'stock_status' => 1,
    
    // SEO & URLs
    'url_key' => 'cassius-sparring-tank',
    'url_path' => 'cassius-sparring-tank.html',
    'meta_title' => 'Cassius Sparring Tank - Premium Athletic Wear',
    'meta_keyword' => 'tank top, athletic wear, sparring',
    'meta_description' => 'High-quality sparring tank in multiple colors and sizes',
    
    // Content
    'description' => 'Premium quality sparring tank made from breathable fabric...',
    'short_description' => 'Comfortable sparring tank for athletic training',
    
    // Media
    'image' => '/w/s/ws03-black-xs.jpg',
    'small_image' => '/w/s/ws03-black-xs.jpg',
    'thumbnail' => '/w/s/ws03-black-xs.jpg',
    'media_gallery' => [
        ['file' => '/w/s/ws03-black-xs.jpg', 'label' => 'Black XS'],
        ['file' => '/w/s/ws03-blue-m.jpg', 'label' => 'Blue M']
    ],
    
    // Dates
    'created_at' => '2024-01-15 10:30:00',
    'updated_at' => '2024-07-12 14:20:00',
    'new_from_date' => null,
    'new_to_date' => null,
    
    // Categories
    'category_ids' => [11, 12, 20], // Array of category IDs
    
    // Physical Properties
    'weight' => 0.5,
    'dimensions' => [
        'length' => 24,
        'width' => 18,
        'height' => 2
    ],
    
    // Custom Attributes
    'material' => 'Cotton Blend',
    'care_instructions' => 'Machine wash cold',
    'country_of_manufacture' => 'US',
    'gender' => 'Unisex',
    'activity' => 'Training',
    
    // Configuration-Specific
    'configurable_options' => [...], // As shown above
    'variants' => [...], // As shown above
    'associated_product_ids' => [51, 52, 53], // IDs of child products
];
```

### Extended Configurable Options
```php
'configurable_options' => [
    'color' => [
        'attribute_id' => 93,
        'attribute_code' => 'color',
        'label' => 'Color',
        'position' => 0, // Display order
        'is_use_default' => false,
        'values' => [
            [
                'value_index' => 49,
                'label' => 'Black',
                'default_label' => 'Black',
                'store_label' => 'Black',
                'use_default_value' => true
            ]
        ]
    ],
    'size' => [
        'attribute_id' => 142,
        'attribute_code' => 'size',
        'label' => 'Size',
        'position' => 1,
        'is_use_default' => false,
        'values' => [
            [
                'value_index' => 166,
                'label' => 'XS',
                'default_label' => 'Extra Small',
                'store_label' => 'XS',
                'use_default_value' => true
            ]
        ]
    ]
];
```

### Complete Child Product Structure
```php
'variants' => [
    [
        // Basic Info
        'entity_id' => 51,
        'sku' => 'WS03-BLACK-XS',
        'name' => 'Cassius Sparring Tank - Black XS',
        'type_id' => 'simple',
        'parent_id' => 50, // Reference to configurable product
        'status' => 1,
        'visibility' => 1, // Not visible individually
        
        // Pricing
        'price' => 29.00,
        'special_price' => null,
        'final_price' => 29.00,
        
        // Inventory
        'qty' => 25,
        'is_in_stock' => 1,
        'stock_status' => 1,
        'manage_stock' => 1,
        'min_sale_qty' => 1,
        'max_sale_qty' => 10,
        
        // Configurable Attributes
        'color' => 'Black',
        'size' => 'XS',
        'color_value' => 49, // value_index
        'size_value' => 166, // value_index
        
        // Physical
        'weight' => 0.5,
        
        // Media (variant-specific)
        'image' => '/w/s/ws03-black-xs.jpg',
        'small_image' => '/w/s/ws03-black-xs.jpg',
        'thumbnail' => '/w/s/ws03-black-xs.jpg'
    ]
];
```

---

## How It Works on the Frontend

### Customer Experience:
1. **Product Page**: Customer sees "Cassius Sparring Tank" 
2. **Dropdowns**: Two dropdown menus appear:
   - Color: Black, Blue, Red
   - Size: XS, S, M, L
3. **Selection**: Customer selects "Blue" and "M"
4. **Updates**: Page updates to show:
   - Price: $31.00 (variant-specific)
   - Stock: 15 available
   - Image: Blue tank top
   - SKU: WS03-BLUE-M

### Behind the Scenes:
1. JavaScript captures selections
2. AJAX call matches selections to variant
3. Product data updates dynamically
4. Add to cart uses child product (WS03-BLUE-M)

---

## Database Relationships

### Tables Involved:
- **`catalog_product_entity`** - Both parent and child products
- **`catalog_product_super_link`** - Links parent to children
- **`catalog_product_super_attribute`** - Configurable attributes
- **`eav_attribute`** - Attribute definitions
- **`eav_attribute_option`** - Attribute options
- **`cataloginventory_stock_item`** - Stock for each variant

### Key Relationships:
```
Configurable Product (ID: 50)
├── Super Link → Simple Product (ID: 51) [Black, XS]
├── Super Link → Simple Product (ID: 52) [Black, S]  
└── Super Link → Simple Product (ID: 53) [Blue, M]
```

---

## Important Notes

### Inventory Management:
- **Parent Product**: Never manages stock directly
- **Child Products**: Each variant has individual stock
- **Stock Status**: Parent shows "In Stock" if ANY child is in stock

### Pricing Rules:
- **Base Price**: Set on parent product
- **Variant Pricing**: Each child can have different price
- **Price Display**: Shows range if prices vary ("$29.00 - $35.00")

### SEO Considerations:
- **URLs**: Parent product has main URL
- **Child Products**: Usually not indexed individually
- **Meta Data**: Set on parent, inherited by children

This structure allows for powerful, flexible product management while maintaining a clean customer experience!
