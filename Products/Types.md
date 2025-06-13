# Magento 2 Product Types - Complete Comprehensive Guide

## Table of Contents
1. [Product Types Overview](#product-types-overview)
2. [Simple Product](#simple-product)
3. [Configurable Product](#configurable-product)
4. [Grouped Product](#grouped-product)
5. [Bundle Product](#bundle-product)
6. [Virtual Product](#virtual-product)
7. [Downloadable Product](#downloadable-product)
8. [Product Type Architecture](#product-type-architecture)
9. [Custom Product Types](#custom-product-types)
10. [Product Type Behaviors](#product-type-behaviors)
11. [API Integration](#api-integration)
12. [Performance Considerations](#performance-considerations)
13. [Best Practices](#best-practices)

---

## Product Types Overview

### **What are Product Types?**
Product types in Magento 2 define **how products behave**, what **attributes they support**, how they're **displayed**, **priced**, and **purchased**. Each type serves different business needs.

### **Core Product Types in Magento 2**
```
1. Simple Product - Basic single product
2. Configurable Product - Product with variations (size, color, etc.)
3. Grouped Product - Collection of simple products
4. Bundle Product - Product with selectable options
5. Virtual Product - Non-physical product (services, warranties)
6. Downloadable Product - Digital products (ebooks, software)
```

### **Product Type Hierarchy**
```
Magento\Catalog\Model\Product\Type\AbstractType
    ├── Simple
    ├── Virtual (extends Simple)
    ├── Downloadable (extends Virtual)
    ├── Configurable
    ├── Grouped
    └── Bundle
```

---

## Simple Product

### **What is a Simple Product?**
The **most basic product type** - represents a single item with a fixed price and no variations.

### **Characteristics**
- **Single SKU**
- **Fixed price**
- **Direct inventory management**
- **No variations or options**
- **Can have custom options**

### **Real-World Examples**
- T-shirt (specific size and color)
- Book (specific edition)
- Phone case (specific model)
- Coffee mug
- Laptop charger

### **Technical Implementation**

#### **Product Type Class**
```php
<?php

namespace Magento\Catalog\Model\Product\Type;

class Simple extends AbstractType
{
    const TYPE_CODE = 'simple';
    
    /**
     * Check if product can be bought
     */
    public function isSalable($product)
    {
        $salable = parent::isSalable($product);
        
        if ($salable !== false) {
            $salable = $product->getStatus() == Status::STATUS_ENABLED;
        }
        
        return $salable;
    }
    
    /**
     * Prepare product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        $products = [];
        $product->setCartQty($buyRequest->getQty());
        $products[] = $product;
        
        return $products;
    }
}
```

#### **Creating Simple Product Programmatically**
```php
<?php

namespace Vendor\Module\Model;

use Magento\Catalog\Api\Data\ProductInterfaceFactory;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Catalog\Model\Product\Attribute\Source\Status;
use Magento\Catalog\Model\Product\Type;
use Magento\Catalog\Model\Product\Visibility;

class SimpleProductCreator
{
    private $productFactory;
    private $productRepository;
    
    public function __construct(
        ProductInterfaceFactory $productFactory,
        ProductRepositoryInterface $productRepository
    ) {
        $this->productFactory = $productFactory;
        $this->productRepository = $productRepository;
    }
    
    public function createSimpleProduct()
    {
        /** @var \Magento\Catalog\Model\Product $product */
        $product = $this->productFactory->create();
        
        $product->setTypeId(Type::TYPE_SIMPLE)
            ->setAttributeSetId(4) // Default attribute set
            ->setName('Sample T-Shirt')
            ->setSku('sample-tshirt-001')
            ->setUrlKey('sample-tshirt')
            ->setDescription('High quality cotton t-shirt')
            ->setShortDescription('Cotton t-shirt')
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setPrice(29.99)
            ->setWeight(0.5)
            ->setStockData([
                'use_config_manage_stock' => 1,
                'manage_stock' => 1,
                'is_in_stock' => 1,
                'qty' => 100
            ]);
        
        return $this->productRepository->save($product);
    }
}
```

#### **Simple Product with Custom Options**
```php
public function createSimpleProductWithOptions()
{
    $product = $this->createSimpleProduct();
    
    // Add custom options
    $options = [
        [
            'title' => 'Gift Wrapping',
            'type' => 'drop_down',
            'is_require' => 0,
            'sort_order' => 1,
            'values' => [
                [
                    'title' => 'No Gift Wrapping',
                    'price' => 0,
                    'price_type' => 'fixed',
                    'sort_order' => 1
                ],
                [
                    'title' => 'Gift Box (+$5)',
                    'price' => 5,
                    'price_type' => 'fixed',
                    'sort_order' => 2
                ]
            ]
        ]
    ];
    
    $product->setHasOptions(true);
    $product->setCanSaveCustomOptions(true);
    
    foreach ($options as $optionData) {
        $option = $this->optionFactory->create(['data' => $optionData]);
        $option->setProductSku($product->getSku());
        $product->addOption($option);
    }
    
    return $this->productRepository->save($product);
}
```

---

## Configurable Product

### **What is a Configurable Product?**
A **parent product** that groups simple products (variants) based on **configurable attributes** like size, color, material.

### **Characteristics**
- **Parent-child relationship**
- **Configurable attributes** (size, color, etc.)
- **Child products are simple products**
- **Dynamic pricing** based on selected variant
- **Shared attributes** from parent
- **Individual inventory** per variant

### **Real-World Examples**
- T-shirt available in multiple sizes and colors
- Shoes in different sizes and materials
- Laptop with different RAM/storage configurations
- Phone in different colors and storage capacities

### **Technical Implementation**

#### **Product Type Class**
```php
<?php

namespace Magento\ConfigurableProduct\Model\Product\Type;

class Configurable extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    const TYPE_CODE = 'configurable';
    
    /**
     * Get used products (child simple products)
     */
    public function getUsedProducts($product, $requiredAttributeIds = null)
    {
        if (!$product->hasData($this->_usedProducts)) {
            $collection = $this->getUsedProductCollection($product);
            $usedProducts = array_values($collection->getItems());
            $product->setData($this->_usedProducts, $usedProducts);
        }
        
        return $product->getData($this->_usedProducts);
    }
    
    /**
     * Prepare product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        $attributes = $buyRequest->getSuperAttribute();
        $product = $this->getProductByAttributes($attributes, $product);
        
        if (is_string($product)) {
            return $product; // Error message
        }
        
        return parent::prepareForCartAdvanced($buyRequest, $product, $processMode);
    }
}
```

#### **Creating Configurable Product**
```php
<?php

namespace Vendor\Module\Model;

class ConfigurableProductCreator
{
    private $productFactory;
    private $productRepository;
    private $configurableProcessor;
    private $attributeRepository;
    
    public function createConfigurableProduct()
    {
        // 1. Create parent configurable product
        $configurableProduct = $this->productFactory->create();
        $configurableProduct->setTypeId(\Magento\ConfigurableProduct\Model\Product\Type\Configurable::TYPE_CODE)
            ->setAttributeSetId(4)
            ->setName('Configurable T-Shirt')
            ->setSku('configurable-tshirt')
            ->setUrlKey('configurable-tshirt')
            ->setDescription('Available in multiple sizes and colors')
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setPrice(0) // Will be determined by variants
            ->setStockData([
                'use_config_manage_stock' => 1,
                'is_in_stock' => 1
            ]);
        
        $configurableProduct = $this->productRepository->save($configurableProduct);
        
        // 2. Create simple product variants
        $variants = $this->createSimpleVariants();
        
        // 3. Set up configurable attributes
        $configurableAttributesData = [
            [
                'attribute_id' => $this->getAttributeId('size'),
                'code' => 'size',
                'label' => 'Size',
                'position' => 0,
                'values' => $this->getSizeOptions()
            ],
            [
                'attribute_id' => $this->getAttributeId('color'),
                'code' => 'color', 
                'label' => 'Color',
                'position' => 1,
                'values' => $this->getColorOptions()
            ]
        ];
        
        // 4. Link variants to configurable product
        $this->configurableProcessor->generateSimpleProducts($configurableProduct, $configurableAttributesData);
        
        return $configurableProduct;
    }
    
    private function createSimpleVariants()
    {
        $variants = [];
        $sizes = ['S', 'M', 'L', 'XL'];
        $colors = ['Red', 'Blue', 'Green', 'Black'];
        
        foreach ($sizes as $size) {
            foreach ($colors as $color) {
                $variant = $this->productFactory->create();
                $variant->setTypeId(Type::TYPE_SIMPLE)
                    ->setAttributeSetId(4)
                    ->setName("T-Shirt {$size} {$color}")
                    ->setSku("tshirt-{$size}-{$color}")
                    ->setPrice(29.99)
                    ->setSize($this->getSizeOptionId($size))
                    ->setColor($this->getColorOptionId($color))
                    ->setStatus(Status::STATUS_ENABLED)
                    ->setVisibility(Visibility::VISIBILITY_NOT_VISIBLE)
                    ->setStockData([
                        'use_config_manage_stock' => 1,
                        'manage_stock' => 1,
                        'is_in_stock' => 1,
                        'qty' => 50
                    ]);
                
                $variants[] = $this->productRepository->save($variant);
            }
        }
        
        return $variants;
    }
}
```

#### **Frontend Configuration Display**
```php
<?php

namespace Vendor\Module\Block\Product\View\Type;

class Configurable extends \Magento\ConfigurableProduct\Block\Product\View\Type\Configurable
{
    /**
     * Get JSON configuration for frontend
     */
    public function getJsonConfig()
    {
        $config = [];
        $product = $this->getProduct();
        
        // Get configurable attributes
        $configurableAttributes = $this->helper->getAllowAttributes($product);
        
        foreach ($configurableAttributes as $attribute) {
            $attributeId = $attribute->getAttributeId();
            $config['attributes'][$attributeId] = [
                'id' => $attributeId,
                'code' => $attribute->getAttributeCode(),
                'label' => $attribute->getStoreLabel(),
                'options' => $this->getAttributeOptions($attribute, $product)
            ];
        }
        
        // Get used products data
        $config['choicesData'] = $this->getUsedProductsData($product);
        
        return json_encode($config);
    }
    
    private function getUsedProductsData($product)
    {
        $data = [];
        $usedProducts = $this->helper->getAllowProducts($product);
        
        foreach ($usedProducts as $usedProduct) {
            $data[$usedProduct->getId()] = [
                'price' => $this->getProductPrice($usedProduct),
                'oldPrice' => $this->getProductOldPrice($usedProduct),
                'tierPrices' => $this->getTierPrices($usedProduct),
                'images' => $this->getProductImages($usedProduct)
            ];
        }
        
        return $data;
    }
}
```

---

## Grouped Product

### **What is a Grouped Product?**
A **collection of simple products** displayed together, allowing customers to **purchase multiple items** in one transaction.

### **Characteristics**
- **Groups multiple simple products**
- **Individual quantities** for each item
- **Separate pricing** for each product
- **Optional purchases** - can buy any combination
- **Shared product page**

### **Real-World Examples**
- Dining room set (table + 4 chairs + buffet)
- Computer bundle (monitor + keyboard + mouse)
- Skincare routine (cleanser + toner + moisturizer)
- Tool set (drill + bits + case)

### **Technical Implementation**

#### **Product Type Class**
```php
<?php

namespace Magento\GroupedProduct\Model\Product\Type;

class Grouped extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    const TYPE_CODE = 'grouped';
    
    /**
     * Get associated products
     */
    public function getAssociatedProducts($product)
    {
        if (!$product->hasData($this->_keyAssociatedProducts)) {
            $associatedProducts = [];
            
            if (!Mage::app()->getStore()->isAdmin()) {
                $this->setSaleableStatus($product);
            }
            
            $collection = $this->getAssociatedProductCollection($product)
                ->addAttributeToSelect('*')
                ->addFilterByRequiredOptions()
                ->setPositionOrder()
                ->addStoreFilter($this->getStoreFilter($product))
                ->addAttributeToFilter('status', Status::STATUS_ENABLED);
            
            foreach ($collection as $item) {
                $associatedProducts[] = $item;
            }
            
            $product->setData($this->_keyAssociatedProducts, $associatedProducts);
        }
        
        return $product->getData($this->_keyAssociatedProducts);
    }
    
    /**
     * Prepare grouped product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        $products = [];
        $associatedProductsInfo = $buyRequest->getSuperGroup();
        
        if (is_array($associatedProductsInfo)) {
            $associatedProducts = $this->getAssociatedProducts($product);
            
            foreach ($associatedProducts as $associatedProduct) {
                $qty = $associatedProductsInfo[$associatedProduct->getId()] ?? 0;
                
                if ($qty > 0) {
                    $associatedProduct->setCartQty($qty);
                    $products[] = $associatedProduct;
                }
            }
        }
        
        return $products;
    }
}
```

#### **Creating Grouped Product**
```php
<?php

namespace Vendor\Module\Model;

class GroupedProductCreator
{
    private $productFactory;
    private $productRepository;
    private $linkManagement;
    
    public function createGroupedProduct()
    {
        // 1. Create grouped product
        $groupedProduct = $this->productFactory->create();
        $groupedProduct->setTypeId(\Magento\GroupedProduct\Model\Product\Type\Grouped::TYPE_CODE)
            ->setAttributeSetId(4)
            ->setName('Dining Room Set')
            ->setSku('dining-room-set')
            ->setUrlKey('dining-room-set')
            ->setDescription('Complete dining room furniture set')
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setStockData([
                'use_config_manage_stock' => 1,
                'is_in_stock' => 1
            ]);
        
        $groupedProduct = $this->productRepository->save($groupedProduct);
        
        // 2. Create associated simple products
        $associatedProducts = $this->createAssociatedProducts();
        
        // 3. Link products to grouped product
        $this->linkProductsToGrouped($groupedProduct, $associatedProducts);
        
        return $groupedProduct;
    }
    
    private function createAssociatedProducts()
    {
        $products = [];
        
        $productData = [
            ['name' => 'Dining Table', 'sku' => 'dining-table', 'price' => 599.99],
            ['name' => 'Dining Chair', 'sku' => 'dining-chair', 'price' => 149.99],
            ['name' => 'Buffet Cabinet', 'sku' => 'buffet-cabinet', 'price' => 399.99]
        ];
        
        foreach ($productData as $data) {
            $product = $this->productFactory->create();
            $product->setTypeId(Type::TYPE_SIMPLE)
                ->setAttributeSetId(4)
                ->setName($data['name'])
                ->setSku($data['sku'])
                ->setPrice($data['price'])
                ->setStatus(Status::STATUS_ENABLED)
                ->setVisibility(Visibility::VISIBILITY_NOT_VISIBLE)
                ->setStockData([
                    'use_config_manage_stock' => 1,
                    'manage_stock' => 1,
                    'is_in_stock' => 1,
                    'qty' => 10
                ]);
            
            $products[] = $this->productRepository->save($product);
        }
        
        return $products;
    }
    
    private function linkProductsToGrouped($groupedProduct, $associatedProducts)
    {
        $position = 0;
        
        foreach ($associatedProducts as $product) {
            $linkData = [
                'sku' => $groupedProduct->getSku(),
                'link_type' => 'associated',
                'linked_product_sku' => $product->getSku(),
                'linked_product_type' => $product->getTypeId(),
                'position' => $position++,
                'qty' => 1 // Default quantity
            ];
            
            $this->linkManagement->setProductLinks($groupedProduct->getSku(), [$linkData]);
        }
    }
}
```

#### **Frontend Grouped Product Display**
```php
<?php

namespace Vendor\Module\Block\Product\View\Type;

class Grouped extends \Magento\GroupedProduct\Block\Product\View\Type\Grouped
{
    /**
     * Get associated products with additional data
     */
    public function getAssociatedProducts()
    {
        $product = $this->getProduct();
        $associatedProducts = [];
        
        foreach ($product->getTypeInstance()->getAssociatedProducts($product) as $associatedProduct) {
            $associatedProducts[] = [
                'product' => $associatedProduct,
                'price' => $this->getProductPrice($associatedProduct),
                'qty' => $this->getProductDefaultQty($associatedProduct),
                'can_change_qty' => true,
                'position' => $associatedProduct->getPosition()
            ];
        }
        
        // Sort by position
        usort($associatedProducts, function($a, $b) {
            return $a['position'] <=> $b['position'];
        });
        
        return $associatedProducts;
    }
    
    /**
     * Get total price for grouped product
     */
    public function getTotalPrice()
    {
        $total = 0;
        
        foreach ($this->getAssociatedProducts() as $item) {
            $total += $item['price'] * $item['qty'];
        }
        
        return $total;
    }
}
```

---

## Bundle Product

### **What is a Bundle Product?**
A **composite product** where customers can **choose from multiple options** for each bundle component, creating their own product combination.

### **Characteristics**
- **Multiple bundle options** (components)
- **Selectable items** for each option
- **Dynamic pricing** based on selections
- **Required or optional** components
- **Quantity selection** per item
- **Complex pricing rules**

### **Real-World Examples**
- Computer builder (choose CPU, RAM, storage, graphics card)
- Gift basket (choose items from different categories)
- Phone plan (choose device + plan + accessories)
- Custom pizza (choose size, crust, toppings)

### **Technical Implementation**

#### **Product Type Class**
```php
<?php

namespace Magento\Bundle\Model\Product\Type;

class Bundle extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    const TYPE_CODE = 'bundle';
    
    /**
     * Get bundle options
     */
    public function getOptions($product)
    {
        if (!$product->hasData('_cache_instance_options_collection')) {
            $optionsCollection = $this->getOptionsCollection($product);
            $selectionsCollection = $this->getSelectionsCollection($optionsCollection->getAllIds(), $product);
            
            $options = $optionsCollection->appendSelections($selectionsCollection);
            $product->setData('_cache_instance_options_collection', $options);
        }
        
        return $product->getData('_cache_instance_options_collection');
    }
    
    /**
     * Prepare bundle product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        $options = $buyRequest->getBundleOption();
        $optionQty = $buyRequest->getBundleOptionQty();
        
        if (!$options) {
            return __('Please specify product option(s).')->render();
        }
        
        $products = [$product];
        $result = $this->processBundleOptions($product, $options, $optionQty);
        
        if (is_string($result)) {
            return $result; // Error message
        }
        
        $products = array_merge($products, $result);
        
        return $products;
    }
}
```

#### **Creating Bundle Product**
```php
<?php

namespace Vendor\Module\Model;

class BundleProductCreator
{
    private $productFactory;
    private $productRepository;
    private $bundleOptionFactory;
    private $bundleSelectionFactory;
    
    public function createBundleProduct()
    {
        // 1. Create bundle product
        $bundleProduct = $this->productFactory->create();
        $bundleProduct->setTypeId(\Magento\Bundle\Model\Product\Type::TYPE_CODE)
            ->setAttributeSetId(4)
            ->setName('Custom Computer Bundle')
            ->setSku('custom-computer-bundle')
            ->setUrlKey('custom-computer-bundle')
            ->setDescription('Build your custom computer')
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setPriceType(\Magento\Bundle\Model\Product\Price::PRICE_TYPE_DYNAMIC) // Dynamic pricing
            ->setPrice(0)
            ->setSpecialPrice(null)
            ->setPriceView(1) // Price range
            ->setShipmentType(0) // Ship together
            ->setBundleOptionsData($this->getBundleOptionsData())
            ->setCanSaveBundleSelections(true)
            ->setStockData([
                'use_config_manage_stock' => 1,
                'is_in_stock' => 1
            ]);
        
        $bundleProduct = $this->productRepository->save($bundleProduct);
        
        // 2. Create bundle options and selections
        $this->createBundleOptions($bundleProduct);
        
        return $bundleProduct;
    }
    
    private function getBundleOptionsData()
    {
        return [
            // Option 1: Processor
            [
                'title' => 'Processor',
                'type' => 'select', // dropdown
                'required' => 1,
                'position' => 1,
                'selections' => [
                    ['sku' => 'intel-i5-cpu', 'qty' => 1, 'is_default' => 1],
                    ['sku' => 'intel-i7-cpu', 'qty' => 1, 'is_default' => 0],
                    ['sku' => 'amd-ryzen-cpu', 'qty' => 1, 'is_default' => 0]
                ]
            ],
            // Option 2: Memory
            [
                'title' => 'Memory (RAM)',
                'type' => 'radio',
                'required' => 1,
                'position' => 2,
                'selections' => [
                    ['sku' => 'ram-8gb', 'qty' => 1, 'is_default' => 1],
                    ['sku' => 'ram-16gb', 'qty' => 1, 'is_default' => 0],
                    ['sku' => 'ram-32gb', 'qty' => 1, 'is_default' => 0]
                ]
            ],
            // Option 3: Storage
            [
                'title' => 'Storage',
                'type' => 'checkbox',
                'required' => 1,
                'position' => 3,
                'selections' => [
                    ['sku' => 'ssd-256gb', 'qty' => 1, 'is_default' => 1],
                    ['sku' => 'ssd-512gb', 'qty' => 1, 'is_default' => 0],
                    ['sku' => 'hdd-1tb', 'qty' => 1, 'is_default' => 0]
                ]
            ],
            // Option 4: Graphics Card (Optional)
            [
                'title' => 'Graphics Card',
                'type' => 'select',
                'required' => 0,
                'position' => 4,
                'selections' => [
                    ['sku' => 'gpu-basic', 'qty' => 1, 'is_default' => 1],
                    ['sku' => 'gpu-gaming', 'qty' => 1, 'is_default' => 0],
                    ['sku' => 'gpu-professional', 'qty' => 1, 'is_default' => 0]
                ]
            ]
        ];
    }
    
    private function createBundleOptions($bundleProduct)
    {
        $optionsData = $this->getBundleOptionsData();
        
        foreach ($optionsData as $optionData) {
            // Create bundle option
            $option = $this->bundleOptionFactory->create();
            $option->setData([
                'parent_id' => $bundleProduct->getId(),
                'title' => $optionData['title'],
                'type' => $optionData['type'],
                'required' => $optionData['required'],
                'position' => $optionData['position']
            ]);
            $option->save();
            
            // Create selections for this option
            foreach ($optionData['selections'] as $selectionData) {
                $selection = $this->bundleSelectionFactory->create();
                $selection->setData([
                    'option_id' => $option->getId(),
                    'product_id' => $this->getProductIdBySku($selectionData['sku']),
                    'selection_qty' => $selectionData['qty'],
                    'is_default' => $selectionData['is_default'],
                    'position' => 0
                ]);
                $selection->save();
            }
        }
    }
}
```

#### **Bundle Pricing Calculation**
```php
<?php

namespace Vendor\Module\Pricing\Price;

class BundlePriceCalculator
{
    /**
     * Calculate bundle price based on selections
     */
    public function calculateBundlePrice($bundleProduct, $selections)
    {
        $price = 0;
        
        if ($bundleProduct->getPriceType() == \Magento\Bundle\Model\Product\Price::PRICE_TYPE_FIXED) {
            // Fixed price bundle
            $price = $bundleProduct->getPrice();
        } else {
            // Dynamic price bundle - sum of selected products
            foreach ($selections as $optionId => $selectionIds) {
                if (!is_array($selectionIds)) {
                    $selectionIds = [$selectionIds];
                }
                
                foreach ($selectionIds as $selectionId) {
                    $selection = $this->getSelectionById($selectionId);
                    if ($selection) {
                        $selectionProduct = $this->getSelectionProduct($selection);
                        $selectionPrice = $this->getSelectionPrice($selection, $selectionProduct);
                        $price += $selectionPrice * $selection->getSelectionQty();
                    }
                }
            }
        }
        
        return $price;
    }
    
    private function getSelectionPrice($selection, $selectionProduct)
    {
        if ($selection->getSelectionPriceType() == 0) {
            // Fixed price
            return $selection->getSelectionPriceValue();
        } else {
            // Percentage of product price
            return ($selectionProduct->getPrice() * $selection->getSelectionPriceValue()) / 100;
        }
    }
}
```

---

## Virtual Product

### **What is a Virtual Product?**
A **non-physical product** that doesn't require shipping - typically services, digital goods, or warranties.

### **Characteristics**
- **No physical presence**
- **No weight or dimensions**
- **No shipping required**
- **Inherits from Simple product**
- **Can have custom options**

### **Real-World Examples**
- Software licenses
- Extended warranties
- Consulting services
- Online courses
- Gift cards
- Subscriptions

### **Technical Implementation**

#### **Product Type Class**
```php
<?php

namespace Magento\Catalog\Model\Product\Type;

class Virtual extends Simple
{
    const TYPE_CODE = 'virtual';
    
    /**
     * Check if product is virtual (always true)
     */
    public function isVirtual($product = null)
    {
        return true;
    }
    
    /**
     * Virtual products don't have weight
     */
    public function hasWeight()
    {
        return false;
    }
    
    /**
     * Prepare virtual product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        // Same as simple product but without shipping considerations
        return parent::prepareForCartAdvanced($buyRequest, $product, $processMode);
    }
}
```

#### **Creating Virtual Product**
```php
<?php

namespace Vendor\Module\Model;

class VirtualProductCreator
{
    private $productFactory;
    private $productRepository;
    
    public function createVirtualProduct()
    {
        $virtualProduct = $this->productFactory->create();
        $virtualProduct->setTypeId(\Magento\Catalog\Model\Product\Type\Virtual::TYPE_CODE)
            ->setAttributeSetId(4)
            ->setName('Extended Warranty - 2 Years')
            ->setSku('warranty-2year')
            ->setUrlKey('extended-warranty-2year')
            ->setDescription('2-year extended warranty coverage')
            ->setShortDescription('Extended warranty service')
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setPrice(99.99)
            ->setWeight(0) // Virtual products have no weight
            ->setStockData([
                'use_config_manage_stock' => 0, // Don't manage stock for services
                'manage_stock' => 0,
                'is_in_stock' => 1,
                'qty' => 0
            ]);
        
        return $this->productRepository->save($virtualProduct);
    }
    
    public function createConsultingService()
    {
        $consultingProduct = $this->productFactory->create();
        $consultingProduct->setTypeId(\Magento\Catalog\Model\Product\Type\Virtual::TYPE_CODE)
            ->setAttributeSetId(4)
            ->setName('Business Consulting Session')
            ->setSku('consulting-session')
            ->setDescription('1-hour business consulting session with our expert')
            ->setPrice(150.00)
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setStockData([
                'use_config_manage_stock' => 0,
                'manage_stock' => 0,
                'is_in_stock' => 1
            ]);
        
        // Add custom options for scheduling
        $this->addSchedulingOptions($consultingProduct);
        
        return $this->productRepository->save($consultingProduct);
    }
    
    private function addSchedulingOptions($product)
    {
        $options = [
            [
                'title' => 'Preferred Date',
                'type' => 'date',
                'is_require' => 1,
                'sort_order' => 1
            ],
            [
                'title' => 'Time Slot',
                'type' => 'drop_down',
                'is_require' => 1,
                'sort_order' => 2,
                'values' => [
                    ['title' => '9:00 AM - 10:00 AM', 'sort_order' => 1],
                    ['title' => '10:00 AM - 11:00 AM', 'sort_order' => 2],
                    ['title' => '2:00 PM - 3:00 PM', 'sort_order' => 3],
                    ['title' => '3:00 PM - 4:00 PM', 'sort_order' => 4]
                ]
            ]
        ];
        
        $product->setHasOptions(true);
        $product->setCanSaveCustomOptions(true);
        
        foreach ($options as $optionData) {
            $option = $this->optionFactory->create(['data' => $optionData]);
            $product->addOption($option);
        }
    }
}
```

---

## Downloadable Product

### **What is a Downloadable Product?**
A **digital product** that customers can download after purchase - extends Virtual product with download functionality.

### **Characteristics**
- **Extends Virtual product**
- **Downloadable files/links**
- **Download limits** (times/days)
- **Link expiration**
- **Sample downloads**
- **No shipping required**

### **Real-World Examples**
- E-books
- Software downloads
- Music files
- Video courses
- Digital templates
- PDF documents

### **Technical Implementation**

#### **Product Type Class**
```php
<?php

namespace Magento\Downloadable\Model\Product\Type;

class Downloadable extends \Magento\Catalog\Model\Product\Type\Virtual
{
    const TYPE_CODE = 'downloadable';
    
    /**
     * Check if product has downloadable links
     */
    public function hasLinks($product)
    {
        return $product->getTypeInstance()->getLinks($product) ? true : false;
    }
    
    /**
     * Get downloadable links
     */
    public function getLinks($product)
    {
        if (!$product->hasData('_cache_instance_links')) {
            $links = $this->getLinksCollection($product);
            $product->setData('_cache_instance_links', $links);
        }
        
        return $product->getData('_cache_instance_links');
    }
    
    /**
     * Prepare downloadable product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        $result = parent::prepareForCartAdvanced($buyRequest, $product, $processMode);
        
        if (is_string($result)) {
            return $result; // Error message
        }
        
        // Validate downloadable links selection
        $links = $buyRequest->getLinks();
        if ($this->hasRequiredLinks($product) && !$links) {
            return __('Please specify downloadable link(s).')->render();
        }
        
        return $result;
    }
}
```

#### **Creating Downloadable Product**
```php
<?php

namespace Vendor\Module\Model;

class DownloadableProductCreator
{
    private $productFactory;
    private $productRepository;
    private $linkFactory;
    private $sampleFactory;
    
    public function createDownloadableProduct()
    {
        // 1. Create downloadable product
        $downloadableProduct = $this->productFactory->create();
        $downloadableProduct->setTypeId(\Magento\Downloadable\Model\Product\Type::TYPE_CODE)
            ->setAttributeSetId(4)
            ->setName('Programming E-Book Collection')
            ->setSku('ebook-programming-collection')
            ->setUrlKey('programming-ebook-collection')
            ->setDescription('Complete collection of programming e-books')
            ->setStatus(Status::STATUS_ENABLED)
            ->setVisibility(Visibility::VISIBILITY_BOTH)
            ->setPrice(49.99)
            ->setStockData([
                'use_config_manage_stock' => 0,
                'manage_stock' => 0,
                'is_in_stock' => 1
            ]);
        
        $downloadableProduct = $this->productRepository->save($downloadableProduct);
        
        // 2. Add downloadable links
        $this->addDownloadableLinks($downloadableProduct);
        
        // 3. Add samples
        $this->addSamples($downloadableProduct);
        
        return $downloadableProduct;
    }
    
    private function addDownloadableLinks($product)
    {
        $linksData = [
            [
                'title' => 'PHP Fundamentals E-Book',
                'price' => 0, // Included in product price
                'number_of_downloads' => 5, // 5 downloads allowed
                'is_shareable' => 0, // Not shareable
                'link_url' => 'https://example.com/downloads/php-fundamentals.pdf',
                'link_type' => 'url',
                'sample_url' => 'https://example.com/samples/php-fundamentals-sample.pdf',
                'sample_type' => 'url',
                'sort_order' => 1
            ],
            [
                'title' => 'JavaScript Advanced Techniques',
                'price' => 10, // Additional price
                'number_of_downloads' => 3,
                'is_shareable' => 0,
                'link_url' => 'https://example.com/downloads/js-advanced.pdf',
                'link_type' => 'url',
                'sample_url' => 'https://example.com/samples/js-advanced-sample.pdf',
                'sample_type' => 'url',
                'sort_order' => 2
            ],
            [
                'title' => 'Database Design Guide',
                'price' => 5,
                'number_of_downloads' => 0, // Unlimited downloads
                'is_shareable' => 1, // Shareable
                'link_file' => '/path/to/database-design.pdf', // File upload
                'link_type' => 'file',
                'sample_file' => '/path/to/database-design-sample.pdf',
                'sample_type' => 'file',
                'sort_order' => 3
            ]
        ];
        
        foreach ($linksData as $linkData) {
            $link = $this->linkFactory->create();
            $link->setData($linkData);
            $link->setProductId($product->getId());
            $link->save();
        }
    }
    
    private function addSamples($product)
    {
        $samplesData = [
            [
                'title' => 'Programming Best Practices - Preview',
                'sample_url' => 'https://example.com/samples/best-practices-preview.pdf',
                'sample_type' => 'url',
                'sort_order' => 1
            ],
            [
                'title' => 'Code Examples - Preview',
                'sample_file' => '/path/to/code-examples-preview.zip',
                'sample_type' => 'file',
                'sort_order' => 2
            ]
        ];
        
        foreach ($samplesData as $sampleData) {
            $sample = $this->sampleFactory->create();
            $sample->setData($sampleData);
            $sample->setProductId($product->getId());
            $sample->save();
        }
    }
}
```

#### **Download Management**
```php
<?php

namespace Vendor\Module\Model\Downloadable;

class LinkManagement
{
    private $linkPurchasedFactory;
    private $customerSession;
    
    /**
     * Process downloadable links after order completion
     */
    public function processOrderDownloads($order)
    {
        foreach ($order->getAllItems() as $item) {
            if ($item->getProductType() !== \Magento\Downloadable\Model\Product\Type::TYPE_CODE) {
                continue;
            }
            
            $product = $item->getProduct();
            $links = $product->getTypeInstance()->getLinks($product);
            
            foreach ($links as $link) {
                $this->createPurchasedLink($order, $item, $link);
            }
        }
    }
    
    private function createPurchasedLink($order, $orderItem, $link)
    {
        $purchasedLink = $this->linkPurchasedFactory->create();
        $purchasedLink->setData([
            'order_id' => $order->getId(),
            'order_item_id' => $orderItem->getId(),
            'customer_id' => $order->getCustomerId(),
            'product_id' => $orderItem->getProductId(),
            'link_id' => $link->getId(),
            'link_title' => $link->getTitle(),
            'link_url' => $link->getLinkUrl(),
            'link_file' => $link->getLinkFile(),
            'link_type' => $link->getLinkType(),
            'number_of_downloads_bought' => $link->getNumberOfDownloads(),
            'number_of_downloads_used' => 0,
            'status' => 'available',
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s')
        ]);
        
        $purchasedLink->save();
        
        return $purchasedLink;
    }
    
    /**
     * Handle download request
     */
    public function processDownload($linkId, $customerId)
    {
        $purchasedLink = $this->getPurchasedLink($linkId, $customerId);
        
        if (!$purchasedLink || !$this->canDownload($purchasedLink)) {
            throw new \Exception('Download not available');
        }
        
        // Increment download count
        $purchasedLink->setNumberOfDownloadsUsed(
            $purchasedLink->getNumberOfDownloadsUsed() + 1
        );
        
        // Check if downloads exceeded
        if ($this->hasExceededDownloads($purchasedLink)) {
            $purchasedLink->setStatus('expired');
        }
        
        $purchasedLink->save();
        
        return $this->getDownloadContent($purchasedLink);
    }
}
```

---

## Product Type Architecture

### **Core Architecture Overview**
```php
namespace Magento\Catalog\Model\Product\Type;

abstract class AbstractType
{
    // Core methods all product types must implement
    abstract public function prepareForCartAdvanced();
    abstract public function isSalable($product);
    
    // Common functionality
    public function getProduct();
    public function isVirtual();
    public function hasWeight();
    public function getChildrenIds();
}

// Type Registry
class Pool
{
    private $types = [
        'simple' => 'Magento\Catalog\Model\Product\Type\Simple',
        'virtual' => 'Magento\Catalog\Model\Product\Type\Virtual',
        'configurable' => 'Magento\ConfigurableProduct\Model\Product\Type\Configurable',
        'grouped' => 'Magento\GroupedProduct\Model\Product\Type\Grouped',
        'bundle' => 'Magento\Bundle\Model\Product\Type',
        'downloadable' => 'Magento\Downloadable\Model\Product\Type'
    ];
}
```

### **Product Type Factory**
```php
<?php

namespace Magento\Catalog\Model\Product;

class TypeFactory
{
    private $types;
    private $objectManager;
    
    public function create($typeInstance)
    {
        if (isset($this->types[$typeInstance])) {
            return $this->objectManager->create($this->types[$typeInstance]);
        }
        
        throw new \InvalidArgumentException('Unknown product type: ' . $typeInstance);
    }
}
```

### **Type Configuration (product_types.xml)**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Catalog:etc/product_types.xsd">
    
    <type name="simple" label="Simple Product" modelInstance="Magento\Catalog\Model\Product\Type\Simple" 
          indexPriority="10" sortOrder="10" isQty="true">
        <priceModel instance="Magento\Catalog\Model\Product\Type\Price" />
        <indexerModel instance="Magento\Catalog\Model\Indexer\Product\Price\Simple" />
        <stockIndexerModel instance="Magento\CatalogInventory\Model\Indexer\Stock\Simple" />
    </type>
    
    <type name="virtual" label="Virtual Product" modelInstance="Magento\Catalog\Model\Product\Type\Virtual" 
          indexPriority="10" sortOrder="20" isQty="true">
        <priceModel instance="Magento\Catalog\Model\Product\Type\Price" />
    </type>
    
    <type name="configurable" label="Configurable Product" 
          modelInstance="Magento\ConfigurableProduct\Model\Product\Type\Configurable" 
          indexPriority="30" sortOrder="30" isQty="true" canUseQtyDecimals="false">
        <priceModel instance="Magento\ConfigurableProduct\Model\Product\Type\Configurable\Price" />
        <indexerModel instance="Magento\ConfigurableProduct\Model\Indexer\Product\Price\Configurable" />
        <stockIndexerModel instance="Magento\ConfigurableProduct\Model\Indexer\Stock\Configurable" />
    </type>
    
</config>
```

---

## Custom Product Types

### **Creating a Custom Product Type**

#### **1. Create Product Type Class**
```php
<?php

namespace Vendor\Module\Model\Product\Type;

class Custom extends \Magento\Catalog\Model\Product\Type\AbstractType
{
    const TYPE_CODE = 'custom';
    
    /**
     * Custom product is salable if in stock and enabled
     */
    public function isSalable($product)
    {
        $salable = parent::isSalable($product);
        
        if ($salable !== false) {
            // Add custom salability logic
            $salable = $this->isCustomConditionMet($product);
        }
        
        return $salable;
    }
    
    /**
     * Prepare custom product for cart
     */
    public function prepareForCartAdvanced(\Magento\Framework\DataObject $buyRequest, $product, $processMode = null)
    {
        // Custom validation logic
        $customOptions = $buyRequest->getCustomOptions();
        
        if (!$this->validateCustomOptions($customOptions)) {
            return __('Invalid custom options selected');
        }
        
        // Process as simple product
        $product->setCartQty($buyRequest->getQty());
        return [$product];
    }
    
    /**
     * Custom product specific logic
     */
    public function hasWeight()
    {
        return true; // Custom products have weight
    }
    
    public function isVirtual($product = null)
    {
        return false; // Custom products are physical
    }
    
    private function isCustomConditionMet($product)
    {
        // Implement custom business logic
        return $product->getCustomCondition() === 'met';
    }
    
    private function validateCustomOptions($customOptions)
    {
        // Implement custom options validation
        return !empty($customOptions['required_field']);
    }
}
```

#### **2. Register Product Type (product_types.xml)**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Catalog:etc/product_types.xsd">
    
    <type name="custom" 
          label="Custom Product" 
          modelInstance="Vendor\Module\Model\Product\Type\Custom" 
          indexPriority="50" 
          sortOrder="50" 
          isQty="true">
        <priceModel instance="Vendor\Module\Model\Product\Type\Custom\Price" />
        <indexerModel instance="Vendor\Module\Model\Indexer\Product\Price\Custom" />
        <stockIndexerModel instance="Vendor\Module\Model\Indexer\Stock\Custom" />
        <allowedSelectionTypes>
            <type name="simple" />
            <type name="virtual" />
        </allowedSelectionTypes>
    </type>
    
</config>
```

#### **3. Create Custom Price Model**
```php
<?php

namespace Vendor\Module\Model\Product\Type\Custom;

class Price extends \Magento\Catalog\Model\Product\Type\Price
{
    /**
     * Get custom product price
     */
    public function getPrice($product)
    {
        $price = parent::getPrice($product);
        
        // Add custom pricing logic
        $customPriceModifier = $product->getCustomPriceModifier();
        if ($customPriceModifier) {
            $price = $this->applyCustomPricing($price, $customPriceModifier);
        }
        
        return $price;
    }
    
    /**
     * Get final price with custom calculations
     */
    public function getFinalPrice($qty, $product)
    {
        $finalPrice = parent::getFinalPrice($qty, $product);
        
        // Apply quantity-based custom pricing
        if ($qty >= 10) {
            $finalPrice *= 0.9; // 10% bulk discount
        }
        
        return $finalPrice;
    }
    
    private function applyCustomPricing($basePrice, $modifier)
    {
        switch ($modifier['type']) {
            case 'percentage':
                return $basePrice * (1 + $modifier['value'] / 100);
            case 'fixed':
                return $basePrice + $modifier['value'];
            default:
                return $basePrice;
        }
    }
}
```

#### **4. Custom Product Form (Admin)**
```php
<?php

namespace Vendor\Module\Block\Adminhtml\Product\Edit\Tab;

class Custom extends \Magento\Backend\Block\Widget implements \Magento\Backend\Block\Widget\Tab\TabInterface
{
    /**
     * Get tab label
     */
    public function getTabLabel()
    {
        return __('Custom Options');
    }
    
    /**
     * Get tab title
     */
    public function getTabTitle()
    {
        return __('Custom Product Options');
    }
    
    /**
     * Check if tab can be shown
     */
    public function canShowTab()
    {
        return $this->getProduct()->getTypeId() === 'custom';
    }
    
    /**
     * Tab is hidden
     */
    public function isHidden()
    {
        return false;
    }
    
    /**
     * Get template
     */
    protected function _construct()
    {
        parent::_construct();
        $this->setTemplate('Vendor_Module::product/edit/custom.phtml');
    }
    
    /**
     * Get current product
     */
    public function getProduct()
    {
        return $this->_coreRegistry->registry('current_product');
    }
}
```

---

## Product Type Behaviors

### **Inventory Management by Type**

```php
class ProductTypeInventoryBehavior
{
    public function getInventoryBehaviors()
    {
        return [
            'simple' => [
                'manages_stock' => true,
                'has_qty' => true,
                'stock_item' => 'single',
                'reservation' => 'direct'
            ],
            'configurable' => [
                'manages_stock' => false, // Children manage stock
                'has_qty' => false,
                'stock_item' => 'children',
                'reservation' => 'children'
            ],
            'grouped' => [
                'manages_stock' => false, // Associated products manage stock
                'has_qty' => false,
                'stock_item' => 'associated',
                'reservation' => 'associated'
            ],
            'bundle' => [
                'manages_stock' => false, // Selections manage stock
                'has_qty' => false,
                'stock_item' => 'selections',
                'reservation' => 'selections'
            ],
            'virtual' => [
                'manages_stock' => false, // Usually no stock management
                'has_qty' => false,
                'stock_item' => 'none',
                'reservation' => 'none'
            ],
            'downloadable' => [
                'manages_stock' => false, // Unlimited digital copies
                'has_qty' => false,
                'stock_item' => 'none',
                'reservation' => 'none'
            ]
        ];
    }
}
```

### **Pricing Behavior by Type**
```php
class ProductTypePricingBehavior
{
    public function getPricingBehaviors()
    {
        return [
            'simple' => [
                'price_source' => 'direct',
                'supports_tier_pricing' => true,
                'supports_special_price' => true,
                'dynamic_pricing' => false
            ],
            'configurable' => [
                'price_source' => 'children_lowest',
                'supports_tier_pricing' => true,
                'supports_special_price' => true,
                'dynamic_pricing' => true,
                'price_display' => 'range_or_as_low_as'
            ],
            'grouped' => [
                'price_source' => 'associated_sum',
                'supports_tier_pricing' => false,
                'supports_special_price' => false,
                'dynamic_pricing' => true,
                'price_display' => 'starting_at'
            ],
            'bundle' => [
                'price_source' => 'selections_sum_or_fixed',
                'supports_tier_pricing' => true,
                'supports_special_price' => true,
                'dynamic_pricing' => true,
                'price_display' => 'range',
                'price_types' => ['fixed', 'dynamic']
            ],
            'virtual' => [
                'price_source' => 'direct',
                'supports_tier_pricing' => true,
                'supports_special_price' => true,
                'dynamic_pricing' => false
            ],
            'downloadable' => [
                'price_source' => 'direct_plus_links',
                'supports_tier_pricing' => true,
                'supports_special_price' => true,
                'dynamic_pricing' => true,
                'additional_pricing' => 'downloadable_links'
            ]
        ];
    }
}
```

### **Cart Behavior by Type**
```php
class ProductTypeCartBehavior
{
    public function getCartBehaviors()
    {
        return [
            'simple' => [
                'cart_items_count' => 1,
                'requires_selection' => false,
                'options_required' => false
            ],
            'configurable' => [
                'cart_items_count' => 1, // Adds child product
                'requires_selection' => true,
                'options_required' => true,
                'selection_type' => 'configurable_attributes'
            ],
            'grouped' => [
                'cart_items_count' => 'variable', // Multiple associated products
                'requires_selection' => true,
                'options_required' => false,
                'selection_type' => 'quantity_per_product'
            ],
            'bundle' => [
                'cart_items_count' => 'variable', // Bundle + selections
                'requires_selection' => true,
                'options_required' => true,
                'selection_type' => 'bundle_options'
            ],
            'virtual' => [
                'cart_items_count' => 1,
                'requires_selection' => false,
                'options_required' => false,
                'shipping_required' => false
            ],
            'downloadable' => [
                'cart_items_count' => 1,
                'requires_selection' => true,
                'options_required' => false,
                'selection_type' => 'downloadable_links',
                'shipping_required' => false
            ]
        ];
    }
}
```

---

## API Integration

### **REST API Product Type Handling**
```php
<?php

namespace Vendor\Module\Model\Api;

class ProductTypeHandler
{
    /**
     * Get product by type via REST API
     */
    public function getProductByType($sku, $storeId = null)
    {
        $product = $this->productRepository->get($sku, false, $storeId);
        
        switch ($product->getTypeId()) {
            case 'simple':
                return $this->formatSimpleProduct($product);
            case 'configurable':
                return $this->formatConfigurableProduct($product);
            case 'grouped':
                return $this->formatGroupedProduct($product);
            case 'bundle':
                return $this->formatBundleProduct($product);
            case 'virtual':
                return $this->formatVirtualProduct($product);
            case 'downloadable':
                return $this->formatDownloadableProduct($product);
            default:
                return $this->formatGenericProduct($product);
        }
    }
    
    private function formatConfigurableProduct($product)
    {
        $data = $this->formatGenericProduct($product);
        
        // Add configurable-specific data
        $data['configurable_options'] = $this->getConfigurableOptions($product);
        $data['variants'] = $this->getConfigurableVariants($product);
        
        return $data;
    }
    
    private function formatBundleProduct($product)
    {
        $data = $this->formatGenericProduct($product);
        
        // Add bundle-specific data
        $data['bundle_options'] = $this->getBundleOptions($product);
        $data['price_type'] = $product->getPriceType();
        $data['shipment_type'] = $product->getShipmentType();
        
        return $data;
    }
    
    private function formatDownloadableProduct($product)
    {
        $data = $this->formatVirtualProduct($product);
        
        // Add downloadable-specific data
        $data['downloadable_links'] = $this->getDownloadableLinks($product);
        $data['downloadable_samples'] = $this->getDownloadableSamples($product);
        
        return $data;
    }
}
```

### **GraphQL Product Type Schema**
```graphql
# GraphQL schema for different product types
interface ProductInterface {
    id: ID!
    sku: String!
    name: String!
    type_id: String!
    price: Money!
    image: String
    description: String
}

type SimpleProduct implements ProductInterface {
    id: ID!
    sku: String!
    name: String!
    type_id: String!
    price: Money!
    image: String
    description: String
    weight: Float
    dimensions: ProductDimensions
}

type ConfigurableProduct implements ProductInterface {
    id: ID!
    sku: String!
    name: String!
    type_id: String!
    price: Money!
    image: String
    description: String
    configurable_options: [ConfigurableOption!]!
    variants: [SimpleProduct!]!
}

type GroupedProduct implements ProductInterface {
    id: ID!
    sku: String!
    name: String!
    type_id: String!
    price: Money!
    image: String
    description: String
    items: [GroupedProductItem!]!
}

type BundleProduct implements ProductInterface {
    id: ID!
    sku: String!
    name: String!
    type_id: String!
    price: Money!
    image: String
    description: String
    items: [BundleItem!]!
    dynamic_price: Boolean!
    dynamic_sku: Boolean!
    dynamic_weight: Boolean!
    price_view: PriceViewEnum!
    shipment_type: ShipmentTypeEnum!
}

type DownloadableProduct implements ProductInterface {
    id: ID!
    sku: String!
    name: String!
    type_id: String!
    price: Money!
    image: String
    description: String
    downloadable_product_links: [DownloadableProductLinks]
    downloadable_product_samples: [DownloadableProductSamples]
}
```

---

## Performance Considerations

### **Product Type Performance Impact**
```php
class ProductTypePerformanceAnalysis
{
    public function getPerformanceMetrics()
    {
        return [
            'simple' => [
                'load_time' => 'fast',
                'memory_usage' => 'low',
                'database_queries' => 'minimal',
                'cache_efficiency' => 'high'
            ],
            'configurable' => [
                'load_time' => 'medium',
                'memory_usage' => 'medium',
                'database_queries' => 'moderate', // Loads child products
                'cache_efficiency' => 'medium'
            ],
            'grouped' => [
                'load_time' => 'medium',
                'memory_usage' => 'medium',
                'database_queries' => 'moderate', // Loads associated products
                'cache_efficiency' => 'medium'
            ],
            'bundle' => [
                'load_time' => 'slow',
                'memory_usage' => 'high',
                'database_queries' => 'many', // Loads options + selections
                'cache_efficiency' => 'low'
            ],
            'virtual' => [
                'load_time' => 'fast',
                'memory_usage' => 'low',
                'database_queries' => 'minimal',
                'cache_efficiency' => 'high'
            ],
            'downloadable' => [
                'load_time' => 'medium',
                'memory_usage' => 'medium',
                'database_queries' => 'moderate', // Loads links + samples
                'cache_efficiency' => 'medium'
            ]
        ];
    }
}
```

### **Optimization Strategies**
```php
class ProductTypeOptimization
{
    /**
     * Optimize configurable product loading
     */
    public function optimizeConfigurableLoading($product)
    {
        // Lazy load variants
        if (!$product->hasData('_cache_instance_used_products')) {
            $usedProducts = $this->loadUsedProductsOptimized($product);
            $product->setData('_cache_instance_used_products', $usedProducts);
        }
        
        return $product;
    }
    
    /**
     * Optimize bundle product loading
     */
    public function optimizeBundleLoading($product)
    {
        // Use separate queries instead of joins
        $options = $this->bundleOptionCollection->getOptionsByProduct($product);
        $selections = $this->bundleSelectionCollection->getSelectionsByOptions($options);
        
        // Cache the results
        $this->cache->save(
            serialize($options),
            'bundle_options_' . $product->getId(),
            ['bundle_product_' . $product->getId()]
        );
        
        return $product;
    }
    
    /**
     * Cache downloadable links separately
     */
    public function optimizeDownloadableLoading($product)
    {
        $cacheKey = 'downloadable_links_' . $product->getId();
        $links = $this->cache->load($cacheKey);
        
        if (!$links) {
            $links = $this->linkCollection->getLinks($product);
            $this->cache->save(serialize($links), $cacheKey, ['downloadable_' . $product->getId()]);
        } else {
            $links = unserialize($links);
        }
        
        return $links;
    }
}
```

---

## Best Practices

### **1. Product Type Selection Guidelines**
```php
class ProductTypeSelectionGuide
{
    public function getRecommendations()
    {
        return [
            'use_simple_for' => [
                'Basic physical products',
                'Products without variations',
                'Digital products without downloads',
                'Services with fixed pricing'
            ],
            'use_configurable_for' => [
                'Products with color/size variations',
                'Products with different specifications',
                'Products where variants share most attributes',
                'When you need variation-specific inventory'
            ],
            'use_grouped_for' => [
                'Related product collections',
                'Product kits with optional items',
                'When customers can buy items separately',
                'Cross-selling scenarios'
            ],
            'use_bundle_for' => [
                'Build-your-own product scenarios',
                'Complex product configurations',
                'When you need dynamic pricing',
                'Customizable product packages'
            ],
            'use_virtual_for' => [
                'Services and consultations',
                'Warranties and extended services',
                'Non-physical products',
                'Subscription services'
            ],
            'use_downloadable_for' => [
                'Digital downloads (files, software)',
                'E-books and documents',
                'Digital media (music, videos)',
                'Software licenses'
            ]
        ];
    }
}
```

### **2. Performance Best Practices**
```php
class ProductTypePerformanceBestPractices
{
    public function getBestPractices()
    {
        return [
            'general' => [
                'Use appropriate caching strategies',
                'Implement lazy loading for related products',
                'Optimize database queries with proper indexing',
                'Use flat catalog for better performance'
            ],
            'configurable' => [
                'Limit number of variants per product',
                'Use proper attribute configuration',
                'Cache configurable options',
                'Optimize variant loading'
            ],
            'bundle' => [
                'Limit bundle complexity',
                'Cache bundle options and selections',
                'Use indexed pricing',
                'Optimize bundle calculations'
            ],
            'downloadable' => [
                'Use CDN for download files',
                'Implement proper download tracking',
                'Cache link information',
                'Optimize file delivery'
            ]
        ];
    }
}
```

### **3. Development Best Practices**
```php
class ProductTypeDevelopmentBestPractices
{
    public function getGuidelines()
    {
        return [
            'code_structure' => [
                'Follow Magento 2 architecture patterns',
                'Use dependency injection properly',
                'Implement proper error handling',
                'Follow coding standards'
            ],
            'custom_types' => [
                'Extend existing types when possible',
                'Implement all required methods',
                'Provide proper documentation',
                'Add comprehensive tests'
            ],
            'testing' => [
                'Unit test product type logic',
                'Integration test with cart functionality',
                'Performance test with large datasets',
                'Test API endpoints'
            ],
            'maintenance' => [
                'Monitor performance metrics',
                'Keep type definitions updated',
                'Document custom modifications',
                'Plan for Magento upgrades'
            ]
        ];
    }
}
```

---

## Summary

### **Product Type Quick Reference**
| Type | Use Case | Complexity | Performance | Inventory |
|------|----------|------------|-------------|-----------|
| **Simple** | Basic products | Low | High | Direct |
| **Configurable** | Products with variations | Medium | Medium | Children |
| **Grouped** | Product collections | Medium | Medium | Associated |
| **Bundle** | Build-your-own | High | Low | Selections |
| **Virtual** | Services/Non-physical | Low | High | None |
| **Downloadable** | Digital downloads | Medium | Medium | None |

### **Key Takeaways**
1. **Choose the right product type** based on business requirements
2. **Understand performance implications** of each type
3. **Implement proper caching strategies** for complex types
4. **Follow Magento 2 architecture patterns** for custom types
5. **Test thoroughly** across all channels (frontend, admin, API)
6. **Monitor performance** and optimize as needed

This comprehensive guide provides the foundation for working with all Magento 2 product types, from basic implementation to advanced customization and optimization strategies.