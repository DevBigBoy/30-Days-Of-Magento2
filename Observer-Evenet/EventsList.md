# Magento 2 Events - Complete Categorized Guide with API Compatibility

## Table of Contents
1. [Event System Overview](#event-system-overview)
2. [API Compatibility](#api-compatibility)
3. [Customer Events](#customer-events)
4. [Order Events](#order-events)
5. [Product Events](#product-events)
6. [Category Events](#category-events)
7. [Cart/Quote Events](#cartquote-events)
8. [Payment Events](#payment-events)
9. [Shipping Events](#shipping-events)
10. [Inventory Events](#inventory-events)
11. [Admin Events](#admin-events)
12. [System Events](#system-events)
13. [Email Events](#email-events)
14. [CMS Events](#cms-events)
15. [Search Events](#search-events)
16. [API-Specific Events](#api-specific-events)

---

## Event System Overview

### Event Types in Magento 2
- **Model Events** - Triggered by model operations (save, load, delete)
- **Controller Events** - Triggered by controller actions
- **Block Events** - Triggered during block rendering
- **System Events** - Triggered by system operations
- **Custom Events** - Triggered by custom code

### Event Naming Convention
```
[module]_[entity]_[action]_[timing]

Examples:
- customer_save_before
- sales_order_place_after
- catalog_product_load_after
```

---

## API Compatibility

### Events and REST API
**REST API Events:**
- Triggered when operations happen through REST endpoints
- Same events as frontend/admin but with API context
- Additional headers and authentication context available

### Events and GraphQL
**GraphQL Events:**
- Triggered during GraphQL resolver execution
- Same core events but with GraphQL-specific context
- Can access GraphQL query/mutation information

### API-Triggered Event Context
```php
// Check if event was triggered by API
$request = $observer->getEvent()->getRequest();
if ($request && $request->isApi()) {
    // Handle API-specific logic
}

// Check if GraphQL
if ($request && strpos($request->getPathInfo(), '/graphql') !== false) {
    // Handle GraphQL-specific logic
}
```

---

## Customer Events

### 🔥 **Core Customer Events**

#### **Authentication & Registration**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `customer_register_success` | ✅ REST/GraphQL | After customer registration | Welcome emails, loyalty setup |
| `customer_login` | ✅ REST/GraphQL | Customer login | Track login activity, security |
| `customer_logout` | ✅ REST/GraphQL | Customer logout | Session cleanup, analytics |
| `customer_password_reset` | ✅ REST/GraphQL | Password reset request | Security notifications |

#### **Customer Data Operations**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `customer_save_before` | ✅ REST/GraphQL | Before customer save | Data validation, preprocessing |
| `customer_save_after` | ✅ REST/GraphQL | After customer save | CRM sync, notifications |
| `customer_delete_before` | ✅ REST/GraphQL | Before customer deletion | Data backup, cleanup prep |
| `customer_delete_after` | ✅ REST/GraphQL | After customer deletion | External system cleanup |
| `customer_load_after` | ✅ REST/GraphQL | After customer load | Data enrichment |

#### **Customer Address Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `customer_address_save_before` | ✅ REST/GraphQL | Before address save | Address validation |
| `customer_address_save_after` | ✅ REST/GraphQL | After address save | Tax calculation updates |
| `customer_address_delete_before` | ✅ REST/GraphQL | Before address deletion | Dependency checks |
| `customer_address_delete_after` | ✅ REST/GraphQL | After address deletion | Cache cleanup |

### **Example Implementation**
```php
// Observer for customer registration
class CustomerRegisterObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $customer = $observer->getEvent()->getCustomer();
        $request = $observer->getEvent()->getRequest();
        
        // Check if API request
        if ($request && $request->isApi()) {
            // API-specific handling
            $this->handleApiRegistration($customer);
        } else {
            // Frontend registration
            $this->handleFrontendRegistration($customer);
        }
    }
}
```

---

## Order Events

### 🛒 **Order Lifecycle Events**

#### **Order Creation & Placement**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_place_before` | ✅ REST/GraphQL | Before order placement | Final validation, fraud check |
| `sales_order_place_after` | ✅ REST/GraphQL | After order placement | Notifications, inventory reserve |
| `checkout_onepage_controller_success_action` | ❌ Frontend Only | Frontend checkout success | Frontend-specific actions |
| `checkout_submit_all_after` | ✅ REST/GraphQL | After checkout submission | Post-checkout processing |

#### **Order Status & State Changes**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_save_before` | ✅ REST/GraphQL | Before order save | Status validation |
| `sales_order_save_after` | ✅ REST/GraphQL | After order save | Status notifications |
| `sales_order_state_change_before` | ✅ REST/GraphQL | Before state change | State validation |
| `sales_order_cancel_after` | ✅ REST/GraphQL | After order cancellation | Inventory return, notifications |

#### **Order Item Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_item_save_before` | ✅ REST/GraphQL | Before order item save | Item validation |
| `sales_order_item_save_after` | ✅ REST/GraphQL | After order item save | Item processing |
| `sales_order_item_cancel_after` | ✅ REST/GraphQL | After item cancellation | Inventory management |

### **Invoice Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_invoice_save_before` | ✅ REST/GraphQL | Before invoice save | Invoice validation |
| `sales_order_invoice_save_after` | ✅ REST/GraphQL | After invoice save | Payment processing |
| `sales_order_invoice_pay` | ✅ REST/GraphQL | Invoice payment capture | Accounting integration |
| `sales_order_invoice_register` | ✅ REST/GraphQL | Invoice registration | Financial records |

### **Shipment Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_shipment_save_before` | ✅ REST/GraphQL | Before shipment save | Shipping validation |
| `sales_order_shipment_save_after` | ✅ REST/GraphQL | After shipment save | Tracking setup |
| `sales_order_shipment_track_save_after` | ✅ REST/GraphQL | After tracking save | Customer notifications |

### **Credit Memo Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_creditmemo_save_before` | ✅ REST/GraphQL | Before credit memo save | Refund validation |
| `sales_order_creditmemo_save_after` | ✅ REST/GraphQL | After credit memo save | Refund processing |
| `sales_order_creditmemo_refund` | ✅ REST/GraphQL | Credit memo refund | Payment gateway refund |

---

## Product Events

### 📦 **Product Management Events**

#### **Product CRUD Operations**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `catalog_product_save_before` | ✅ REST/GraphQL | Before product save | Data validation, pricing rules |
| `catalog_product_save_after` | ✅ REST/GraphQL | After product save | Search indexing, cache clear |
| `catalog_product_delete_before` | ✅ REST/GraphQL | Before product deletion | Dependency checks |
| `catalog_product_delete_after` | ✅ REST/GraphQL | After product deletion | Cleanup, reindexing |
| `catalog_product_load_after` | ✅ REST/GraphQL | After product load | Data enrichment |

#### **Product Import/Export Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `catalog_product_import_bunch_delete_commit_before` | ⚠️ Admin/Import | Before import deletion | Pre-deletion processing |
| `catalog_product_import_bunch_save_after` | ⚠️ Admin/Import | After import batch save | Post-import processing |
| `catalog_product_attribute_update_before` | ✅ REST/GraphQL | Before attribute update | Validation |
| `catalog_product_media_save_before` | ✅ REST/GraphQL | Before media save | Image processing |
| `catalog_product_media_save_after` | ✅ REST/GraphQL | After media save | CDN sync |

#### **Product Collection Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `catalog_product_collection_load_before` | ✅ REST/GraphQL | Before collection load | Filter modification |
| `catalog_product_collection_load_after` | ✅ REST/GraphQL | After collection load | Data processing |
| `catalog_product_collection_apply_limitations_after` | ✅ REST/GraphQL | After limitations applied | Custom filtering |

---

## Category Events

### 🗂️ **Category Management Events**

| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `catalog_category_save_before` | ✅ REST/GraphQL | Before category save | Validation, URL key generation |
| `catalog_category_save_after` | ✅ REST/GraphQL | After category save | Menu rebuild, cache clear |
| `catalog_category_delete_before` | ✅ REST/GraphQL | Before category deletion | Product reassignment |
| `catalog_category_delete_after` | ✅ REST/GraphQL | After category deletion | Cleanup, reindexing |
| `catalog_category_move_before` | ✅ REST/GraphQL | Before category move | Structure validation |
| `catalog_category_move_after` | ✅ REST/GraphQL | After category move | URL updates |
| `catalog_category_tree_init_inactive_category_ids` | ✅ REST/GraphQL | Category tree initialization | Permission filtering |

---

## Cart/Quote Events

### 🛒 **Shopping Cart Events**

#### **Cart Operations**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `checkout_cart_add_product_complete` | ❌ Frontend Only | After product added to cart | Frontend notifications |
| `checkout_cart_update_items_before` | ❌ Frontend Only | Before cart update | Frontend validation |
| `checkout_cart_update_items_after` | ❌ Frontend Only | After cart update | Frontend notifications |
| `checkout_cart_product_add_after` | ❌ Frontend Only | After product add | Analytics tracking |

#### **Quote Events (API Compatible)**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_quote_save_before` | ✅ REST/GraphQL | Before quote save | Quote validation |
| `sales_quote_save_after` | ✅ REST/GraphQL | After quote save | Price calculations |
| `sales_quote_add_item` | ✅ REST/GraphQL | Item added to quote | Inventory checks |
| `sales_quote_remove_item` | ✅ REST/GraphQL | Item removed from quote | Inventory release |
| `sales_quote_merge_before` | ✅ REST/GraphQL | Before quote merge | Guest to customer merge |
| `sales_quote_merge_after` | ✅ REST/GraphQL | After quote merge | Price recalculation |

#### **Quote Address Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_quote_address_save_before` | ✅ REST/GraphQL | Before address save | Address validation |
| `sales_quote_address_save_after` | ✅ REST/GraphQL | After address save | Shipping calculations |
| `sales_quote_address_collect_totals_before` | ✅ REST/GraphQL | Before totals collection | Custom totals |
| `sales_quote_address_collect_totals_after` | ✅ REST/GraphQL | After totals collection | Final adjustments |

#### **Quote Item Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_quote_item_save_before` | ✅ REST/GraphQL | Before quote item save | Item validation |
| `sales_quote_item_save_after` | ✅ REST/GraphQL | After quote item save | Price updates |
| `sales_quote_item_qty_set_after` | ✅ REST/GraphQL | After quantity change | Inventory checks |

---

## Payment Events

### 💳 **Payment Processing Events**

#### **Payment Method Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_payment_save_before` | ✅ REST/GraphQL | Before payment save | Payment validation |
| `sales_order_payment_save_after` | ✅ REST/GraphQL | After payment save | Transaction logging |
| `sales_order_payment_place_end` | ✅ REST/GraphQL | Payment placement complete | Order finalization |
| `sales_order_payment_pay` | ✅ REST/GraphQL | Payment capture | Gateway processing |
| `sales_order_payment_void` | ✅ REST/GraphQL | Payment void | Authorization cancellation |
| `sales_order_payment_cancel` | ✅ REST/GraphQL | Payment cancellation | Transaction reversal |
| `sales_order_payment_refund` | ✅ REST/GraphQL | Payment refund | Refund processing |

#### **Payment Transaction Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_order_payment_transaction_save_before` | ✅ REST/GraphQL | Before transaction save | Transaction validation |
| `sales_order_payment_transaction_save_after` | ✅ REST/GraphQL | After transaction save | Transaction logging |

---

## Shipping Events

### 🚚 **Shipping & Fulfillment Events**

#### **Shipping Rate Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `shipping_rate_request_before` | ✅ REST/GraphQL | Before rate request | Custom rate logic |
| `shipping_rate_request_after` | ✅ REST/GraphQL | After rate request | Rate modification |

#### **Shipping Method Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `sales_quote_collect_totals_before` | ✅ REST/GraphQL | Before shipping calculation | Custom shipping rules |
| `sales_quote_collect_totals_after` | ✅ REST/GraphQL | After shipping calculation | Final adjustments |

---

## Inventory Events

### 📊 **Inventory Management Events**

#### **Stock Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `cataloginventory_stock_item_save_before` | ✅ REST/GraphQL | Before stock save | Stock validation |
| `cataloginventory_stock_item_save_after` | ✅ REST/GraphQL | After stock save | Reorder alerts |
| `catalog_product_stock_status_changed` | ✅ REST/GraphQL | Stock status change | Availability notifications |

#### **MSI (Multi-Source Inventory) Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `inventory_source_item_save_after` | ✅ REST/GraphQL | After source item save | Multi-warehouse sync |
| `inventory_reservation_placed` | ✅ REST/GraphQL | After reservation | Inventory allocation |
| `inventory_low_stock_item` | ✅ REST/GraphQL | Low stock detected | Reorder notifications |

---

## Admin Events

### ⚙️ **Admin Panel Events**

#### **Admin User Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `admin_user_authenticate_after` | ❌ Admin Only | Admin login | Security logging |
| `admin_user_save_before` | ✅ REST/GraphQL | Before admin user save | User validation |
| `admin_user_save_after` | ✅ REST/GraphQL | After admin user save | Audit logging |

#### **Admin Session Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `admin_session_user_login_success` | ❌ Admin Only | Successful admin login | Security monitoring |
| `admin_session_user_login_failed` | ❌ Admin Only | Failed admin login | Brute force protection |
| `admin_session_user_logout` | ❌ Admin Only | Admin logout | Session cleanup |

---

## System Events

### 🔧 **System-Level Events**

#### **Configuration Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `config_data_save_before` | ⚠️ Admin/API | Before config save | Config validation |
| `config_data_save_after` | ⚠️ Admin/API | After config save | Cache invalidation |
| `store_save_before` | ⚠️ Admin/API | Before store save | Store validation |
| `store_save_after` | ⚠️ Admin/API | After store save | Multi-store updates |

#### **Cache Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `application_clean_cache` | ⚠️ System | Cache cleaning | Custom cache cleanup |
| `clean_cache_by_tags` | ⚠️ System | Tagged cache cleaning | Selective cache clearing |

#### **Indexer Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `reindex_all_invalid` | ⚠️ System | All indexers invalid | Full reindex trigger |
| `clean_cache_after_reindex` | ⚠️ System | After reindex | Cache warming |

---

## Email Events

### 📧 **Email System Events**

#### **Order Email Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `email_order_set_template_vars_before` | ✅ REST/GraphQL | Before email template vars | Custom variables |
| `email_invoice_set_template_vars_before` | ✅ REST/GraphQL | Before invoice email vars | Invoice customization |
| `email_shipment_set_template_vars_before` | ✅ REST/GraphQL | Before shipment email vars | Tracking info |
| `email_creditmemo_set_template_vars_before` | ✅ REST/GraphQL | Before credit memo email | Refund details |

#### **Customer Email Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `customer_registration_is_allowed` | ✅ REST/GraphQL | Registration validation | Custom restrictions |
| `newsletter_subscriber_save_before` | ✅ REST/GraphQL | Before subscription save | Subscription validation |
| `newsletter_subscriber_save_after` | ✅ REST/GraphQL | After subscription save | Welcome emails |

---

## CMS Events

### 📝 **Content Management Events**

#### **CMS Page Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `cms_page_save_before` | ✅ REST/GraphQL | Before CMS page save | Content validation |
| `cms_page_save_after` | ✅ REST/GraphQL | After CMS page save | Cache invalidation |
| `cms_page_delete_before` | ✅ REST/GraphQL | Before CMS page deletion | Link checking |
| `cms_page_delete_after` | ✅ REST/GraphQL | After CMS page deletion | Cleanup |

#### **CMS Block Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `cms_block_save_before` | ✅ REST/GraphQL | Before CMS block save | Block validation |
| `cms_block_save_after` | ✅ REST/GraphQL | After CMS block save | Cache clearing |

---

## Search Events

### 🔍 **Search & Indexing Events**

#### **Search Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `catalogsearch_index_process_start` | ⚠️ System | Search reindex start | Custom indexing |
| `catalogsearch_index_process_complete` | ⚠️ System | Search reindex complete | Post-index processing |
| `prepare_catalog_product_collection_prices` | ✅ REST/GraphQL | Price preparation | Custom pricing |

---

## API-Specific Events

### 🔌 **REST API Events**

| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `webapi_rest_server_execute_request_before` | ✅ REST Only | Before REST request | API logging, auth |
| `webapi_rest_server_execute_request_after` | ✅ REST Only | After REST request | Response modification |
| `webapi_rest_server_response_created` | ✅ REST Only | REST response created | Response formatting |

### **GraphQL Events**
| Event Name | API Compatible | When Triggered | Use Cases |
|------------|----------------|----------------|-----------|
| `graphql_server_execute_request_before` | ✅ GraphQL Only | Before GraphQL request | Query validation |
| `graphql_server_execute_request_after` | ✅ GraphQL Only | After GraphQL request | Response caching |
| `graphql_schema_get_types_before` | ✅ GraphQL Only | Before schema build | Schema modification |

---

## Event Usage Examples

### **Customer Registration with API Detection**
```php
class CustomerRegistrationObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $customer = $observer->getEvent()->getCustomer();
        $request = $observer->getEvent()->getRequest();
        
        if ($this->isApiRequest($request)) {
            // Handle API registration
            $this->apiNotificationService->sendWelcomeEmail($customer);
        } else {
            // Handle frontend registration
            $this->frontendNotificationService->sendWelcomeEmail($customer);
        }
    }
    
    private function isApiRequest($request)
    {
        return $request && (
            $request->isApi() || 
            strpos($request->getPathInfo(), '/rest/') !== false ||
            strpos($request->getPathInfo(), '/graphql') !== false
        );
    }
}
```

### **Order Processing with Context Awareness**
```php
class OrderProcessingObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $order = $observer->getEvent()->getOrder();
        $context = $this->getRequestContext();
        
        switch ($context['type']) {
            case 'rest':
                $this->handleRestOrderPlacement($order, $context);
                break;
            case 'graphql':
                $this->handleGraphQLOrderPlacement($order, $context);
                break;
            case 'frontend':
                $this->handleFrontendOrderPlacement($order, $context);
                break;
            case 'admin':
                $this->handleAdminOrderPlacement($order, $context);
                break;
        }
    }
}
```

### **Product Update with API Differentiation**
```php
class ProductUpdateObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $product = $observer->getEvent()->getProduct();
        $request = $observer->getEvent()->getRequest();
        
        if ($this->isGraphQLRequest($request)) {
            // GraphQL-specific handling
            $this->graphqlProductProcessor->process($product);
        } elseif ($this->isRestRequest($request)) {
            // REST-specific handling
            $this->restProductProcessor->process($product);
        } else {
            // Standard handling
            $this->standardProductProcessor->process($product);
        }
    }
}
```

---

## Event Priority and Best Practices

### **Event Listener Priority**
```xml
<!-- events.xml -->
<event name="sales_order_place_after">
    <observer name="high_priority_observer" 
              instance="Vendor\Module\Observer\HighPriorityObserver" 
              sortOrder="10" />
    <observer name="low_priority_observer" 
              instance="Vendor\Module\Observer\LowPriorityObserver" 
              sortOrder="100" />
</event>
```

### **API Context Detection Best Practices**
```php
class ApiContextDetector
{
    public function getApiContext($request)
    {
        if (!$request) {
            return ['type' => 'unknown'];
        }
        
        $pathInfo = $request->getPathInfo();
        
        if (strpos($pathInfo, '/rest/') !== false) {
            return [
                'type' => 'rest',
                'version' => $this->extractApiVersion($pathInfo),
                'endpoint' => $this->extractEndpoint($pathInfo)
            ];
        }
        
        if (strpos($pathInfo, '/graphql') !== false) {
            return [
                'type' => 'graphql',
                'operation' => $this->extractGraphQLOperation($request)
            ];
        }
        
        if ($request->isAdmin()) {
            return ['type' => 'admin'];
        }
        
        return ['type' => 'frontend'];
    }
}
```

## Summary

### **Key Points:**
- **✅ API Compatible Events** work with both REST and GraphQL
- **❌ Frontend/Admin Only Events** don't trigger through APIs
- **⚠️ System Events** may trigger during API operations but aren't API-specific
- **Event context detection** helps differentiate between API and frontend requests
- **Proper event handling** ensures consistent behavior across all channels

### **Most Important API-Compatible Event Categories:**
1. **Customer Events** - Registration, login, data updates
2. **Order Events** - Order lifecycle, status changes
3. **Product Events** - CRUD operations, inventory updates
4. **Quote Events** - Cart operations, totals calculation
5. **Payment Events** - Payment processing, transactions

This comprehensive guide provides the foundation for understanding and implementing event-driven architecture in Magento 2 across all channels.