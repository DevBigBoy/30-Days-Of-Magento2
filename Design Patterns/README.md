# Chain of Responsibility Pattern in Magento 2 - Complete Implementation Guide

## Table of Contents
1. [Pattern Overview](#pattern-overview)
2. [Chain of Responsibility in Magento 2](#chain-of-responsibility-in-magento-2)
3. [Implementation Architecture](#implementation-architecture)
4. [Order Processing Chain Example](#order-processing-chain-example)
5. [Payment Validation Chain](#payment-validation-chain)
6. [Product Validation Chain](#product-validation-chain)
7. [API Request Processing Chain](#api-request-processing-chain)
8. [Customer Registration Chain](#customer-registration-chain)
9. [Configuration and DI Setup](#configuration-and-di-setup)
10. [Advanced Patterns](#advanced-patterns)
11. [Best Practices](#best-practices)
12. [Real-World Use Cases](#real-world-use-cases)

---

## Pattern Overview

### What is Chain of Responsibility?
The **Chain of Responsibility** pattern allows you to pass requests along a chain of handlers. Each handler decides either to process the request or pass it to the next handler in the chain.

### Key Benefits in Magento 2
- **Decoupled processing** - Each handler is independent
- **Flexible ordering** - Easy to reorder or add new handlers
- **Single responsibility** - Each handler has one specific job
- **Extensible** - Third-party modules can add their own handlers
- **Testable** - Each handler can be tested independently

### Pattern Structure
```php
Request → Handler1 → Handler2 → Handler3 → Result
   ↓         ↓         ↓         ↓
Process   Process   Process   Process
   or        or        or        or
  Skip      Skip      Skip    Final
```

---

## Chain of Responsibility in Magento 2

### Existing Implementations in Magento
Magento 2 already uses this pattern in several places:

1. **Plugin System** - Interceptors form a chain
2. **Total Collectors** - Cart total calculation chain
3. **Shipping Rate Collectors** - Shipping calculation chain
4. **Tax Collectors** - Tax calculation chain
5. **Price Modifiers** - Product price calculation chain

### Example: Total Collectors Chain
```php
// Magento's existing total collectors chain
Subtotal Collector → Discount Collector → Shipping Collector → Tax Collector → Grand Total Collector
```

---

## Implementation Architecture

### Base Handler Interface
```php
<?php

namespace Vendor\Module\Api;

interface ProcessorHandlerInterface
{
    /**
     * Process the request
     *
     * @param \Vendor\Module\Api\Data\ProcessRequestInterface $request
     * @return \Vendor\Module\Api\Data\ProcessResultInterface
     */
    public function process(\Vendor\Module\Api\Data\ProcessRequestInterface $request): \Vendor\Module\Api\Data\ProcessResultInterface;

    /**
     * Set next handler in chain
     *
     * @param \Vendor\Module\Api\ProcessorHandlerInterface $handler
     * @return \Vendor\Module\Api\ProcessorHandlerInterface
     */
    public function setNext(\Vendor\Module\Api\ProcessorHandlerInterface $handler): \Vendor\Module\Api\ProcessorHandlerInterface;
}
```

### Abstract Base Handler
```php
<?php

namespace Vendor\Module\Model\Handler;

use Vendor\Module\Api\ProcessorHandlerInterface;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

abstract class AbstractHandler implements ProcessorHandlerInterface
{
    /**
     * @var ProcessorHandlerInterface
     */
    private $nextHandler;

    /**
     * Set next handler
     */
    public function setNext(ProcessorHandlerInterface $handler): ProcessorHandlerInterface
    {
        $this->nextHandler = $handler;
        return $handler;
    }

    /**
     * Process request
     */
    public function process(ProcessRequestInterface $request): ProcessResultInterface
    {
        // Check if this handler can process the request
        if ($this->canProcess($request)) {
            return $this->doProcess($request);
        }

        // Pass to next handler if exists
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        // No handler could process the request
        return $this->createErrorResult('No handler found for request');
    }

    /**
     * Check if this handler can process the request
     */
    abstract protected function canProcess(ProcessRequestInterface $request): bool;

    /**
     * Do the actual processing
     */
    abstract protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface;

    /**
     * Create error result
     */
    abstract protected function createErrorResult(string $message): ProcessResultInterface;
}
```

### Request and Result Data Interfaces
```php
<?php

namespace Vendor\Module\Api\Data;

interface ProcessRequestInterface
{
    /**
     * Get request type
     *
     * @return string
     */
    public function getType(): string;

    /**
     * Get request data
     *
     * @return array
     */
    public function getData(): array;

    /**
     * Get specific data by key
     *
     * @param string $key
     * @return mixed
     */
    public function getDataByKey(string $key);

    /**
     * Set data
     *
     * @param string $key
     * @param mixed $value
     * @return $this
     */
    public function setData(string $key, $value): ProcessRequestInterface;
}

interface ProcessResultInterface
{
    /**
     * Is processing successful
     *
     * @return bool
     */
    public function isSuccess(): bool;

    /**
     * Get result data
     *
     * @return array
     */
    public function getData(): array;

    /**
     * Get error messages
     *
     * @return array
     */
    public function getErrors(): array;

    /**
     * Add error
     *
     * @param string $error
     * @return $this
     */
    public function addError(string $error): ProcessResultInterface;
}
```

---

## Order Processing Chain Example

### Use Case: Complex Order Processing
Let's create a chain to process orders through multiple validation and enrichment steps.

### Order Processing Handlers

#### 1. Customer Validation Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Order;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;
use Magento\Customer\Api\CustomerRepositoryInterface;

class CustomerValidationHandler extends AbstractHandler
{
    private $customerRepository;
    private $resultFactory;

    public function __construct(
        CustomerRepositoryInterface $customerRepository,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->customerRepository = $customerRepository;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'order_processing' 
            && $request->getDataByKey('customer_id');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $customerId = $request->getDataByKey('customer_id');
        $result = $this->resultFactory->create();

        try {
            // Validate customer exists and is active
            $customer = $this->customerRepository->getById($customerId);
            
            if (!$this->isCustomerValid($customer)) {
                return $result->addError('Customer is not valid for order processing');
            }

            // Enrich request with customer data
            $request->setData('customer_data', [
                'group_id' => $customer->getGroupId(),
                'email' => $customer->getEmail(),
                'is_vip' => $this->isVipCustomer($customer)
            ]);

            // Continue to next handler
            if ($this->nextHandler) {
                return $this->nextHandler->process($request);
            }

            return $result;

        } catch (\Exception $e) {
            return $result->addError('Customer validation failed: ' . $e->getMessage());
        }
    }

    private function isCustomerValid($customer): bool
    {
        // Custom validation logic
        return $customer->getId() && 
               $customer->getConfirmation() === null;
    }

    private function isVipCustomer($customer): bool
    {
        // VIP customer logic
        return $customer->getGroupId() === 2; // VIP group
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 2. Inventory Validation Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Order;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;
use Magento\CatalogInventory\Api\StockStateInterface;

class InventoryValidationHandler extends AbstractHandler
{
    private $stockState;
    private $resultFactory;

    public function __construct(
        StockStateInterface $stockState,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->stockState = $stockState;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'order_processing' 
            && $request->getDataByKey('order_items');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $orderItems = $request->getDataByKey('order_items');
        $result = $this->resultFactory->create();

        foreach ($orderItems as $item) {
            if (!$this->isItemInStock($item)) {
                return $result->addError(
                    sprintf('Product %s is out of stock', $item['sku'])
                );
            }
        }

        // Add inventory status to request
        $request->setData('inventory_validated', true);
        $request->setData('inventory_status', $this->getInventoryStatus($orderItems));

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    private function isItemInStock($item): bool
    {
        return $this->stockState->verifyStock(
            $item['product_id'], 
            $item['qty']
        );
    }

    private function getInventoryStatus($items): array
    {
        $status = [];
        foreach ($items as $item) {
            $status[$item['sku']] = [
                'available_qty' => $this->stockState->getStockQty($item['product_id']),
                'is_in_stock' => $this->isItemInStock($item)
            ];
        }
        return $status;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 3. Fraud Detection Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Order;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class FraudDetectionHandler extends AbstractHandler
{
    private $fraudService;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\FraudDetectionService $fraudService,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->fraudService = $fraudService;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'order_processing';
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();

        // Perform fraud detection
        $fraudScore = $this->fraudService->calculateFraudScore([
            'customer_data' => $request->getDataByKey('customer_data'),
            'order_total' => $request->getDataByKey('order_total'),
            'billing_address' => $request->getDataByKey('billing_address'),
            'payment_method' => $request->getDataByKey('payment_method')
        ]);

        if ($fraudScore > 75) {
            $request->setData('requires_manual_review', true);
            $request->setData('fraud_score', $fraudScore);
        } elseif ($fraudScore > 50) {
            $request->setData('fraud_score', $fraudScore);
            $request->setData('fraud_warnings', $this->fraudService->getWarnings());
        }

        // Continue processing regardless of fraud score
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 4. Price Calculation Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Order;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class PriceCalculationHandler extends AbstractHandler
{
    private $priceCalculator;
    private $discountService;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\PriceCalculatorService $priceCalculator,
        \Vendor\Module\Service\DiscountService $discountService,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->priceCalculator = $priceCalculator;
        $this->discountService = $discountService;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'order_processing' 
            && $request->getDataByKey('order_items');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $customerData = $request->getDataByKey('customer_data');
        $orderItems = $request->getDataByKey('order_items');

        // Calculate base prices
        $subtotal = $this->priceCalculator->calculateSubtotal($orderItems);
        
        // Apply customer-specific pricing
        if ($customerData['is_vip']) {
            $subtotal = $this->applyVipDiscount($subtotal);
        }

        // Apply available discounts
        $discounts = $this->discountService->getApplicableDiscounts([
            'customer_group' => $customerData['group_id'],
            'subtotal' => $subtotal,
            'items' => $orderItems
        ]);

        $finalTotal = $this->discountService->applyDiscounts($subtotal, $discounts);

        // Update request with pricing data
        $request->setData('subtotal', $subtotal);
        $request->setData('applied_discounts', $discounts);
        $request->setData('final_total', $finalTotal);

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    private function applyVipDiscount($subtotal): float
    {
        return $subtotal * 0.9; // 10% VIP discount
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

### Order Processing Chain Manager
```php
<?php

namespace Vendor\Module\Service;

use Vendor\Module\Api\ProcessorHandlerInterface;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class OrderProcessingChainManager
{
    private $handlers;

    public function __construct(array $handlers = [])
    {
        $this->handlers = $handlers;
    }

    /**
     * Process order through the chain
     *
     * @param ProcessRequestInterface $request
     * @return ProcessResultInterface
     */
    public function processOrder(ProcessRequestInterface $request): ProcessResultInterface
    {
        $chain = $this->buildChain();
        return $chain->process($request);
    }

    /**
     * Build the processing chain
     *
     * @return ProcessorHandlerInterface
     */
    private function buildChain(): ProcessorHandlerInterface
    {
        if (empty($this->handlers)) {
            throw new \InvalidArgumentException('No handlers configured for order processing chain');
        }

        $firstHandler = null;
        $previousHandler = null;

        foreach ($this->handlers as $handler) {
            if (!$handler instanceof ProcessorHandlerInterface) {
                throw new \InvalidArgumentException('Handler must implement ProcessorHandlerInterface');
            }

            if ($firstHandler === null) {
                $firstHandler = $handler;
            }

            if ($previousHandler !== null) {
                $previousHandler->setNext($handler);
            }

            $previousHandler = $handler;
        }

        return $firstHandler;
    }

    /**
     * Add handler to chain
     *
     * @param ProcessorHandlerInterface $handler
     * @return $this
     */
    public function addHandler(ProcessorHandlerInterface $handler): self
    {
        $this->handlers[] = $handler;
        return $this;
    }
}
```

---

## Payment Validation Chain

### Payment Validation Use Case
Create a chain to validate different payment methods with specific requirements.

### Payment Validation Handlers

#### 1. Credit Card Validation Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Payment;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class CreditCardValidationHandler extends AbstractHandler
{
    private $cardValidator;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\CreditCardValidatorService $cardValidator,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->cardValidator = $cardValidator;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        $paymentMethod = $request->getDataByKey('payment_method');
        return $request->getType() === 'payment_validation' 
            && in_array($paymentMethod, ['stripe', 'authorize_net', 'braintree']);
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $paymentData = $request->getDataByKey('payment_data');

        // Validate credit card data
        if (!$this->cardValidator->validateCardNumber($paymentData['card_number'])) {
            return $result->addError('Invalid credit card number');
        }

        if (!$this->cardValidator->validateExpiryDate($paymentData['expiry_date'])) {
            return $result->addError('Invalid or expired credit card');
        }

        if (!$this->cardValidator->validateCvv($paymentData['cvv'])) {
            return $result->addError('Invalid CVV code');
        }

        // Add validation results
        $request->setData('card_validated', true);
        $request->setData('card_type', $this->cardValidator->getCardType($paymentData['card_number']));

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 2. Address Verification Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Payment;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class AddressVerificationHandler extends AbstractHandler
{
    private $addressValidator;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\AddressValidatorService $addressValidator,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->addressValidator = $addressValidator;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'payment_validation' 
            && $request->getDataByKey('billing_address');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $billingAddress = $request->getDataByKey('billing_address');

        // Verify billing address
        $verification = $this->addressValidator->verifyAddress($billingAddress);

        if (!$verification['is_valid']) {
            if ($verification['severity'] === 'error') {
                return $result->addError('Billing address verification failed');
            } else {
                $request->setData('address_warnings', $verification['warnings']);
            }
        }

        $request->setData('address_verified', $verification['is_valid']);
        $request->setData('normalized_address', $verification['normalized_address']);

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 3. Risk Assessment Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Payment;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class RiskAssessmentHandler extends AbstractHandler
{
    private $riskCalculator;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\RiskCalculatorService $riskCalculator,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->riskCalculator = $riskCalculator;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'payment_validation';
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();

        $riskScore = $this->riskCalculator->calculateRisk([
            'payment_method' => $request->getDataByKey('payment_method'),
            'amount' => $request->getDataByKey('amount'),
            'customer_history' => $request->getDataByKey('customer_history'),
            'card_validated' => $request->getDataByKey('card_validated'),
            'address_verified' => $request->getDataByKey('address_verified')
        ]);

        $request->setData('risk_score', $riskScore);

        if ($riskScore > 80) {
            return $result->addError('Transaction blocked due to high risk score');
        } elseif ($riskScore > 50) {
            $request->setData('requires_3ds', true);
        }

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

---

## Product Validation Chain

### Product Import/Update Validation

#### 1. SKU Validation Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Product;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class SkuValidationHandler extends AbstractHandler
{
    private $productRepository;
    private $resultFactory;

    public function __construct(
        \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->productRepository = $productRepository;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'product_validation' 
            && $request->getDataByKey('sku');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $sku = $request->getDataByKey('sku');
        $isUpdate = $request->getDataByKey('is_update', false);

        // Validate SKU format
        if (!$this->isValidSkuFormat($sku)) {
            return $result->addError('Invalid SKU format');
        }

        // Check for duplicates (for new products)
        if (!$isUpdate && $this->skuExists($sku)) {
            return $result->addError('SKU already exists');
        }

        $request->setData('sku_validated', true);

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    private function isValidSkuFormat($sku): bool
    {
        // Custom SKU validation logic
        return preg_match('/^[A-Z0-9\-_]{3,50}$/i', $sku);
    }

    private function skuExists($sku): bool
    {
        try {
            $this->productRepository->get($sku);
            return true;
        } catch (\Magento\Framework\Exception\NoSuchEntityException $e) {
            return false;
        }
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 2. Price Validation Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Product;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class PriceValidationHandler extends AbstractHandler
{
    private $resultFactory;
    private $scopeConfig;

    public function __construct(
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory,
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig
    ) {
        $this->resultFactory = $resultFactory;
        $this->scopeConfig = $scopeConfig;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'product_validation' 
            && $request->getDataByKey('price') !== null;
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $price = $request->getDataByKey('price');
        $specialPrice = $request->getDataByKey('special_price');

        // Validate price format
        if (!is_numeric($price) || $price < 0) {
            return $result->addError('Invalid price format');
        }

        // Validate minimum price
        $minPrice = $this->scopeConfig->getValue('catalog/product/minimum_price');
        if ($minPrice && $price < $minPrice) {
            return $result->addError('Price below minimum allowed');
        }

        // Validate special price
        if ($specialPrice !== null) {
            if (!is_numeric($specialPrice) || $specialPrice < 0) {
                return $result->addError('Invalid special price format');
            }
            
            if ($specialPrice >= $price) {
                return $result->addError('Special price must be lower than regular price');
            }
        }

        $request->setData('price_validated', true);

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

---

## API Request Processing Chain

### API Authentication and Rate Limiting Chain

#### 1. Authentication Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Api;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class AuthenticationHandler extends AbstractHandler
{
    private $tokenValidator;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\TokenValidatorService $tokenValidator,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->tokenValidator = $tokenValidator;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'api_request' 
            && $request->getDataByKey('auth_token');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $token = $request->getDataByKey('auth_token');

        $validation = $this->tokenValidator->validateToken($token);

        if (!$validation['is_valid']) {
            return $result->addError('Invalid authentication token');
        }

        if ($validation['is_expired']) {
            return $result->addError('Authentication token expired');
        }

        // Add user context to request
        $request->setData('user_id', $validation['user_id']);
        $request->setData('user_role', $validation['user_role']);
        $request->setData('permissions', $validation['permissions']);

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

#### 2. Rate Limiting Handler
```php
<?php

namespace Vendor\Module\Model\Handler\Api;

use Vendor\Module\Model\Handler\AbstractHandler;
use Vendor\Module\Api\Data\ProcessRequestInterface;
use Vendor\Module\Api\Data\ProcessResultInterface;

class RateLimitingHandler extends AbstractHandler
{
    private $rateLimiter;
    private $resultFactory;

    public function __construct(
        \Vendor\Module\Service\RateLimiterService $rateLimiter,
        \Vendor\Module\Api\Data\ProcessResultInterfaceFactory $resultFactory
    ) {
        $this->rateLimiter = $rateLimiter;
        $this->resultFactory = $resultFactory;
    }

    protected function canProcess(ProcessRequestInterface $request): bool
    {
        return $request->getType() === 'api_request' 
            && $request->getDataByKey('user_id');
    }

    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        $result = $this->resultFactory->create();
        $userId = $request->getDataByKey('user_id');
        $endpoint = $request->getDataByKey('endpoint');

        $rateLimit = $this->rateLimiter->checkRateLimit($userId, $endpoint);

        if ($rateLimit['exceeded']) {
            return $result->addError(
                sprintf(
                    'Rate limit exceeded. Try again in %d seconds',
                    $rateLimit['reset_time']
                )
            );
        }

        // Add rate limit info to response headers
        $request->setData('rate_limit_remaining', $rateLimit['remaining']);
        $request->setData('rate_limit_reset', $rateLimit['reset_time']);

        // Continue to next handler
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }

        return $result;
    }

    protected function createErrorResult(string $message): ProcessResultInterface
    {
        return $this->resultFactory->create()->addError($message);
    }
}
```

---

## Configuration and DI Setup

### Dependency Injection Configuration
```xml
<!-- etc/di.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    
    <!-- Order Processing Chain Configuration -->
    <type name="Vendor\Module\Service\OrderProcessingChainManager">
        <arguments>
            <argument name="handlers" xsi:type="array">
                <item name="customer_validation" xsi:type="object">Vendor\Module\Model\Handler\Order\CustomerValidationHandler</item>
                <item name="inventory_validation" xsi:type="object">Vendor\Module\Model\Handler\Order\InventoryValidationHandler</item>
                <item name="fraud_detection" xsi:type="object">Vendor\Module\Model\Handler\Order\FraudDetectionHandler</item>
                <item name="price_calculation" xsi:type="object">Vendor\Module\Model\Handler\Order\PriceCalculationHandler</item>
            </argument>
        </arguments>
    </type>

    <!-- Payment Validation Chain Configuration -->
    <type name="Vendor\Module\Service\PaymentValidationChainManager">
        <arguments>
            <argument name="handlers" xsi:type="array">
                <item name="credit_card_validation" xsi:type="object">Vendor\Module\Model\Handler\Payment\CreditCardValidationHandler</item>
                <item name="address_verification" xsi:type="object">Vendor\Module\Model\Handler\Payment\AddressVerificationHandler</item>
                <item name="risk_assessment" xsi:type="object">Vendor\Module\Model\Handler\Payment\RiskAssessmentHandler</item>
            </argument>
        </arguments>
    </type>

    <!-- Product Validation Chain Configuration -->
    <type name="Vendor\Module\Service\ProductValidationChainManager">
        <arguments>
            <argument name="handlers" xsi:type="array">
                <item name="sku_validation" xsi:type="object">Vendor\Module\Model\Handler\Product\SkuValidationHandler</item>
                <item name="price_validation" xsi:type="object">Vendor\Module\Model\Handler\Product\PriceValidationHandler</item>
                <item name="category_validation" xsi:type="object">Vendor\Module\Model\Handler\Product\CategoryValidationHandler</item>
                <item name="image_validation" xsi:type="object">Vendor\Module\Model\Handler\Product\ImageValidationHandler</item>
            </argument>
        </arguments>
    </type>

    <!-- API Request Processing Chain Configuration -->
    <type name="Vendor\Module\Service\ApiRequestChainManager">
        <arguments>
            <argument name="handlers" xsi:type="array">
                <item name="authentication" xsi:type="object">Vendor\Module\Model\Handler\Api\AuthenticationHandler</item>
                <item name="rate_limiting" xsi:type="object">Vendor\Module\Model\Handler\Api\RateLimitingHandler</item>
                <item name="permission_check" xsi:type="object">Vendor\Module\Model\Handler\Api\PermissionHandler</item>
                <item name="request_validation" xsi:type="object">Vendor\Module\Model\Handler\Api\RequestValidationHandler</item>
            </argument>
        </arguments>
    </type>

    <!-- Virtual Types for Different Chain Configurations -->
    <virtualType name="OrderProcessingChainForVipCustomers" type="Vendor\Module\Service\OrderProcessingChainManager">
        <arguments>
            <argument name="handlers" xsi:type="array">
                <item name="customer_validation" xsi:type="object">Vendor\Module\Model\Handler\Order\VipCustomerValidationHandler</item>
                <item name="inventory_validation" xsi:type="object">Vendor\Module\Model\Handler\Order\InventoryValidationHandler</item>
                <item name="vip_pricing" xsi:type="object">Vendor\Module\Model\Handler\Order\VipPricingHandler</item>
                <item name="fraud_detection" xsi:type="object">Vendor\Module\Model\Handler\Order\FraudDetectionHandler</item>
            </argument>
        </arguments>
    </virtualType>

</config>
```

### Factory Pattern Integration
```php
<?php

namespace Vendor\Module\Service;

class ChainManagerFactory
{
    private $objectManager;

    public function __construct(
        \Magento\Framework\ObjectManagerInterface $objectManager
    ) {
        $this->objectManager = $objectManager;
    }

    /**
     * Create chain manager by type
     *
     * @param string $type
     * @param array $customHandlers
     * @return mixed
     */
    public function create(string $type, array $customHandlers = [])
    {
        $managerClass = $this->getManagerClass($type);
        
        if (!empty($customHandlers)) {
            return $this->objectManager->create($managerClass, [
                'handlers' => $customHandlers
            ]);
        }

        return $this->objectManager->get($managerClass);
    }

    private function getManagerClass(string $type): string
    {
        $managers = [
            'order_processing' => \Vendor\Module\Service\OrderProcessingChainManager::class,
            'payment_validation' => \Vendor\Module\Service\PaymentValidationChainManager::class,
            'product_validation' => \Vendor\Module\Service\ProductValidationChainManager::class,
            'api_request' => \Vendor\Module\Service\ApiRequestChainManager::class,
        ];

        if (!isset($managers[$type])) {
            throw new \InvalidArgumentException("Unknown chain type: {$type}");
        }

        return $managers[$type];
    }
}
```

---

## Advanced Patterns

### Conditional Chain Building
```php
<?php

namespace Vendor\Module\Service;

class ConditionalChainBuilder
{
    private $handlerFactory;
    private $conditions;

    public function __construct(
        \Vendor\Module\Factory\HandlerFactory $handlerFactory,
        array $conditions = []
    ) {
        $this->handlerFactory = $handlerFactory;
        $this->conditions = $conditions;
    }

    /**
     * Build chain based on conditions
     *
     * @param array $context
     * @return ProcessorHandlerInterface
     */
    public function buildChain(array $context): ProcessorHandlerInterface
    {
        $handlers = [];

        foreach ($this->conditions as $condition) {
            if ($this->evaluateCondition($condition, $context)) {
                $handlers[] = $this->handlerFactory->create($condition['handler_type']);
            }
        }

        return $this->chainHandlers($handlers);
    }

    private function evaluateCondition(array $condition, array $context): bool
    {
        $field = $condition['field'];
        $operator = $condition['operator'];
        $value = $condition['value'];
        $contextValue = $context[$field] ?? null;

        switch ($operator) {
            case 'equals':
                return $contextValue === $value;
            case 'not_equals':
                return $contextValue !== $value;
            case 'in':
                return in_array($contextValue, $value);
            case 'greater_than':
                return $contextValue > $value;
            case 'less_than':
                return $contextValue < $value;
            default:
                return false;
        }
    }

    private function chainHandlers(array $handlers): ProcessorHandlerInterface
    {
        if (empty($handlers)) {
            throw new \RuntimeException('No handlers available for chain');
        }

        $firstHandler = null;
        $previousHandler = null;

        foreach ($handlers as $handler) {
            if ($firstHandler === null) {
                $firstHandler = $handler;
            }

            if ($previousHandler !== null) {
                $previousHandler->setNext($handler);
            }

            $previousHandler = $handler;
        }

        return $firstHandler;
    }
}
```

### Parallel Processing Chain
```php
<?php

namespace Vendor\Module\Service;

class ParallelProcessingChain
{
    private $handlers;
    private $resultAggregator;

    public function __construct(
        array $handlers,
        \Vendor\Module\Service\ResultAggregatorService $resultAggregator
    ) {
        $this->handlers = $handlers;
        $this->resultAggregator = $resultAggregator;
    }

    /**
     * Process request through multiple handlers in parallel
     *
     * @param ProcessRequestInterface $request
     * @return ProcessResultInterface
     */
    public function processParallel(ProcessRequestInterface $request): ProcessResultInterface
    {
        $results = [];

        // Process all handlers simultaneously
        foreach ($this->handlers as $handlerName => $handler) {
            try {
                $results[$handlerName] = $handler->process(clone $request);
            } catch (\Exception $e) {
                $results[$handlerName] = $this->createErrorResult($e->getMessage());
            }
        }

        // Aggregate results
        return $this->resultAggregator->aggregateResults($results);
    }
}
```

---

## Best Practices

### 1. Handler Design Principles
```php
// ✅ Good - Single Responsibility
class InventoryValidationHandler extends AbstractHandler
{
    // Only handles inventory validation
    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        // Focused validation logic
    }
}

// ❌ Bad - Multiple Responsibilities
class ValidationHandler extends AbstractHandler
{
    protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
    {
        // Validates inventory, customer, payment, and shipping - too much!
    }
}
```

### 2. Error Handling
```php
protected function doProcess(ProcessRequestInterface $request): ProcessResultInterface
{
    $result = $this->resultFactory->create();

    try {
        // Main processing logic
        $this->processRequest($request);
        
        // Continue chain on success
        if ($this->nextHandler) {
            return $this->nextHandler->process($request);
        }
        
        return $result;
        
    } catch (\Exception $e) {
        // Log error
        $this->logger->error('Handler processing failed', [
            'handler' => get_class($this),
            'error' => $e->getMessage(),
            'request_data' => $request->getData()
        ]);
        
        // Decide whether to continue or stop chain
        if ($this->isCriticalError($e)) {
            return $result->addError($e->getMessage());
        } else {
            // Continue with warning
            $result->addWarning($e->getMessage());
            if ($this->nextHandler) {
                return $this->nextHandler->process($request);
            }
        }
    }
}
```

### 3. Configuration-Driven Chains
```php
// Configuration in etc/chain_config.xml
<?xml version="1.0"?>
<config>
    <chains>
        <order_processing>
            <handler name="customer_validation" sort_order="10" critical="true"/>
            <handler name="inventory_validation" sort_order="20" critical="true"/>
            <handler name="fraud_detection" sort_order="30" critical="false"/>
            <handler name="price_calculation" sort_order="40" critical="true"/>
        </order_processing>
    </chains>
</config>
```

### 4. Testing Strategy
```php
<?php

namespace Vendor\Module\Test\Unit\Handler;

class CustomerValidationHandlerTest extends \PHPUnit\Framework\TestCase
{
    private $handler;
    private $mockCustomerRepository;
    private $mockResultFactory;

    protected function setUp(): void
    {
        $this->mockCustomerRepository = $this->createMock(CustomerRepositoryInterface::class);
        $this->mockResultFactory = $this->createMock(ProcessResultInterfaceFactory::class);
        
        $this->handler = new CustomerValidationHandler(
            $this->mockCustomerRepository,
            $this->mockResultFactory
        );
    }

    public function testCanProcess()
    {
        $request = $this->createMockRequest([
            'type' => 'order_processing',
            'customer_id' => 123
        ]);

        $this->assertTrue($this->handler->canProcess($request));
    }

    public function testProcessValidCustomer()
    {
        // Test with valid customer
        $request = $this->createMockRequest(['customer_id' => 123]);
        $mockCustomer = $this->createMockCustomer();
        
        $this->mockCustomerRepository
            ->expects($this->once())
            ->method('getById')
            ->with(123)
            ->willReturn($mockCustomer);

        $result = $this->handler->process($request);
        
        $this->assertTrue($result->isSuccess());
    }

    public function testProcessInvalidCustomer()
    {
        // Test with invalid customer
        $request = $this->createMockRequest(['customer_id' => 999]);
        
        $this->mockCustomerRepository
            ->expects($this->once())
            ->method('getById')
            ->with(999)
            ->willThrowException(new NoSuchEntityException());

        $result = $this->handler->process($request);
        
        $this->assertFalse($result->isSuccess());
        $this->assertContains('Customer validation failed', $result->getErrors());
    }
}
```

---

## Real-World Use Cases

### 1. E-commerce Order Processing
```php
// Complete order processing workflow
$orderData = [
    'type' => 'order_processing',
    'customer_id' => 123,
    'order_items' => [...],
    'billing_address' => [...],
    'payment_method' => 'stripe',
    'shipping_method' => 'ups_ground'
];

$request = $this->requestFactory->create($orderData);
$result = $this->orderProcessingChain->processOrder($request);

if ($result->isSuccess()) {
    // Create order
    $order = $this->orderService->createOrder($request->getData());
} else {
    // Handle validation errors
    $this->handleOrderErrors($result->getErrors());
}
```

### 2. API Request Processing
```php
// API endpoint with validation chain
public function processApiRequest(RequestInterface $httpRequest)
{
    $apiRequest = $this->requestFactory->create([
        'type' => 'api_request',
        'auth_token' => $httpRequest->getHeader('Authorization'),
        'endpoint' => $httpRequest->getPathInfo(),
        'method' => $httpRequest->getMethod(),
        'data' => $httpRequest->getContent()
    ]);

    $result = $this->apiChain->processRequest($apiRequest);

    if (!$result->isSuccess()) {
        return $this->errorResponse($result->getErrors());
    }

    // Process actual API request
    return $this->handleApiRequest($apiRequest);
}
```

### 3. Product Import Validation
```php
// Mass product import with validation
foreach ($importData as $productData) {
    $request = $this->requestFactory->create([
        'type' => 'product_validation',
        'sku' => $productData['sku'],
        'price' => $productData['price'],
        'inventory' => $productData['inventory'],
        'categories' => $productData['categories']
    ]);

    $result = $this->productValidationChain->validateProduct($request);

    if ($result->isSuccess()) {
        $validProducts[] = $request->getData();
    } else {
        $importErrors[$productData['sku']] = $result->getErrors();
    }
}
```

---

## Summary

The **Chain of Responsibility pattern** in Magento 2 provides:

### **Key Benefits:**
1. **Modular Processing** - Each handler has a single responsibility
2. **Flexible Configuration** - Easy to add, remove, or reorder handlers
3. **Extensible Design** - Third-party modules can inject their own handlers
4. **Testable Code** - Each handler can be unit tested independently
5. **Maintainable Architecture** - Clear separation of concerns

### **Implementation Steps:**
1. **Define interfaces** for handlers, requests, and results
2. **Create abstract handler** with chain management
3. **Implement concrete handlers** for specific processing
4. **Configure via DI** for flexibility
5. **Build chain managers** for different use cases

### **Best Use Cases:**
- **Order processing workflows**
- **Payment validation chains**
- **Product import/validation**
- **API request processing**
- **Customer registration flows**
- **Complex business rule validation**

This pattern helps create **maintainable, testable, and extensible** Magento 2 backends that can handle complex business logic while remaining flexible for future requirements.