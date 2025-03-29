

# 🧠 What is a View Model in Magento 2?

A **view model** is a PHP class that’s injected into your templates (`.phtml`) and meant *specifically* to prepare data for the view. Think of it as the middleman between your layout and the actual HTML.

View models are **block-less data providers**—they help keep your templates clean and your business logic *outside* of the `.phtml` files.

---

### ✅ When to Use a View Model

Here are the **best use cases** for using a view model:

---

#### 1. **To Move Logic Out of Templates**
Templates should be super simple—just loops, `if` statements, and rendering. If you find yourself writing logic like this in a `.phtml`:

```php
<?php
$today = new \DateTime();
$isNew = $product->getCreatedAt() > $today->modify('-30 days');
?>
```

👉 That logic belongs in a **view model**, not in the view.

---

#### 2. **You Don't Want to Extend or Create a Block**
Traditionally, we’d create or extend a block just to pass data to the template. View models let you avoid bloating the block layer when all you need is to pass data.

If your block has no other purpose than to provide helpers or values—it’s a candidate for a view model.

---

#### 3. **Reusability & Testability**
View models are **injectable**, so you can:
- Reuse them across multiple templates
- Easily test them in unit tests (unlike blocks that require layout setup)

---

#### 4. **Inject Services or Helpers**
If you need to inject services like repositories, helpers, or other dependencies into your template—do it through a view model. For example:

```php
<?php
$stock = $viewModel->getStockStatus($product->getId());
?>
```

This avoids calling helpers or loading objects directly in the `.phtml`.

---

#### 5. **You’re Using Knockout or JS Components**
In UI components or Knockout templates, view models are a natural fit to prepare structured data to pass to JS.

---

### 📦 Real Example: Adding a View Model

Let’s say we want to show a custom product label if it was created in the last 30 days.

#### a. Create the View Model class:

`YourVendor/ModuleName/ViewModel/ProductLabel.php`

```php
<?php
namespace YourVendor\ModuleName\ViewModel;

use Magento\Framework\View\Element\Block\ArgumentInterface;
use Magento\Catalog\Api\ProductRepositoryInterface;
use Magento\Framework\Stdlib\DateTime\DateTime;

class ProductLabel implements ArgumentInterface
{
    protected $productRepository;
    protected $date;

    public function __construct(
        ProductRepositoryInterface $productRepository,
        DateTime $date
    ) {
        $this->productRepository = $productRepository;
        $this->date = $date;
    }

    public function isNew($product)
    {
        $createdAt = strtotime($product->getCreatedAt());
        $now = strtotime($this->date->gmtDate());
        $daysSinceCreation = ($now - $createdAt) / (60 * 60 * 24);
        return $daysSinceCreation <= 30;
    }
}
```

#### b. Inject ViewModel into layout:

In your layout XML (e.g., `catalog_product_view.xml`):

```xml
<block class="Magento\Framework\View\Element\Template" name="custom.product.label" template="YourVendor_ModuleName::product/label.phtml">
    <arguments>
        <argument name="view_model" xsi:type="object">YourVendor\ModuleName\ViewModel\ProductLabel</argument>
    </arguments>
</block>
```

#### c. Use in template:

```php
<?php if ($block->getViewModel()->isNew($_product)): ?>
    <span class="label-new">New!</span>
<?php endif; ?>
```

---

### 🚫 When *Not* to Use View Models

- When your logic belongs to the **business layer** (that should go in models/services)
- When you already have a **complex block class** and extending it makes more sense
- If you need **full access to layout elements**, blocks are better

---

### 💬 TL;DR

| Use Case | Use View Model? |
|----------|------------------|
| Prepare simple display logic? | ✅ Yes |
| Inject helpers/services into `.phtml`? | ✅ Yes |
| Template getting bloated with PHP logic? | ✅ Yes |
| Need to manipulate layout or child blocks? | ❌ Use a block instead |
| Need core Magento block features (cache, identity)? | ❌ Use a block |

