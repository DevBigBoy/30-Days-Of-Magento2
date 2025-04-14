# 🚀 What is the EAV Model?

**EAV = Entity - Attribute - Value**

Magento uses the EAV model primarily for entities that need to be **extensible** and store **highly customizable attributes** — like **products**, **categories**, **customers**, etc.

Instead of storing all attributes in a single flat table (like a traditional relational model), EAV spreads them across multiple tables depending on their data types.

---

### 🔍 1. **Entity**

An **entity** is the item you're storing data for — e.g., a **product**, a **category**, or a **customer**.

Examples:

- `catalog_product_entity`
- `customer_entity`
- `catalog_category_entity`

These entity tables store the **core data** — like entity IDs, timestamps, etc.

---

### 🎛️ 2. **Attribute**

Attributes are the **individual data points** you want to store for each entity. Like:

- `name`
- `price`
- `description`
- `color`
- `size`
- `date_of_birth` (for customers)

All the metadata about attributes is stored in:

- `eav_attribute`
- `eav_entity_type` (defines the entity types like product, customer, etc.)
- `catalog_eav_attribute` (extra metadata for product attributes like visibility, filterable, etc.)

Each attribute is defined once and can be reused across many entities.

---

### 📦 3. **Value**

This is where the actual data lives, and it’s stored in **type-specific tables**:

- `*_entity_varchar`
- `*_entity_int`
- `*_entity_decimal`
- `*_entity_text`
- `*_entity_datetime`

So for a product, you might have:

- `catalog_product_entity_varchar` → for `name`, `meta_title`, etc.
- `catalog_product_entity_decimal` → for `price`, `weight`, etc.

Each of these value tables has:

- `value_id`
- `attribute_id`
- `store_id`
- `entity_id`
- `value`

The data is joined based on `attribute_id` and `entity_id`.

---

### 🧠 4. **Store Scope**

A key EAV feature in Magento: **store view level values**.

Each value table includes a `store_id` column, which lets you:

- Store different attribute values for different store views.
- e.g., a product name in English for Store ID 1 and in French for Store ID 2.

---

### 🏗️ 5. **Attribute Sets & Groups**

Attributes are grouped together into:

- **Attribute Sets** (like a "Product Type" — e.g., Shoes, TVs, Books)
- **Attribute Groups** (for admin UI display — like General, Prices, SEO)

Relevant tables:

- `eav_attribute_set`
- `eav_attribute_group`
- `eav_entity_attribute` (mapping attributes to sets & groups)

This helps Magento dynamically show only relevant fields on the admin form for different product types.

---

### 🛠️ 6. **Creating Custom Attributes**

As a dev, you can add custom attributes using data patches or setup scripts. For example:

```php
$eavSetup->addAttribute(
    \Magento\Catalog\Model\Product::ENTITY,
    'custom_attribute',
    [
        'type' => 'varchar',
        'label' => 'My Custom Attribute',
        'input' => 'text',
        'required' => false,
        'sort_order' => 100,
        'global' => \Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface::SCOPE_STORE,
        'visible' => true,
        'user_defined' => true,
        'group' => 'General',
    ]
);
```

---

### 🧪 7. **Pros & Cons of EAV**

**Pros:**

- Super flexible and extensible.
- Great for entities with many dynamic attributes.
- Easy to support multi-store and multi-language setups.

**Cons:**

- Performance is slower than flat table models (lots of joins).
- More complex to query and work with directly in SQL.
- Needs indexing to be performant.

---

### 🧰 8. **Indexers and Flat Tables**

Magento improves performance by generating **flat tables** for catalog entities:

- `catalog_product_flat_X` (where X is store ID)

These are generated via indexers to optimize frontend performance.

---

### 🔄 9. **How It All Works Together**

When you load a product like this:

```php
$product = $productRepository->getById(123);
```

Magento:

1. Knows the entity (`catalog_product_entity`)
2. Pulls all attribute metadata via `eav_attribute`
3. Joins the value tables (varchar, decimal, etc.) to build a full product object
4. Applies store-specific values if needed

---

Whew! 😅 That’s a lot, but this is the **heart of Magento’s flexibility**.