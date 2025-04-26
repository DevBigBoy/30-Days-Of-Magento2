# 🧠 First, What Are Events and Observers in Magento 2?

**Event** = a signal that something *happened* in the system (e.g., a product was saved, a customer logged in).  
**Observer** = a piece of code that *listens* to that event and *reacts* by doing something.

**Analogy**:  
Imagine Magento is a huge party 🥳. Every time someone enters (an event), a photographer takes their picture (the observer).

---
# 📚 Basic Theory of Events and Observers:

- **Event** = Magento "dispatches" an event with some **data**.
- **Observer** = You "observe" this event and **execute your custom logic** when it happens.

**Magento 2 has two types of events**:
- Core Events (Magento fires them by default, e.g., `controller_action_predispatch`, `customer_login`, `sales_order_place_after`)
- Custom Events (you can dispatch your own event manually)

---

# 📦 Full Step-by-Step Tutorial

I'll show **5 examples** you can actually use.

---

## 🛠️ Setup: Create a Simple Module

First, you need a module to place your observers.

**Folder structure**:

```
app/code/Pkl/ObserverDemo
```

**Files**:

1. `app/code/Pkl/ObserverDemo/registration.php`
```php
<?php
use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Pkl_ObserverDemo',
    __DIR__
);
```

2. `app/code/Pkl/ObserverDemo/etc/module.xml`
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Pkl_ObserverDemo" setup_version="1.0.0" />
</config>
```

**Enable your module**:
```bash
php bin/magento setup:upgrade
php bin/magento module:enable Pkl_ObserverDemo
```

✅ Now you’re ready to build Observers!

---

# 🧑‍🏫 Example 1: **Log when customer logs in**

**Magento event**: `customer_login`

---
### 1. Create `events.xml`

**Path**: `app/code/Pkl/ObserverDemo/etc/frontend/events.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="customer_login">
        <observer name="pkl_customer_login_logger" instance="Pkl\ObserverDemo\Observer\CustomerLoginObserver" />
    </event>
</config>
```

---
### 2. Create the Observer Class

**Path**: `app/code/Pkl/ObserverDemo/Observer/CustomerLoginObserver.php`

```php
<?php
namespace Pkl\ObserverDemo\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;
use Psr\Log\LoggerInterface;

class CustomerLoginObserver implements ObserverInterface
{
    protected $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function execute(Observer $observer)
    {
        $customer = $observer->getEvent()->getCustomer();
        $this->logger->info('Customer Logged In: ' . $customer->getEmail());
    }
}
```

---
✅ Now, when a customer logs in, their email is logged to `var/log/system.log`!

---

# 🧑‍🏫 Example 2: **Change product name before saving**

**Magento event**: `catalog_product_save_before`

---
### 1. Update `events.xml`

Same file as before, just add:

```xml
<event name="catalog_product_save_before">
    <observer name="pkl_modify_product_name" instance="Pkl\ObserverDemo\Observer\ProductSaveBeforeObserver" />
</event>
```

---
### 2. Create the Observer Class

**Path**: `app/code/Pkl/ObserverDemo/Observer/ProductSaveBeforeObserver.php`

```php
<?php
namespace Pkl\ObserverDemo\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;

class ProductSaveBeforeObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $product = $observer->getEvent()->getProduct();
        $productName = $product->getName();
        $product->setName($productName . ' [Modified]');
    }
}
```

---
✅ Now when you save any product, it will automatically append `[Modified]` to the product name.

---

# 🧑‍🏫 Example 3: **Add custom logic when order is placed**

**Magento event**: `sales_order_place_after`

---
### 1. Update `events.xml`

Add:

```xml
<event name="sales_order_place_after">
    <observer name="pkl_order_place_after" instance="Pkl\ObserverDemo\Observer\OrderPlaceObserver" />
</event>
```

---
### 2. Create Observer Class

**Path**: `app/code/Pkl/ObserverDemo/Observer/OrderPlaceObserver.php`

```php
<?php
namespace Pkl\ObserverDemo\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;
use Psr\Log\LoggerInterface;

class OrderPlaceObserver implements ObserverInterface
{
    protected $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function execute(Observer $observer)
    {
        $order = $observer->getEvent()->getOrder();
        $this->logger->info('Order Placed: #' . $order->getIncrementId());
    }
}
```

---
✅ Now every time an order is placed, the order number will be logged.

---

# 🧑‍🏫 Example 4: **Create a custom event and observe it**

---
### 1. Dispatch a Custom Event

In any custom Controller/Helper/Model, dispatch your event like this:

```php
$this->_eventManager->dispatch(
    'pkl_custom_event',
    ['data' => 'Hello from custom event!']
);
```

---
### 2. Create `events.xml` for it

```xml
<event name="pkl_custom_event">
    <observer name="pkl_custom_event_observer" instance="Pkl\ObserverDemo\Observer\CustomEventObserver" />
</event>
```

---
### 3. Create Observer Class

```php
<?php
namespace Pkl\ObserverDemo\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;
use Psr\Log\LoggerInterface;

class CustomEventObserver implements ObserverInterface
{
    protected $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function execute(Observer $observer)
    {
        $data = $observer->getEvent()->getData('data');
        $this->logger->info('Custom Event Triggered: ' . $data);
    }
}
```

---
✅ You just created and listened to your own event!

---

# 🧑‍🏫 Example 5: **Automatically send an email after product save**

_(This is a slightly more real-world example)_

---
### 1. Update `events.xml`

```xml
<event name="catalog_product_save_after">
    <observer name="pkl_product_save_send_email" instance="Pkl\ObserverDemo\Observer\ProductSaveAfterObserver" />
</event>
```

---
### 2. Create Observer Class

```php
<?php
namespace Pkl\ObserverDemo\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;
use Magento\Framework\Mail\Template\TransportBuilder;
use Magento\Store\Model\StoreManagerInterface;

class ProductSaveAfterObserver implements ObserverInterface
{
    protected $transportBuilder;
    protected $storeManager;

    public function __construct(
        TransportBuilder $transportBuilder,
        StoreManagerInterface $storeManager
    ) {
        $this->transportBuilder = $transportBuilder;
        $this->storeManager = $storeManager;
    }

    public function execute(Observer $observer)
    {
        $product = $observer->getEvent()->getProduct();
        
        $transport = $this->transportBuilder
            ->setTemplateIdentifier('pkl_product_saved_email_template') // You need to create this template in Admin
            ->setTemplateOptions([
                'area' => \Magento\Framework\App\Area::AREA_FRONTEND,
                'store' => $this->storeManager->getStore()->getId(),
            ])
            ->setTemplateVars(['product_name' => $product->getName()])
            ->setFrom(['email' => "sender@example.com", 'name' => 'Sender'])
            ->addTo('admin@example.com')
            ->getTransport();

        $transport->sendMessage();
    }
}
```

---
✅ Now after saving a product, an email notification can be sent automatically!

---

# 🔥 Summary

| Concept | Magento 2 Term | Example |
|:-------|:--------------|:--------|
| Something Happens | Event | `customer_login` |
| React to Event | Observer | Log customer email |
| Make Own Event | Dispatch Custom Event | `pkl_custom_event` |
| Do Complex Actions | Business Logic in Observer | Email after save |

---

# ✅ Pro Tips

- Always **log** inside Observers to debug (`$this->logger->info()`).
- Don't make Observers **heavy** (e.g., don’t load lots of models).
- If you modify entities in `*_save_before`, Magento will **save your changes** automatically.
- Be sure to register Observers in the right area (`global`, `frontend`, or `adminhtml`).

---

# 🧑‍🏫 I hope this super detailed tutorial really helps you!

Would you also like me to create a real-world example like "**Add a discount if order total > $500 via observer**"?  
I can show even more advanced stuff if you want! 🚀  
Would you like that?