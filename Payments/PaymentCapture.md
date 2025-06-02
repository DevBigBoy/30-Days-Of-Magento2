# Payment Capture in Magento 2 - Complete Guide

## Table of Contents
1. [What is Payment Capture](#what-is-payment-capture)
2. [Authorization vs Capture](#authorization-vs-capture)
3. [Payment Capture Process](#payment-capture-process)
4. [Capture Timing Scenarios](#capture-timing-scenarios)
5. [Payment Actions in Magento](#payment-actions-in-magento)
6. [Automatic vs Manual Capture](#automatic-vs-manual-capture)
7. [Technical Implementation](#technical-implementation)
8. [Payment Method Examples](#payment-method-examples)
9. [Capture States and Lifecycle](#capture-states-and-lifecycle)
10. [Configuration Settings](#configuration-settings)
11. [Best Practices](#best-practices)
12. [Troubleshooting](#troubleshooting)

---

## What is Payment Capture

### Definition
**Payment Capture** is the process of actually collecting money from a customer's payment method (credit card, bank account, etc.) after the payment has been authorized.

### Key Concepts
```
Authorization = "Reserve the money" (Hold funds)
    ↓
Capture = "Take the money" (Transfer funds to merchant)
```

### Real-World Analogy
Think of it like a hotel reservation:
- **Authorization** = Hotel places a hold on your credit card
- **Capture** = Hotel actually charges your card when you check out

### Why Two Steps?
1. **Fraud Prevention** - Review orders before taking money
2. **Inventory Management** - Don't charge until items are available
3. **Business Flexibility** - Capture when ready to fulfill
4. **Legal Compliance** - Some industries require authorization-only initially

---

## Authorization vs Capture

### Payment Authorization
```
Customer → Payment Gateway → Bank → "Is $100 available?" → YES → Hold $100
```

**What Happens:**
- Verifies payment method is valid
- Checks sufficient funds/credit available
- Places a temporary hold on funds
- **No money actually moves**
- Hold typically expires in 7-30 days

**Purpose:**
- Validate payment method
- Reserve funds for the transaction
- Prevent overspending by customer
- Allow merchant time to fulfill order

### Payment Capture
```
Merchant → Payment Gateway → Bank → "Transfer the held $100" → Money moves to merchant
```

**What Happens:**
- Converts the authorization into actual payment
- **Money actually transfers** from customer to merchant
- Reduces customer's available balance/credit
- Completes the financial transaction

**Purpose:**
- Collect payment for delivered goods/services
- Complete the sales transaction
- Settle funds with the merchant

### Timeline Comparison
```
Authorization-Only Flow:
Day 1: Order placed → Authorization → Hold funds
Day 3: Items shipped → Manual capture → Money transferred
Day 7: Customer receives items

Authorize-and-Capture Flow:
Day 1: Order placed → Authorization + Capture → Money transferred immediately
Day 3: Items shipped
Day 7: Customer receives items
```

---

## Payment Capture Process

### Step-by-Step Capture Flow

#### 1. **Initial Setup**
```php
// Order is placed with authorized payment
$order = $this->orderFactory->create();
$payment = $order->getPayment();
$payment->setAmountAuthorized($orderTotal);
$payment->setBaseAmountAuthorized($orderTotal);
```

#### 2. **Capture Initiation**
```php
// Trigger capture (manual or automatic)
$invoice = $this->invoiceService->prepareInvoice($order);
$invoice->setRequestedCaptureCase(Invoice::CAPTURE_ONLINE);
$invoice->register(); // This triggers capture
```

#### 3. **Gateway Processing**
```php
// Payment method processes capture
public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
{
    $authorizationTransaction = $payment->getAuthorizationTransaction();
    
    $captureResult = $this->gateway->capture([
        'transaction_id' => $authorizationTransaction->getTxnId(),
        'amount' => $amount,
        'currency' => $payment->getOrder()->getOrderCurrencyCode()
    ]);
    
    if ($captureResult->isSuccessful()) {
        $payment->setTransactionId($captureResult->getTransactionId());
        $payment->setIsTransactionClosed(false);
    }
    
    return $this;
}
```

#### 4. **Settlement**
```php
// Update payment and order status
$payment->setAmountPaid($amount);
$payment->setBaseAmountPaid($amount);
$order->setTotalPaid($order->getTotalPaid() + $amount);
$order->setBaseTotalPaid($order->getBaseTotalPaid() + $amount);
```

---

## Capture Timing Scenarios

### 1. **Immediate Capture (Authorize and Capture)**
```
Order Placed → Authorization + Capture (simultaneously) → Money transferred
```

**When Used:**
- Digital products (immediate delivery)
- Low-risk transactions
- Standard e-commerce flow
- Cash on delivery

**Benefits:**
- Simplified workflow
- Guaranteed payment
- Reduced authorization expiry risk

**Risks:**
- Money taken before shipping
- Harder to handle cancellations
- Customer service issues if shipping delayed

### 2. **Delayed Capture (Authorize Only)**
```
Order Placed → Authorization → Review/Fulfill → Manual Capture → Money transferred
```

**When Used:**
- High-value orders
- Custom/made-to-order products
- Dropshipping scenarios
- Fraud prevention workflows

**Benefits:**
- Better fraud protection
- Flexible fulfillment timing
- Easier cancellation handling
- Cash flow control

**Risks:**
- Authorization may expire
- Manual process required
- Potential capture failures

### 3. **Partial Capture**
```
Order Placed → Full Authorization → Partial Capture 1 → Partial Capture 2 → Complete
```

**When Used:**
- Partial shipments
- Backordered items
- Subscription services
- Split deliveries

**Example:**
```php
// Capture part of the authorized amount
$partialAmount = 150.00; // Out of $300 authorized
$invoice = $this->createPartialInvoice($order, $partialAmount);
$invoice->capture(); // Captures $150, leaves $150 authorized
```

---

## Payment Actions in Magento

### Configuration Options
```
Stores → Configuration → Sales → Payment Methods → [Method] → Payment Action
```

### Available Payment Actions

#### 1. **Authorize and Capture**
```php
'payment_action' => 'authorize_capture'
```
- **Immediate capture** upon order placement
- **Money transferred** right away
- **Invoice created** automatically
- **Standard flow** for most e-commerce

#### 2. **Authorize Only**
```php
'payment_action' => 'authorize'
```
- **Authorization only** upon order placement
- **Manual capture** required later
- **Invoice creation** triggers capture
- **Better control** over payment timing

#### 3. **Order (No Authorization)**
```php
'payment_action' => 'order'
```
- **No payment processing** initially
- **Offline payment** methods
- **Manual payment** handling required
- **Used for** checks, bank transfers, etc.

### Payment Action Impact
```php
public function getPaymentActionBehavior($paymentAction)
{
    return [
        'authorize_capture' => [
            'authorization' => 'immediate',
            'capture' => 'immediate',
            'invoice' => 'auto_created',
            'money_transfer' => 'immediate'
        ],
        'authorize' => [
            'authorization' => 'immediate',
            'capture' => 'manual_required',
            'invoice' => 'manual_creation',
            'money_transfer' => 'delayed'
        ],
        'order' => [
            'authorization' => 'none',
            'capture' => 'offline',
            'invoice' => 'manual_creation',
            'money_transfer' => 'external'
        ]
    ];
}
```

---

## Automatic vs Manual Capture

### Automatic Capture
```php
// Configured in payment method settings
class AutoCapturePayment extends AbstractMethod
{
    protected $_paymentAction = 'authorize_capture';
    
    public function initialize($paymentAction, $stateObject)
    {
        if ($paymentAction === 'authorize_capture') {
            // Capture happens automatically during order processing
            $payment = $this->getInfoInstance();
            $this->capture($payment, $payment->getOrder()->getGrandTotal());
        }
        
        return $this;
    }
}
```

**Triggers:**
- Order placement (for authorize_capture methods)
- Invoice creation (for authorize_only methods with auto-capture)
- Successful payment gateway response

### Manual Capture
```php
// Admin-initiated capture
public function manualCapture($orderId, $amount = null)
{
    $order = $this->orderRepository->get($orderId);
    $payment = $order->getPayment();
    
    // Determine capture amount
    $captureAmount = $amount ?: $order->getGrandTotal();
    
    // Create invoice with capture
    $invoice = $this->invoiceService->prepareInvoice($order);
    $invoice->setRequestedCaptureCase(Invoice::CAPTURE_ONLINE);
    $invoice->register();
    
    return $invoice;
}
```

**Triggers:**
- Admin creates invoice with "Capture Online"
- API calls for invoice creation
- Scheduled jobs/cron processes
- Custom business logic triggers

---

## Technical Implementation

### Payment Method Capture Implementation
```php
<?php

namespace Vendor\Payment\Model\Method;

use Magento\Payment\Model\Method\AbstractMethod;

class CustomPaymentMethod extends AbstractMethod
{
    protected $_code = 'custom_payment';
    protected $_canCapture = true;
    protected $_canCapturePartial = true;
    
    /**
     * Capture payment through gateway
     */
    public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        $order = $payment->getOrder();
        $authTransactionId = $payment->getParentTransactionId();
        
        try {
            // Prepare capture request
            $captureRequest = [
                'transaction_id' => $authTransactionId,
                'amount' => $amount,
                'currency' => $order->getOrderCurrencyCode(),
                'order_id' => $order->getIncrementId()
            ];
            
            // Process capture with payment gateway
            $result = $this->gateway->capturePayment($captureRequest);
            
            if ($result->isSuccessful()) {
                // Update payment with capture details
                $payment->setTransactionId($result->getCaptureTransactionId());
                $payment->setIsTransactionClosed(false);
                $payment->setShouldCloseParentTransaction(false);
                
                // Add transaction details
                $payment->setTransactionAdditionalInfo([
                    'capture_id' => $result->getCaptureTransactionId(),
                    'capture_amount' => $amount,
                    'capture_date' => date('Y-m-d H:i:s'),
                    'gateway_response' => $result->getGatewayResponse()
                ]);
                
            } else {
                throw new \Exception('Capture failed: ' . $result->getErrorMessage());
            }
            
        } catch (\Exception $e) {
            $this->logger->error('Payment capture failed', [
                'order_id' => $order->getId(),
                'amount' => $amount,
                'error' => $e->getMessage()
            ]);
            
            throw new \Magento\Framework\Exception\LocalizedException(
                __('Payment capture failed: %1', $e->getMessage())
            );
        }
        
        return $this;
    }
    
    /**
     * Check if payment can be captured
     */
    public function canCapture()
    {
        $payment = $this->getInfoInstance();
        
        // Can capture if:
        // 1. Payment method supports capture
        // 2. Payment is authorized but not captured
        // 3. Authorization hasn't expired
        
        return $this->_canCapture && 
               $payment->getAmountAuthorized() > $payment->getAmountPaid() &&
               $this->isAuthorizationValid($payment);
    }
    
    /**
     * Validate authorization is still valid
     */
    private function isAuthorizationValid($payment)
    {
        $authTransaction = $payment->getAuthorizationTransaction();
        
        if (!$authTransaction) {
            return false;
        }
        
        // Check if authorization has expired (typically 7-30 days)
        $authDate = new \DateTime($authTransaction->getCreatedAt());
        $expiryDays = $this->getConfigData('authorization_expiry_days') ?: 7;
        $expiryDate = $authDate->add(new \DateInterval("P{$expiryDays}D"));
        
        return new \DateTime() < $expiryDate;
    }
}
```

### Capture Event Handling
```php
<?php

namespace Vendor\Module\Observer;

use Magento\Framework\Event\ObserverInterface;
use Magento\Framework\Event\Observer;

class PaymentCaptureObserver implements ObserverInterface
{
    public function execute(Observer $observer)
    {
        $invoice = $observer->getEvent()->getInvoice();
        $payment = $invoice->getOrder()->getPayment();
        
        // Log capture for audit trail
        $this->logger->info('Payment captured', [
            'order_id' => $invoice->getOrder()->getId(),
            'invoice_id' => $invoice->getId(),
            'amount_captured' => $invoice->getGrandTotal(),
            'payment_method' => $payment->getMethod(),
            'transaction_id' => $payment->getLastTransId()
        ]);
        
        // Trigger additional business logic
        $this->updateInventoryOnCapture($invoice);
        $this->sendCaptureNotification($invoice);
        $this->updateCustomerLoyaltyPoints($invoice);
    }
}
```

---

## Payment Method Examples

### Stripe Payment Capture
```php
public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
{
    $chargeId = $payment->getParentTransactionId();
    
    try {
        \Stripe\Stripe::setApiKey($this->getPrivateKey());
        
        // Capture the payment
        $charge = \Stripe\PaymentIntent::retrieve($chargeId);
        $charge = $charge->capture([
            'amount_to_capture' => $amount * 100 // Convert to cents
        ]);
        
        $payment->setTransactionId($charge->id);
        $payment->setIsTransactionClosed(false);
        
    } catch (\Stripe\Exception\CardException $e) {
        throw new \Exception('Stripe capture failed: ' . $e->getMessage());
    }
    
    return $this;
}
```

### PayPal Capture
```php
public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
{
    $authorizationId = $payment->getParentTransactionId();
    
    $captureRequest = [
        'METHOD' => 'DoCapture',
        'AUTHORIZATIONID' => $authorizationId,
        'AMT' => number_format($amount, 2),
        'CURRENCYCODE' => $payment->getOrder()->getOrderCurrencyCode(),
        'COMPLETETYPE' => 'Complete'
    ];
    
    $response = $this->paypalApi->call($captureRequest);
    
    if ($response['ACK'] === 'Success') {
        $payment->setTransactionId($response['TRANSACTIONID']);
        $payment->setIsTransactionClosed(false);
    } else {
        throw new \Exception('PayPal capture failed: ' . $response['L_LONGMESSAGE0']);
    }
    
    return $this;
}
```

### Authorize.Net Capture
```php
public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
{
    $transactionId = $payment->getParentTransactionId();
    
    $request = new \net\authorize\api\contract\v1\CreateTransactionRequest();
    $request->setMerchantAuthentication($this->getMerchantAuth());
    
    // Set transaction type to prior auth capture
    $transactionRequestType = new \net\authorize\api\contract\v1\TransactionRequestType();
    $transactionRequestType->setTransactionType("priorAuthCaptureTransaction");
    $transactionRequestType->setAmount($amount);
    $transactionRequestType->setRefTransId($transactionId);
    
    $request->setTransactionRequest($transactionRequestType);
    
    $controller = new \net\authorize\api\controller\CreateTransactionController($request);
    $response = $controller->executeWithApiResponse($this->getApiUrl());
    
    if ($response->getMessages()->getResultCode() === "Ok") {
        $captureTransactionId = $response->getTransactionResponse()->getTransId();
        $payment->setTransactionId($captureTransactionId);
    } else {
        throw new \Exception('Authorize.Net capture failed');
    }
    
    return $this;
}
```

---

## Capture States and Lifecycle

### Payment Transaction States
```php
// Transaction states related to capture
const TYPE_AUTH = 'authorization';           // Authorization only
const TYPE_CAPTURE = 'capture';             // Full capture
const TYPE_PARTIAL_CAPTURE = 'partial_capture'; // Partial capture
const TYPE_VOID = 'void';                   // Void authorization
```

### Capture Lifecycle Flow
```
Authorization Created → Can Capture → Capture Initiated → Capture Processing → Capture Complete
        ↓                   ↓              ↓                     ↓                  ↓
   Hold funds         Manual/Auto     Gateway call        Network processing    Money settled
        ↓                   ↓              ↓                     ↓                  ↓
   Can void/expire    Can still cancel  Can timeout       Can fail/succeed    Transaction closed
```

### Order State Changes During Capture
```php
public function getOrderStateChangesOnCapture()
{
    return [
        'before_capture' => [
            'state' => Order::STATE_PENDING_PAYMENT,
            'status' => 'pending_payment',
            'can_cancel' => true,
            'can_void' => true
        ],
        'capture_initiated' => [
            'state' => Order::STATE_PROCESSING,
            'status' => 'processing',
            'can_cancel' => false,
            'can_void' => false
        ],
        'capture_complete' => [
            'state' => Order::STATE_PROCESSING,
            'status' => 'processing',
            'can_ship' => true,
            'can_refund' => true
        ]
    ];
}
```

---

## Configuration Settings

### Payment Method Configuration
```xml
<!-- Payment method configuration -->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <default>
        <payment>
            <stripe_payments>
                <payment_action>authorize</payment_action>
                <can_capture>1</can_capture>
                <can_capture_partial>1</can_capture_partial>
                <can_void>1</can_void>
            </stripe_payments>
        </payment>
    </default>
</config>
```

### Admin Configuration Interface
```
Stores → Configuration → Sales → Payment Methods → [Payment Method]
```

**Key Settings:**
- **Payment Action**: authorize_capture / authorize / order
- **Automatic Invoice**: Create invoice on capture
- **Capture Timeout**: Days before authorization expires
- **Partial Capture**: Allow partial amount captures

### Programmatic Configuration
```php
public function configurePaymentCapture($paymentMethod, $settings)
{
    $configData = [
        "payment/{$paymentMethod}/payment_action" => $settings['payment_action'],
        "payment/{$paymentMethod}/can_capture" => $settings['can_capture'],
        "payment/{$paymentMethod}/can_capture_partial" => $settings['can_capture_partial'],
        "payment/{$paymentMethod}/authorization_expiry" => $settings['auth_expiry_days']
    ];
    
    foreach ($configData as $path => $value) {
        $this->configWriter->save($path, $value);
    }
    
    $this->cacheManager->clean(['config']);
}
```

---

## Best Practices

### 1. **Choose Appropriate Payment Action**
```php
public function recommendPaymentAction($businessType, $orderCharacteristics)
{
    // High-volume, low-risk: Authorize and Capture
    if ($businessType === 'digital_goods' || $orderCharacteristics['risk_score'] < 30) {
        return 'authorize_capture';
    }
    
    // High-value, custom orders: Authorize Only
    if ($orderCharacteristics['order_value'] > 1000 || $orderCharacteristics['custom_order']) {
        return 'authorize';
    }
    
    // Dropshipping, long fulfillment: Authorize Only
    if ($businessType === 'dropshipping' || $orderCharacteristics['fulfillment_days'] > 5) {
        return 'authorize';
    }
    
    return 'authorize_capture'; // Default
}
```

### 2. **Monitor Authorization Expiry**
```php
public function monitorAuthorizationExpiry()
{
    $expiringAuthorizations = $this->orderCollection
        ->addFieldToFilter('state', Order::STATE_PENDING_PAYMENT)
        ->addFieldToFilter('created_at', [
            'lt' => date('Y-m-d H:i:s', strtotime('-6 days'))
        ]);
    
    foreach ($expiringAuthorizations as $order) {
        // Notify admin of expiring authorization
        $this->notificationService->sendAuthExpiryAlert($order);
        
        // Attempt capture before expiry
        if ($order->canInvoice()) {
            $this->attemptAutoCapture($order);
        }
    }
}
```

### 3. **Handle Capture Failures Gracefully**
```php
public function handleCaptureFailure($order, $exception)
{
    // Log the failure
    $this->logger->error('Payment capture failed', [
        'order_id' => $order->getId(),
        'payment_method' => $order->getPayment()->getMethod(),
        'error' => $exception->getMessage()
    ]);
    
    // Notify customer service
    $this->customerService->notifyCaptureFailure($order, $exception);
    
    // Try alternative actions
    if ($this->canRetryCapture($exception)) {
        $this->scheduleCaptureRetry($order);
    } else {
        $this->suggestManualIntervention($order);
    }
}
```

### 4. **Implement Partial Capture Strategy**
```php
public function implementPartialCaptureStrategy($order)
{
    $shippableItems = $this->getShippableItems($order);
    $backorderedItems = $this->getBackorderedItems($order);
    
    if (!empty($shippableItems)) {
        // Capture for items ready to ship
        $shippableAmount = $this->calculateItemsTotal($shippableItems);
        $this->createPartialCapture($order, $shippableAmount, $shippableItems);
    }
    
    // Schedule capture for backordered items when available
    if (!empty($backorderedItems)) {
        $this->scheduleBackorderCapture($order, $backorderedItems);
    }
}
```

---

## Troubleshooting

### Common Capture Issues

#### 1. **Authorization Expired**
**Problem**: Cannot capture because authorization expired
**Solution**:
```php
public function handleExpiredAuthorization($order)
{
    // Check if we can re-authorize
    $payment = $order->getPayment();
    
    if ($this->canReauthorize($payment)) {
        // Attempt re-authorization
        $newAuth = $this->reauthorizePayment($payment, $order->getGrandTotal());
        
        if ($newAuth->isSuccessful()) {
            // Proceed with capture
            $this->capturePayment($payment, $order->getGrandTotal());
        }
    } else {
        // Require new payment method
        $this->requestNewPaymentMethod($order);
    }
}
```

#### 2. **Insufficient Funds During Capture**
**Problem**: Customer's account lacks funds at capture time
**Solution**:
```php
public function handleInsufficientFunds($order, $exception)
{
    // Notify customer
    $this->emailService->sendPaymentFailureNotification($order, $exception);
    
    // Put order on hold
    $order->setStatus('payment_failed');
    $order->setState(Order::STATE_PAYMENT_REVIEW);
    $order->addCommentToStatusHistory(
        'Payment capture failed due to insufficient funds. Customer notified.'
    );
    
    // Schedule retry after some time
    $this->schedulePaymentRetry($order, '+2 days');
}
```

#### 3. **Gateway Capture Timeout**
**Problem**: Payment gateway doesn't respond within timeout
**Solution**:
```php
public function handleCaptureTimeout($order, $payment)
{
    // Log timeout incident
    $this->logger->warning('Payment capture timeout', [
        'order_id' => $order->getId(),
        'payment_method' => $payment->getMethod(),
        'timeout_duration' => $this->getGatewayTimeout()
    ]);
    
    // Query gateway for transaction status
    $transactionStatus = $this->queryTransactionStatus($payment->getLastTransId());
    
    switch ($transactionStatus) {
        case 'captured':
            $this->markCaptureComplete($payment);
            break;
        case 'pending':
            $this->scheduleCaptureStatusCheck($payment);
            break;
        case 'failed':
            $this->handleCaptureFailed($order, $payment);
            break;
    }
}
```

#### 4. **Partial Capture Issues**
**Problem**: Partial capture amounts don't add up correctly
**Solution**:
```php
public function validatePartialCapture($order, $captureAmount)
{
    $payment = $order->getPayment();
    $authorizedAmount = $payment->getAmountAuthorized();
    $alreadyCaptured = $payment->getAmountPaid();
    $remainingAmount = $authorizedAmount - $alreadyCaptured;
    
    if ($captureAmount > $remainingAmount) {
        throw new \Exception(
            "Capture amount ({$captureAmount}) exceeds remaining authorized amount ({$remainingAmount})"
        );
    }
    
    // Log partial capture for audit
    $this->auditLogger->info('Partial capture validated', [
        'order_id' => $order->getId(),
        'capture_amount' => $captureAmount,
        'remaining_amount' => $remainingAmount - $captureAmount
    ]);
    
    return true;
}
```

---

## Summary

**Payment Capture in Magento 2** is the critical step that converts authorized payments into actual money transfer:

### **Key Concepts:**
- **Authorization** = Reserve funds (temporary hold)
- **Capture** = Collect money (actual transfer)
- **Two-step process** provides flexibility and fraud protection

### **Capture Timing:**
- **Immediate** (Authorize and Capture) - Money taken at order placement
- **Delayed** (Authorize Only) - Money taken when invoice is created
- **Partial** - Multiple captures for single authorization

### **Configuration:**
- **Payment Action** setting controls when capture happens
- **"Authorize and Capture"** = Automatic immediate capture
- **"Authorize Only"** = Manual capture required

### **Technical Process:**
1. Customer places order → Authorization
2. Invoice creation triggers → Capture
3. Payment gateway processes → Money transfer
4. Order status updates → Ready for fulfillment

### **Best Practices:**
- Choose payment action based on business model
- Monitor authorization expiry dates
- Handle capture failures gracefully
- Use partial capture for complex fulfillment
- Implement proper error handling and logging

**Payment capture is essential for completing transactions and enabling order fulfillment** while providing the flexibility to manage cash flow and fraud prevention effectively.