# Magento 2 Invoice Creation - Complete Guide

## Table of Contents
1. [Invoice Overview](#invoice-overview)
2. [When Invoices Are Created](#when-invoices-are-created)
3. [Automatic vs Manual Invoice Creation](#automatic-vs-manual-invoice-creation)
4. [Invoice Creation Methods](#invoice-creation-methods)
5. [Invoice States and Lifecycle](#invoice-states-and-lifecycle)
6. [Payment Method Impact](#payment-method-impact)
7. [Admin Panel Invoice Creation](#admin-panel-invoice-creation)
8. [Programmatic Invoice Creation](#programmatic-invoice-creation)
9. [API Invoice Creation](#api-invoice-creation)
10. [Invoice Types](#invoice-types)
11. [Configuration Settings](#configuration-settings)
12. [Best Practices](#best-practices)
13. [Troubleshooting](#troubleshooting)

---

## Invoice Overview

### What is an Invoice in Magento 2?
An **invoice** is a commercial document that:
- **Records the sale** of products/services
- **Requests payment** from the customer
- **Enables shipping** (invoiced items can be shipped)
- **Allows refunds** (credit memos require invoices)
- **Tracks payment capture** for online payments

### Invoice vs Order Relationship
```
Order (Intent to Purchase) → Invoice (Request for Payment) → Payment (Money Collection)
                          ↓                               ↓
                      Items can be shipped         Order completion
```

### Key Invoice Functions
- **Financial record** of the transaction
- **Legal document** for accounting purposes
- **Prerequisite** for shipping and refunds
- **Payment capture** trigger for some gateways

---

## When Invoices Are Created

### Automatic Invoice Creation Scenarios

#### 1. **Online Payment Methods (Immediate Capture)**
```php
// Payment methods that auto-create invoices
$autoInvoicePaymentMethods = [
    'paypal_standard',     // PayPal Standard
    'stripe_payments',     // Stripe (when set to capture)
    'braintree',          // Braintree (capture mode)
    'authorizenet_directpost', // Authorize.Net (capture)
    'cashondelivery'      // Cash on Delivery
];
```

**When**: Immediately after order placement and successful payment
**Trigger**: Payment capture success

#### 2. **Cash on Delivery (COD)**
**When**: Immediately after order placement
**Reason**: No online payment processing required
**Configuration**: Usually enabled by default for COD

#### 3. **Offline Payment Methods**
```php
// Offline methods that may auto-invoice
$offlinePaymentMethods = [
    'checkmo',           // Check/Money Order
    'banktransfer',      // Bank Transfer
    'purchaseorder'      // Purchase Order
];
```

**When**: Based on configuration settings
**Control**: Admin setting "Payment Action"

### Manual Invoice Creation Scenarios

#### 1. **Authorize-Only Payment Methods**
**When**: After admin manually captures payment
**Examples**: Credit cards set to "Authorize Only"
**Process**: Order → Manual Invoice → Payment Capture

#### 2. **Pending Payment Orders**
**When**: After payment confirmation
**Examples**: Bank transfers, checks
**Process**: Order → Payment Received → Manual Invoice

#### 3. **Partial Invoicing**
**When**: Admin wants to invoice specific items
**Use Cases**: Partial shipments, backordered items
**Process**: Select items → Create partial invoice

---

## Automatic vs Manual Invoice Creation

### Automatic Invoice Creation

#### Configuration Location
```
Stores → Configuration → Sales → Payment Methods → [Payment Method] → Payment Action
```

#### Payment Actions That Auto-Create Invoices
```php
// Payment action configurations
$paymentActions = [
    'authorize_capture' => 'Authorize and Capture (Auto-Invoice)',
    'authorize'        => 'Authorize Only (Manual Invoice)',
    'order'           => 'Order (Manual Invoice)'
];
```

#### Example Configuration
```xml
<!-- payment method configuration -->
<config>
    <default>
        <payment>
            <stripe_payments>
                <payment_action>authorize_capture</payment_action>
                <automatic_invoice>1</automatic_invoice>
            </stripe_payments>
        </payment>
    </default>
</config>
```

### Manual Invoice Creation Benefits
- **Better cash flow control**
- **Inventory management** (don't capture until ready to ship)
- **Fraud prevention** (review orders before capturing)
- **Partial fulfillment** handling

---

## Invoice Creation Methods

### 1. **Automatic Creation (Payment-Triggered)**
```php
// Triggered during order payment processing
class PaymentCapture
{
    public function capturePayment($order, $payment)
    {
        // Process payment
        $result = $this->gateway->capture($payment);
        
        if ($result->isSuccessful()) {
            // Auto-create invoice if configured
            if ($this->shouldAutoInvoice($payment->getMethod())) {
                $this->createInvoiceForOrder($order);
            }
        }
    }
    
    private function shouldAutoInvoice($paymentMethod)
    {
        return $this->scopeConfig->isSetFlag(
            "payment/{$paymentMethod}/automatic_invoice"
        );
    }
}
```

### 2. **Admin Panel Creation**
**Navigation**: Sales → Orders → Select Order → Invoice

### 3. **Programmatic Creation**
```php
use Magento\Sales\Model\Service\InvoiceService;
use Magento\Sales\Model\Order\Email\Sender\InvoiceSender;

$invoice = $this->invoiceService->prepareInvoice($order);
$invoice->register();
$order->save();
```

### 4. **API Creation**
**REST API**: `POST /rest/V1/order/{orderId}/invoice`
**GraphQL**: Custom mutation for invoice creation

---

## Invoice States and Lifecycle

### Invoice States
```php
// Invoice state constants
const STATE_OPEN = 1;       // Invoice created, not paid
const STATE_PAID = 2;       // Invoice paid/captured
const STATE_CANCELED = 3;   // Invoice canceled
```

### Invoice Lifecycle Flow
```
Order Placed → Invoice Created (STATE_OPEN) → Payment Captured → Invoice Paid (STATE_PAID)
                     ↓                                               ↓
              Can be canceled                               Can create credit memo
                     ↓                                               ↓
            Invoice Canceled (STATE_CANCELED)                    Refund process
```

### State Transitions
```php
public function getInvoiceStateTransitions()
{
    return [
        'new_order' => [
            'can_create_invoice' => true,
            'auto_invoice' => 'depends_on_payment_method'
        ],
        'invoice_created' => [
            'state' => Invoice::STATE_OPEN,
            'can_capture' => true,
            'can_cancel' => true
        ],
        'payment_captured' => [
            'state' => Invoice::STATE_PAID,
            'can_ship' => true,
            'can_refund' => true
        ]
    ];
}
```

---

## Payment Method Impact

### Payment Action Types

#### 1. **Authorize and Capture**
```php
// Immediate invoice creation
'payment_action' => 'authorize_capture'
```
- **Invoice**: Created automatically
- **Payment**: Captured immediately
- **Inventory**: Reserved immediately
- **Use Case**: Standard online payments

#### 2. **Authorize Only**
```php
// Manual invoice creation required
'payment_action' => 'authorize'
```
- **Invoice**: Manual creation required
- **Payment**: Authorized, captured later
- **Inventory**: Reserved after invoice
- **Use Case**: Fraud prevention, custom workflows

#### 3. **Order (No Authorization)**
```php
// No payment processing
'payment_action' => 'order'
```
- **Invoice**: Manual creation
- **Payment**: Processed offline
- **Inventory**: Managed manually
- **Use Case**: Offline payments, B2B orders

### Payment Method Examples

#### Credit Cards (Stripe)
```php
// Stripe configuration example
'stripe_payments' => [
    'payment_action' => 'authorize_capture', // Auto-invoice
    'automatic_invoice' => true,
    'can_capture' => true,
    'can_void' => true
]
```

#### PayPal Standard
```php
// PayPal auto-invoices on successful payment
'paypal_standard' => [
    'payment_action' => 'sale', // Auto-invoice
    'skip_order_review_step' => false
]
```

#### Check/Money Order
```php
// Manual invoice creation
'checkmo' => [
    'payment_action' => 'order', // Manual invoice
    'automatic_invoice' => false
]
```

---

## Admin Panel Invoice Creation

### Step-by-Step Process

#### 1. **Navigate to Order**
```
Admin Panel → Sales → Orders → Select Order
```

#### 2. **Create Invoice**
```
Order View → Invoice Button (top right)
```

#### 3. **Configure Invoice**
**Invoice Options:**
- **Items to Invoice**: Select specific products
- **Quantities**: Specify invoice quantities
- **Capture Online**: Capture payment immediately
- **Comments**: Add internal/customer comments
- **Email**: Send invoice email to customer

### Admin Interface Example
```html
<!-- Invoice creation form -->
<form id="invoice_form" action="{{invoice_save_url}}" method="post">
    <!-- Items selection -->
    <table class="data-table">
        <thead>
            <tr>
                <th>Product</th>
                <th>Price</th>
                <th>Qty Ordered</th>
                <th>Qty to Invoice</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>Sample Product</td>
                <td>$50.00</td>
                <td>2</td>
                <td><input type="number" name="invoice[items][1][qty]" value="2" max="2"/></td>
            </tr>
        </tbody>
    </table>
    
    <!-- Invoice options -->
    <div class="invoice-options">
        <input type="checkbox" name="invoice[capture_case]" value="online" checked/>
        <label>Capture Online</label>
        
        <input type="checkbox" name="invoice[send_email]" value="1" checked/>
        <label>Send Email Copy of Invoice</label>
        
        <textarea name="invoice[comment_text]" placeholder="Comments"></textarea>
    </div>
    
    <button type="submit" class="submit-invoice">Submit Invoice</button>
</form>
```

---

## Programmatic Invoice Creation

### Full Invoice Creation
```php
<?php

namespace Vendor\Module\Service;

use Magento\Sales\Api\OrderRepositoryInterface;
use Magento\Sales\Model\Service\InvoiceService;
use Magento\Sales\Model\Order\Email\Sender\InvoiceSender;
use Magento\Framework\DB\Transaction;

class InvoiceCreationService
{
    private $orderRepository;
    private $invoiceService;
    private $invoiceSender;
    private $transaction;

    public function __construct(
        OrderRepositoryInterface $orderRepository,
        InvoiceService $invoiceService,
        InvoiceSender $invoiceSender,
        Transaction $transaction
    ) {
        $this->orderRepository = $orderRepository;
        $this->invoiceService = $invoiceService;
        $this->invoiceSender = $invoiceSender;
        $this->transaction = $transaction;
    }

    /**
     * Create full invoice for order
     */
    public function createFullInvoice($orderId, $captureOnline = true, $sendEmail = true)
    {
        try {
            $order = $this->orderRepository->get($orderId);
            
            // Check if order can be invoiced
            if (!$order->canInvoice()) {
                throw new \Exception('Order cannot be invoiced');
            }
            
            // Prepare invoice
            $invoice = $this->invoiceService->prepareInvoice($order);
            
            if (!$invoice->getTotalQty()) {
                throw new \Exception('Cannot create invoice without products');
            }
            
            // Set capture mode
            if ($captureOnline) {
                $invoice->setRequestedCaptureCase(\Magento\Sales\Model\Order\Invoice::CAPTURE_ONLINE);
            } else {
                $invoice->setRequestedCaptureCase(\Magento\Sales\Model\Order\Invoice::CAPTURE_OFFLINE);
            }
            
            // Register invoice
            $invoice->register();
            
            // Add comment
            $invoice->addComment('Invoice created programmatically', false, false);
            
            // Save invoice and order
            $transactionSave = $this->transaction
                ->addObject($invoice)
                ->addObject($invoice->getOrder());
            $transactionSave->save();
            
            // Send email
            if ($sendEmail) {
                $this->invoiceSender->send($invoice);
            }
            
            return $invoice;
            
        } catch (\Exception $e) {
            throw new \Exception('Invoice creation failed: ' . $e->getMessage());
        }
    }

    /**
     * Create partial invoice
     */
    public function createPartialInvoice($orderId, $invoiceData, $captureOnline = true)
    {
        $order = $this->orderRepository->get($orderId);
        
        // Prepare invoice with specific items
        $invoice = $this->invoiceService->prepareInvoice($order, $invoiceData);
        
        if ($captureOnline) {
            $invoice->setRequestedCaptureCase(\Magento\Sales\Model\Order\Invoice::CAPTURE_ONLINE);
        } else {
            $invoice->setRequestedCaptureCase(\Magento\Sales\Model\Order\Invoice::CAPTURE_OFFLINE);
        }
        
        $invoice->register();
        
        // Save
        $transactionSave = $this->transaction
            ->addObject($invoice)
            ->addObject($invoice->getOrder());
        $transactionSave->save();
        
        return $invoice;
    }
}
```

### Usage Examples
```php
// Create full invoice with online capture
$invoice = $invoiceService->createFullInvoice(12345, true, true);

// Create partial invoice
$invoiceData = [
    1 => 2, // Item ID 1, quantity 2
    2 => 1  // Item ID 2, quantity 1
];
$partialInvoice = $invoiceService->createPartialInvoice(12345, $invoiceData, false);
```

---

## API Invoice Creation

### REST API Invoice Creation
```javascript
// Create invoice via REST API
const createInvoice = async (orderId, invoiceData) => {
    const response = await fetch(`/rest/V1/order/${orderId}/invoice`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${adminToken}`
        },
        body: JSON.stringify({
            capture: true, // Online capture
            items: invoiceData.items,
            comment: {
                comment: 'Invoice created via API',
                is_visible_on_front: false
            },
            notify: true // Send email
        })
    });
    
    return await response.json();
};

// Usage
const invoiceData = {
    items: [
        { order_item_id: 1, qty: 2 },
        { order_item_id: 2, qty: 1 }
    ]
};

createInvoice(12345, invoiceData);
```

### GraphQL Invoice Creation
```graphql
# Custom GraphQL mutation (requires implementation)
mutation createInvoice($orderId: Int!, $invoiceData: InvoiceInput!) {
    createInvoice(order_id: $orderId, invoice_data: $invoiceData) {
        invoice {
            id
            increment_id
            state
            grand_total
        }
        success
        message
    }
}
```

---

## Invoice Types

### 1. **Full Invoice**
```php
// Invoice entire order
$invoice = $this->invoiceService->prepareInvoice($order);
```
- **All items** in the order
- **Complete order total**
- **Single transaction**

### 2. **Partial Invoice**
```php
// Invoice specific items/quantities
$invoiceData = [
    'item_id_1' => 2, // Quantity to invoice
    'item_id_2' => 1
];
$invoice = $this->invoiceService->prepareInvoice($order, $invoiceData);
```
- **Selected items only**
- **Specific quantities**
- **Multiple invoices possible**

### 3. **Capture Invoice**
```php
// Online payment capture
$invoice->setRequestedCaptureCase(Invoice::CAPTURE_ONLINE);
```
- **Processes payment** immediately
- **Updates payment status**
- **Captures authorized amount**

### 4. **Non-Capture Invoice**
```php
// Offline/manual capture
$invoice->setRequestedCaptureCase(Invoice::CAPTURE_OFFLINE);
```
- **No payment processing**
- **Manual capture required**
- **Used for offline payments**

---

## Configuration Settings

### Global Invoice Settings
```
Stores → Configuration → Sales → Sales → Invoice and Packing Slip Design
```

#### Key Configuration Options
- **Logo**: Company logo on invoices
- **Address**: Company address information
- **Invoice Comments**: Default comments
- **Email Settings**: Invoice email templates

### Payment Method Invoice Settings
```php
// Payment method specific settings
<config>
    <default>
        <payment>
            <stripe_payments>
                <payment_action>authorize_capture</payment_action>
                <automatic_invoice>1</automatic_invoice>
                <invoice_email>1</invoice_email>
            </stripe_payments>
            <checkmo>
                <payment_action>order</payment_action>
                <automatic_invoice>0</automatic_invoice>
            </checkmo>
        </payment>
    </default>
</config>
```

### Order Status and Invoice Automation
```php
// Configure automatic invoice creation based on order status
public function configureAutoInvoice()
{
    return [
        'processing' => [
            'auto_invoice' => true,
            'capture_online' => true,
            'send_email' => true
        ],
        'pending_payment' => [
            'auto_invoice' => false,
            'wait_for_payment' => true
        ]
    ];
}
```

---

## Best Practices

### 1. **Invoice Creation Timing**
```php
public function determineInvoiceCreationTiming($order)
{
    $paymentMethod = $order->getPayment()->getMethod();
    
    // Immediate invoicing for secure payments
    $immediateInvoiceMethods = [
        'paypal_standard',
        'stripe_payments',
        'cashondelivery'
    ];
    
    // Delayed invoicing for review
    $delayedInvoiceMethods = [
        'checkmo',
        'banktransfer',
        'purchaseorder'
    ];
    
    if (in_array($paymentMethod, $immediateInvoiceMethods)) {
        return 'immediate';
    } elseif (in_array($paymentMethod, $delayedInvoiceMethods)) {
        return 'manual_review';
    }
    
    return 'default_flow';
}
```

### 2. **Partial Invoice Strategy**
```php
public function createPartialInvoiceForShipment($order, $shipmentItems)
{
    // Only invoice items that are being shipped
    $invoiceData = [];
    
    foreach ($shipmentItems as $item) {
        $invoiceData[$item->getOrderItemId()] = $item->getQty();
    }
    
    // Create invoice for shipped items only
    $invoice = $this->invoiceService->prepareInvoice($order, $invoiceData);
    $invoice->setRequestedCaptureCase(Invoice::CAPTURE_ONLINE);
    $invoice->register();
    
    return $invoice;
}
```

### 3. **Error Handling**
```php
public function createInvoiceWithErrorHandling($orderId)
{
    try {
        $order = $this->orderRepository->get($orderId);
        
        // Validate order state
        if (!$order->canInvoice()) {
            throw new \Exception("Order {$orderId} cannot be invoiced. Current state: " . $order->getState());
        }
        
        // Check payment method capability
        if (!$order->getPayment()->getMethodInstance()->canCapture()) {
            throw new \Exception("Payment method does not support capture");
        }
        
        $invoice = $this->createFullInvoice($orderId);
        
        // Log success
        $this->logger->info("Invoice created successfully", [
            'order_id' => $orderId,
            'invoice_id' => $invoice->getId(),
            'amount' => $invoice->getGrandTotal()
        ]);
        
        return $invoice;
        
    } catch (\Exception $e) {
        $this->logger->error("Invoice creation failed", [
            'order_id' => $orderId,
            'error' => $e->getMessage()
        ]);
        
        throw $e;
    }
}
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **"Order cannot be invoiced" Error**
**Causes:**
- Order already fully invoiced
- Order canceled or on hold
- Payment not authorized

**Solution:**
```php
public function diagnoseInvoiceIssue($order)
{
    $issues = [];
    
    if ($order->getState() === Order::STATE_CANCELED) {
        $issues[] = 'Order is canceled';
    }
    
    if ($order->getState() === Order::STATE_HOLDED) {
        $issues[] = 'Order is on hold';
    }
    
    if ($order->getTotalInvoiced() >= $order->getGrandTotal()) {
        $issues[] = 'Order already fully invoiced';
    }
    
    if (!$order->getPayment()->canCapture()) {
        $issues[] = 'Payment cannot be captured';
    }
    
    return $issues;
}
```

#### 2. **Invoice Not Auto-Creating**
**Check:**
- Payment method configuration
- Payment action setting
- Automatic invoice setting

**Debug:**
```php
public function debugAutoInvoice($paymentMethod)
{
    $config = $this->scopeConfig->getValue("payment/{$paymentMethod}");
    
    return [
        'payment_action' => $config['payment_action'] ?? 'not_set',
        'automatic_invoice' => $config['automatic_invoice'] ?? 'not_set',
        'can_capture' => $this->paymentMethodInstance->canCapture()
    ];
}
```

#### 3. **Partial Invoice Calculation Issues**
**Common Problems:**
- Tax calculations
- Discount distributions
- Shipping allocations

**Solution:**
```php
public function validatePartialInvoiceAmounts($order, $invoiceData)
{
    $calculatedTotal = 0;
    
    foreach ($invoiceData as $itemId => $qty) {
        $orderItem = $order->getItemById($itemId);
        $itemTotal = ($orderItem->getPrice() * $qty);
        $calculatedTotal += $itemTotal;
    }
    
    // Add proportional tax and shipping
    $proportionalTax = $this->calculateProportionalTax($order, $invoiceData);
    $proportionalShipping = $this->calculateProportionalShipping($order, $invoiceData);
    
    $expectedTotal = $calculatedTotal + $proportionalTax + $proportionalShipping;
    
    return $expectedTotal;
}
```

---

## Summary

**Invoice Creation in Magento 2:**

### **When Invoices Are Created:**
1. **Automatically** - For "Authorize and Capture" payment methods
2. **Manually** - For "Authorize Only" or offline payment methods
3. **On-demand** - When admin creates them for specific business needs

### **Key Triggers:**
- **Payment capture success** (automatic)
- **Admin action** (manual)
- **API calls** (programmatic)
- **Order status changes** (configured)

### **Invoice Requirements:**
- Order must be in valid state (not canceled/closed)
- Payment must be authorized (for online methods)
- Items must be available for invoicing

### **Best Practices:**
- Configure payment actions appropriately
- Use partial invoices for complex fulfillment
- Implement proper error handling
- Monitor invoice creation logs
- Send email notifications to customers

**Invoice creation is crucial for order fulfillment** as it enables shipping, refunds, and proper financial record-keeping in Magento 2's order lifecycle.