# Day 01: Introduction to Magento 2

## What is Magento 2?

Magento 2 is an open-source e-commerce platform written in PHP that provides online merchants with a flexible shopping cart system, as well as control over the look, content, and functionality of their online store.

### Key Features

- **Open Source & Scalable**: Built on modern technologies with support for millions of products and customers
- **Multi-Store Management**: Manage multiple stores from a single admin panel
- **Mobile Commerce**: Responsive design and mobile-optimized checkout
- **SEO Friendly**: Built-in SEO capabilities and URL optimization
- **Flexible Payment & Shipping**: Support for multiple payment gateways and shipping methods
- **Advanced Marketing Tools**: Promotions, coupons, cross-sells, up-sells
- **Powerful API**: REST and GraphQL APIs for integrations
- **Security**: PCI compliance and advanced security features

### Magento Editions

1. **Magento Open Source** (formerly Community Edition)
   - Free to use
   - Community-driven development
   - Basic e-commerce features

2. **Adobe Commerce** (formerly Magento Commerce/Enterprise Edition)
   - Commercial license
   - Advanced features (B2B, staging, advanced reporting)
   - Cloud infrastructure options
   - Professional support

## Magento 2 Architecture Overview

Magento 2 follows a modular architecture based on several key design patterns and principles.

### Layered Architecture

Magento 2 is organized into four main layers:

```
┌─────────────────────────────────────────────┐
│         Presentation Layer                  │
│  (Blocks, Controllers, Templates, Layouts)  │
├─────────────────────────────────────────────┤
│         Service Layer                       │
│  (Service Contracts, APIs, Repositories)    │
├─────────────────────────────────────────────┤
│         Domain Layer                        │
│  (Business Logic, Models, Resource Models)  │
├─────────────────────────────────────────────┤
│         Persistence Layer                   │
│  (Database, File System, Cache)             │
└─────────────────────────────────────────────┘
```

#### 1. Presentation Layer

- **Responsibility**: Handles web requests and displays data to users
- **Components**:
  - Controllers (handle HTTP requests)
  - Blocks (prepare data for templates)
  - Templates (PHTML files for HTML output)
  - Layouts (XML files defining page structure)
  - UI Components (complex interface elements)

#### 2. Service Layer

- **Responsibility**: Provides a stable API for business logic
- **Components**:
  - Service Contracts (interfaces defining public APIs)
  - Repositories (data access patterns)
  - Data interfaces (data transfer objects)
  - API modules (REST/SOAP/GraphQL endpoints)

#### 3. Domain Layer

- **Responsibility**: Contains business logic and domain models
- **Components**:
  - Models (business entities)
  - Resource Models (data access)
  - Collections (groups of models)
  - Business logic classes

#### 4. Persistence Layer

- **Responsibility**: Handles data storage and retrieval
- **Components**:
  - Database adapters
  - ORM (Object-Relational Mapping)
  - Indexers
  - Cache management

### Modular Architecture

Magento 2 is built using a **modular architecture** where functionality is divided into modules.

#### Module Types

1. **Core Modules**: Located in `vendor/magento/module-*`
2. **Custom Modules**: Located in `app/code/{Vendor}/{ModuleName}`
3. **Community Modules**: Installed via Composer

#### Module Structure

```
app/code/Vendor/ModuleName/
├── Api/                    # Service contracts (interfaces)
├── Block/                  # Block classes
├── Controller/             # Controllers
├── etc/                    # Configuration files
│   ├── module.xml         # Module declaration
│   ├── di.xml             # Dependency injection
│   └── ...
├── Model/                  # Models and business logic
├── Setup/                  # Installation/upgrade scripts
├── Test/                   # Unit and integration tests
├── Ui/                     # UI components
├── view/                   # Frontend/adminhtml templates & layouts
│   ├── frontend/
│   └── adminhtml/
├── registration.php        # Module registration
└── composer.json          # Composer dependencies
```

### Key Architectural Concepts

#### 1. Dependency Injection (DI)

Magento 2 uses dependency injection for managing class dependencies.

```php
class MyClass
{
    protected $productRepository;

    public function __construct(
        \Magento\Catalog\Api\ProductRepositoryInterface $productRepository
    ) {
        $this->productRepository = $productRepository;
    }
}
```

#### 2. Service Contracts

Interfaces that define public APIs for modules, ensuring backward compatibility.

```php
interface ProductRepositoryInterface
{
    public function save(ProductInterface $product);
    public function get($sku);
    public function delete(ProductInterface $product);
}
```

#### 3. Plugins (Interceptors)

Allow you to modify the behavior of public methods without changing the original class.

```php
class MyPlugin
{
    public function beforeSave($subject, $product) { }
    public function afterSave($subject, $result) { }
    public function aroundSave($subject, $proceed, $product) { }
}
```

#### 4. Events & Observers

Event-driven architecture for loose coupling between modules.

```php
// Dispatch event
$this->eventManager->dispatch('event_name', ['data' => $data]);

// Observe event
class MyObserver implements ObserverInterface
{
    public function execute(Observer $observer) { }
}
```

## MVC Pattern in Magento 2

Magento 2 implements a modified Model-View-Controller (MVC) pattern.

### MVC Components

```
┌──────────────────────────────────────────────┐
│                  Request                     │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │     Controller       │ ◄── Front Controller
        │   (Action Class)     │     Routes request
        └──────┬───────────────┘
               │
               ├─────────────────┐
               ▼                 ▼
        ┌──────────┐      ┌──────────┐
        │  Model   │      │  Block   │
        │          │      │          │
        └──────┬───┘      └─────┬────┘
               │                │
               │                ▼
               │         ┌──────────┐
               │         │ Template │
               │         │  (View)  │
               │         └─────┬────┘
               │               │
               ▼               ▼
        ┌──────────────────────────┐
        │       Response           │
        └──────────────────────────┘
```

### 1. Model

**Responsibility**: Business logic, data handling, and database interaction

**Components**:
- **Models**: Represent business entities
- **Resource Models**: Handle database operations
- **Collections**: Groups of model instances
- **Repositories**: Data access layer (part of Service Contracts)

**Example**:

```php
namespace Vendor\Module\Model;

use Magento\Framework\Model\AbstractModel;

class Product extends AbstractModel
{
    protected function _construct()
    {
        $this->_init(\Vendor\Module\Model\ResourceModel\Product::class);
    }

    public function getDiscountedPrice()
    {
        // Business logic
        return $this->getPrice() * 0.9;
    }
}
```

**Resource Model**:

```php
namespace Vendor\Module\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Product extends AbstractDb
{
    protected function _construct()
    {
        $this->_init('custom_product_table', 'entity_id');
    }
}
```

### 2. View

**Responsibility**: Presentation logic and UI rendering

**Components**:
- **Layouts**: XML files defining page structure
- **Templates**: PHTML files containing HTML and PHP
- **Blocks**: PHP classes that prepare data for templates
- **UI Components**: Complex reusable interface elements

**Layout Example** (`view/frontend/layout/catalog_product_view.xml`):

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <body>
        <referenceContainer name="content">
            <block class="Vendor\Module\Block\Product"
                   name="custom.product.block"
                   template="Vendor_Module::product/view.phtml"/>
        </referenceContainer>
    </body>
</page>
```

**Block Example**:

```php
namespace Vendor\Module\Block;

use Magento\Framework\View\Element\Template;

class Product extends Template
{
    protected $productRepository;

    public function __construct(
        Template\Context $context,
        \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        array $data = []
    ) {
        $this->productRepository = $productRepository;
        parent::__construct($context, $data);
    }

    public function getProduct()
    {
        return $this->productRepository->get('product-sku');
    }
}
```

**Template Example** (`view/frontend/templates/product/view.phtml`):

```php
<?php
/** @var \Vendor\Module\Block\Product $block */
$product = $block->getProduct();
?>
<div class="product-info">
    <h1><?= $block->escapeHtml($product->getName()) ?></h1>
    <p>Price: <?= $block->escapeHtml($product->getPrice()) ?></p>
</div>
```

### 3. Controller

**Responsibility**: Handle HTTP requests and coordinate between Model and View

**Components**:
- **Routers**: Map URLs to controllers
- **Action Classes**: Handle specific requests
- **Result Objects**: Generate responses

**Controller Example**:

```php
namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\View\Result\PageFactory;

class Index extends Action
{
    protected $resultPageFactory;

    public function __construct(
        Context $context,
        PageFactory $resultPageFactory
    ) {
        $this->resultPageFactory = $resultPageFactory;
        parent::__construct($context);
    }

    public function execute()
    {
        // Load layout and render page
        return $this->resultPageFactory->create();
    }
}
```

**API Controller Example**:

```php
namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\Action;
use Magento\Framework\Controller\Result\JsonFactory;

class Data extends Action
{
    protected $resultJsonFactory;

    public function execute()
    {
        $result = $this->resultJsonFactory->create();
        return $result->setData(['success' => true, 'message' => 'Data loaded']);
    }
}
```

## Magento 2 Request Flow

```
1. index.php
   ↓
2. Bootstrap
   ↓
3. Front Controller (handles all requests)
   ↓
4. Router (matches URL to controller)
   ↓
5. Controller Action (executes business logic)
   ↓
6. Model/Repository (data operations)
   ↓
7. Block (prepares data for view)
   ↓
8. Template (renders HTML)
   ↓
9. Response
```

### Detailed Flow

1. **Request arrives** at `pub/index.php`
2. **Bootstrap** initializes the application
3. **Front Controller** receives the request
4. **Router** matches URL to appropriate controller
5. **Controller** executes and may:
   - Call models/repositories for data
   - Dispatch events
   - Create result objects
6. **Layout** is loaded and processed
7. **Blocks** are instantiated and prepare data
8. **Templates** are rendered with block data
9. **Response** is sent back to client

## Key Takeaways

1. **Magento 2** is a powerful, modular e-commerce platform
2. **Layered Architecture** separates concerns across presentation, service, domain, and persistence layers
3. **Modular Design** allows for flexible, maintainable code organization
4. **MVC Pattern** structures code into Models (data), Views (presentation), and Controllers (request handling)
5. **Service Contracts** provide stable APIs for module interaction
6. **Dependency Injection** manages class dependencies and promotes loose coupling

## Next Steps

- [Day 02: Installation & Setup](../Day-02-Installation/README.md)
- Explore the Magento 2 codebase structure
- Review official Magento 2 architecture documentation

## Additional Resources

- [Magento 2 Developer Documentation](https://devdocs.magento.com/)
- [Magento 2 Architecture Guide](https://devdocs.magento.com/guides/v2.4/architecture/bk-architecture.html)
- [Magento 2 GitHub Repository](https://github.com/magento/magento2)
