# Comprehensive Guide: Order Status and State in Magento 2

## Table of Contents
1. [Fundamental Concepts](#fundamental-concepts)
2. [Order States vs Order Statuses](#order-states-vs-order-statuses)
3. [Default Order States](#default-order-states)
4. [Default Order Statuses](#default-order-statuses)
5. [Order Lifecycle Flow](#order-lifecycle-flow)
6. [State-Status Relationships](#state-status-relationships)
7. [Order State Transitions](#order-state-transitions)
8. [Programming with Order States](#programming-with-order-states)
9. [Custom Order Statuses](#custom-order-statuses)
10. [Configuration and Admin](#configuration-and-admin)
11. [Events and Observers](#events-and-observers)
12. [Best Practices](#best-practices)
13. [Troubleshooting](#troubleshooting)

---

## Fundamental Concepts

### What are Order States?
**Order States** are **internal system identifiers** that represent the high-level workflow stage of an order in Magento 2. They are:
- **Fixed and predefined** by Magento core
- **Cannot be customized** or added to
- **Used for business logic** and system processing
- **Not directly visible** to customers in most cases

### What are Order Statuses?
**Order Statuses** are **human-readable labels** that provide meaningful information about an order's current condition. They are:
- **Customizable** and can be created by admins
- **Customer-facing** and appear in emails, order history, etc.
- **Mapped to order states** (many-to-one relationship)
- **Used for display purposes** and customer communication

### Key Relationship
```
Multiple Order Statuses → Map to → Single Order State
```

---

## Order States vs Order Statuses

| Aspect | Order States | Order Statuses |
|--------|-------------|----------------|
| **Purpose** | Internal workflow control | Customer communication |
| **Customization** | Fixed (cannot modify) | Fully customizable |
| **Visibility** | Backend/system use | Customer-facing |
| **Quantity** | 7 predefined states | Unlimited custom statuses |
| **Function** | Business logic triggers | Display and messaging |
| **Examples** | `processing`, `complete` | `pending_payment`, `shipped` |

---

## Default Order States

Magento 2 has **7 predefined order states** that cannot be modified:

### 1. **`new`** - New Order
- **When**: Order just placed, payment pending
- **Characteristics**: Initial state for most orders
- **Typical Actions**: Payment processing, inventory reservation

### 2. **`pending_payment`** - Pending Payment  
- **When**: Order placed but payment not yet captured
- **Characteristics**: Awaiting payment confirmation
- **Typical Actions**: Payment gateway processing

### 3. **`processing`** - Processing
- **When**: Payment captured, order being fulfilled
- **Characteristics**: Active fulfillment state
- **Typical Actions**: Picking, packing, shipping preparation

### 4. **`complete`** - Complete
- **When**: Order fully shipped and delivered
- **Characteristics**: Final successful state
- **Typical Actions**: Customer notifications, analytics

### 5. **`closed`** - Closed
- **When**: Order closed manually or due to issues
- **Characteristics**: Manually terminated order
- **Typical Actions**: Cleanup, reporting

### 6. **`canceled`** - Canceled
- **When**: Order canceled before fulfillment
- **Characteristics**: Terminated before completion
- **Typical Actions**: Inventory return, refund processing

### 7. **`holded`** - On Hold
- **When**: Order temporarily suspended
- **Characteristics**: Temporary suspension state
- **Typical Actions**: Manual review, fraud check

---

## Default Order Statuses

Magento 2 ships with several **predefined order statuses**:

### Payment-Related Statuses
- **`pending`** → State: `new`
- **`pending_payment`** → State: `pending_payment`
- **`payment_review`** → State: `payment_review`
- **`suspected_fraud`** → State: `payment_review`

### Processing Statuses
- **`processing`** → State: `processing`

### Fulfillment Statuses
- **`shipped`** → State: `complete`
- **`complete`** → State: `complete`

### Termination Statuses
- **`canceled`** → State: `canceled`
- **`closed`** → State: `closed`
- **`holded`** → State: `holded`

---

## Order Lifecycle Flow

### Typical Order Journey
```
1. ORDER PLACEMENT
   Status: pending → State: new
   ↓
2. PAYMENT PROCESSING
   Status: pending_payment → State: pending_payment
   ↓
3. PAYMENT CAPTURED
   Status: processing → State: processing
   ↓
4. ORDER SHIPPED
   Status: shipped → State: complete
   ↓
5. ORDER DELIVERED
   Status: complete → State: complete
```

### Alternative Paths
```
CANCELLATION PATH:
Any State → Status: canceled → State: canceled

HOLD PATH:
Any State → Status: holded → State: holded

FRAUD REVIEW PATH:
pending_payment → Status: payment_review → State: payment_review
```

---

## State-Status Relationships

### One-to-Many Mapping
Each **state** can have **multiple statuses** assigned to it:

```php
// Example state-status mappings
'new' => [
    'pending',
    'pending_payment'
],
'processing' => [
    'processing',
    'shipped'  // Custom status
],
'complete' => [
    'complete',
    'shipped'
]
```

### Configuration Location
**Admin Panel**: Stores → Settings → Order Status
- View state-status assignments
- Create custom statuses
- Assign statuses to states

---

## Order State Transitions

### Valid State Transitions
Not all state changes are permitted. Magento enforces business rules:

```php
// Valid transitions (examples)
new → processing ✅
new → canceled ✅
processing → complete ✅
processing → canceled ✅
complete → closed ✅

// Invalid transitions (examples)
complete → new ❌
canceled → processing ❌
closed → complete ❌
```

### Transition Triggers
State changes occur through:
- **Payment capture**
- **Shipment creation**
- **Manual admin actions**
- **Automated processes**
- **Custom business logic**

---

## Programming with Order States

### Getting Order State/Status
```php
use Magento\Sales\Api\OrderRepositoryInterface;

class OrderService
{
    private $orderRepository;

    public function __construct(OrderRepositoryInterface $orderRepository)
    {
        $this->orderRepository = $orderRepository;
    }

    public function getOrderInfo($orderId)
    {
        $order = $this->orderRepository->get($orderId);
        
        return [
            'state' => $order->getState(),
            'status' => $order->getStatus(),
            'state_label' => $order->getStateLabel(),
            'status_label' => $order->getStatusLabel()
        ];
    }
}
```

### Setting Order State/Status
```php
use Magento\Sales\Model\Order;

public function updateOrderStatus($order, $status, $comment = '')
{
    // Set status (automatically updates state if needed)
    $order->setStatus($status);
    
    // Add comment to order history
    if ($comment) {
        $order->addCommentToStatusHistory($comment);
    }
    
    // Save the order
    $order->save();
}
```

### Checking Order State
```php
public function isOrderProcessable($order)
{
    $processableStates = [
        Order::STATE_NEW,
        Order::STATE_PENDING_PAYMENT,
        Order::STATE_PROCESSING
    ];
    
    return in_array($order->getState(), $processableStates);
}
```

### State Constants
```php
// Magento\Sales\Model\Order constants
Order::STATE_NEW            // 'new'
Order::STATE_PENDING_PAYMENT // 'pending_payment'
Order::STATE_PROCESSING     // 'processing'
Order::STATE_COMPLETE       // 'complete'
Order::STATE_CLOSED         // 'closed'
Order::STATE_CANCELED       // 'canceled'
Order::STATE_HOLDED         // 'holded'
```

---

## Custom Order Statuses

### Creating Custom Status via Admin
1. **Navigate**: Stores → Order Status
2. **Click**: "Create New Status"
3. **Configure**:
   - Status Code (internal identifier)
   - Status Label (display name)
4. **Assign to State**: Stores → Order Status → Assign Status to State

### Creating Custom Status Programmatically
```php
use Magento\Sales\Model\ResourceModel\Order\Status\CollectionFactory;
use Magento\Sales\Model\Order\StatusFactory;

class CustomStatusCreator
{
    private $statusFactory;
    private $statusCollectionFactory;

    public function createCustomStatus()
    {
        $statusCode = 'awaiting_shipment';
        $statusLabel = 'Awaiting Shipment';
        
        // Create status
        $status = $this->statusFactory->create();
        $status->setData([
            'status' => $statusCode,
            'label' => $statusLabel
        ]);
        $status->save();
        
        // Assign to state
        $status->assignState(
            \Magento\Sales\Model\Order::STATE_PROCESSING,
            false, // not default for state
            true   // visible on frontend
        );
    }
}
```

### Data Patch for Custom Status
```php
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Sales\Model\Order\StatusFactory;

class AddCustomOrderStatus implements DataPatchInterface
{
    private $statusFactory;

    public function apply()
    {
        $statusCode = 'preparing_shipment';
        $statusLabel = 'Preparing for Shipment';
        
        $status = $this->statusFactory->create();
        $status->setData([
            'status' => $statusCode,
            'label' => $statusLabel
        ]);
        $status->save();
        
        $status->assignState(
            \Magento\Sales\Model\Order::STATE_PROCESSING,
            false
        );
    }
    
    public static function getDependencies(): array
    {
        return [];
    }
    
    public function getAliases(): array
    {
        return [];
    }
}
```

---

## Configuration and Admin

### Order Status Grid (Admin)
**Location**: Sales → Orders
- **State Column**: Shows current order state
- **Status Column**: Shows current order status
- **Filter**: Can filter by both state and status

### Order Status Configuration
**Location**: Stores → Settings → Order Status

#### Available Actions:
- **Create New Status**: Add custom statuses
- **Assign Status to State**: Map statuses to states
- **Edit Status**: Modify existing status labels
- **Unassign Status**: Remove state-status mapping

### Order Status Assignment
```php
// Configuration structure
'state_code' => [
    'statuses' => [
        'status_code_1' => true,  // default status for state
        'status_code_2' => false, // additional status for state
    ]
]
```

---

## Events and Observers

### Key Order Status Events
```php
// Dispatched when order status changes
'sales_order_save_before'
'sales_order_save_after'

// Specific status change events
'sales_order_state_change_before'
'sales_order_state_change_after'

// Payment-related events
'sales_order_payment_capture'
'sales_order_payment_refund'

// Shipment events
'sales_order_shipment_save_after'
'sales_order_shipment_track_save_after'
```

### Observer Example
```php
use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;

class OrderStatusChangeObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $order = $observer->getEvent()->getOrder();
        
        if ($order->getState() === Order::STATE_COMPLETE) {
            // Order completed - trigger additional actions
            $this->sendCompletionEmail($order);
            $this->updateLoyaltyPoints($order);
            $this->syncWithExternalSystem($order);
        }
    }
}
```

### Event Registration (events.xml)
```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Event/etc/events.xsd">
    <event name="sales_order_save_after">
        <observer name="custom_order_status_observer" 
                  instance="Vendor\Module\Observer\OrderStatusChangeObserver" />
    </event>
</config>
```

---

## Best Practices

### 1. **Use States for Business Logic**
```php
// ✅ Good - Use state for business decisions
if ($order->getState() === Order::STATE_PROCESSING) {
    $this->allowShipment($order);
}

// ❌ Bad - Don't rely on specific statuses for logic
if ($order->getStatus() === 'pending_shipment') {
    $this->allowShipment($order);
}
```

### 2. **Use Statuses for Display**
```php
// ✅ Good - Use status for customer communication
$customerMessage = "Your order is currently: " . $order->getStatusLabel();

// Email template: "Your order #12345 is {{var order.status_label}}"
```

### 3. **Validate State Transitions**
```php
public function canTransitionTo($currentState, $newState)
{
    $validTransitions = [
        Order::STATE_NEW => [Order::STATE_PROCESSING, Order::STATE_CANCELED],
        Order::STATE_PROCESSING => [Order::STATE_COMPLETE, Order::STATE_CANCELED],
        // ... more transitions
    ];
    
    return in_array($newState, $validTransitions[$currentState] ?? []);
}
```

### 4. **Custom Status Naming**
```php
// ✅ Good naming conventions
'awaiting_payment'     // Clear, descriptive
'preparing_shipment'   // Indicates action in progress
'quality_check'        // Specific business process

// ❌ Poor naming
'status_1'            // Non-descriptive
'temp'                // Unclear purpose
'custom'              // Too generic
```

### 5. **Documentation and Comments**
```php
/**
 * Custom status: 'awaiting_pickup'
 * State: processing
 * Purpose: Order packed and ready for customer pickup
 * Triggers: When warehouse confirms order is ready
 * Next Status: 'picked_up' or 'canceled'
 */
```

---

## Troubleshooting

### Common Issues

#### 1. **Status Not Appearing in Admin**
**Cause**: Status not assigned to any state
**Solution**:
```php
// Assign status to state
$status->assignState(Order::STATE_PROCESSING, false);
```

#### 2. **Cannot Change Order Status**
**Cause**: Invalid state transition or order locked
**Check**:
```php
if ($order->canEdit()) {
    // Order can be modified
} else {
    // Order is locked (invoiced, shipped, etc.)
}
```

#### 3. **Custom Status Not Saving**
**Cause**: Database constraint or duplicate code
**Debug**:
```php
try {
    $status->save();
} catch (\Exception $e) {
    echo "Error: " . $e->getMessage();
}
```

#### 4. **Status Email Not Sending**
**Cause**: Email template not configured for custom status
**Solution**: Configure in Admin → Marketing → Email Templates

### Debugging Order States
```php
public function debugOrderState($orderId)
{
    $order = $this->orderRepository->get($orderId);
    
    return [
        'order_id' => $order->getId(),
        'increment_id' => $order->getIncrementId(),
        'current_state' => $order->getState(),
        'current_status' => $order->getStatus(),
        'can_cancel' => $order->canCancel(),
        'can_ship' => $order->canShip(),
        'can_invoice' => $order->canInvoice(),
        'has_shipments' => $order->hasShipments(),
        'has_invoices' => $order->hasInvoices(),
        'total_paid' => $order->getTotalPaid(),
        'status_history' => $this->getStatusHistory($order)
    ];
}
```

---

## Advanced Topics

### Multi-Store Status Configuration
Different stores can have different status labels:
```php
// Store-specific status labels
$status->setStoreLabels([
    0 => 'Processing',      // Default store view
    1 => 'En cours',        // French store
    2 => 'Verarbeitung'     // German store
]);
```

### API Integration
```php
// REST API endpoints for order status
GET /rest/V1/orders/{id}
PUT /rest/V1/orders/{id}

// GraphQL queries
{
  customer {
    orders {
      items {
        number
        status
        state
      }
    }
  }
}
```

### Performance Considerations
```php
// Use collections for bulk operations
$orders = $this->orderCollectionFactory->create()
    ->addFieldToFilter('state', Order::STATE_PROCESSING)
    ->addFieldToFilter('created_at', ['lt' => $cutoffDate]);

foreach ($orders as $order) {
    $order->setStatus('old_processing');
}

// Bulk save
$orders->save();
```

---

## Summary

Understanding Magento 2's order status and state system is crucial for:
- **Effective order management workflows**
- **Proper customer communication**
- **Robust business logic implementation**
- **Successful customization and extension development**

**Key Takeaways:**
- **States** = Internal system workflow (fixed)
- **Statuses** = Customer-facing labels (customizable)
- **Many statuses** can map to **one state**
- **Use states for logic**, **statuses for display**
- **Always validate** state transitions
- **Document custom statuses** thoroughly

This system provides the flexibility to create meaningful customer experiences while maintaining robust internal business processes.