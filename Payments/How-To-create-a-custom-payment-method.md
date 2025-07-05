# Magento 2 Custom Payment Method - Complete Guide

## Overview
Creating a custom payment method in Magento 2 requires several key components:
- Module structure and configuration
- Payment method model
- Frontend and admin configuration
- Block classes for rendering
- JavaScript components for checkout
- Observer for order processing

## Step 1: Create Module Structure

First, create the basic module structure:

```
app/code/YourVendor/CustomPayment/
├── etc/
│   ├── module.xml
│   ├── config.xml
│   ├── adminhtml/
│   │   └── system.xml
│   └── frontend/
│       └── di.xml
├── Model/
│   └── Payment.php
├── Block/
│   └── Info.php
├── view/
│   ├── adminhtml/
│   │   └── templates/
│   │       └── info.phtml
│   ├── frontend/
│   │   ├── layout/
│   │   │   └── checkout_index_index.xml
│   │   ├── templates/
│   │   │   └── info.phtml
│   │   └── web/
│   │       ├── js/
│   │       │   └── view/
│   │       │       └── payment/
│   │       │           └── method-renderer/
│   │       │               └── custompayment.js
│   │       └── template/
│   │           └── payment/
│   │               └── custompayment.html
├── Observer/
│   └── DataAssignObserver.php
└── registration.php
```

## Step 2: Module Registration and Configuration

**registration.php**
```php
<?php
\Magento\Framework\Component\ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'YourVendor_CustomPayment',
    __DIR__
);
```

**etc/module.xml**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="YourVendor_CustomPayment" setup_version="1.0.0">
        <sequence>
            <module name="Magento_Sales"/>
            <module name="Magento_Payment"/>
            <module name="Magento_Checkout"/>
        </sequence>
    </module>
</config>
```

**etc/config.xml**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <payment>
            <custompayment>
                <active>1</active>
                <model>YourVendor\CustomPayment\Model\Payment</model>
                <title>Custom Payment Method</title>
                <allowspecific>0</allowspecific>
                <payment_action>authorize_capture</payment_action>
                <order_status>pending</order_status>
                <sort_order>10</sort_order>
                <can_use_checkout>1</can_use_checkout>
                <can_use_internal>1</can_use_internal>
            </custompayment>
        </payment>
    </default>
</config>
```

## Step 3: Admin System Configuration

**etc/adminhtml/system.xml**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="payment">
            <group id="custompayment" translate="label" type="text" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="1">
                <label>Custom Payment Method</label>
                <field id="active" translate="label" type="select" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Enabled</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>
                <field id="title" translate="label" type="text" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Title</label>
                </field>
                <field id="order_status" translate="label" type="select" sortOrder="30" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>New Order Status</label>
                    <source_model>Magento\Sales\Model\Config\Source\Order\Status\NewStatus</source_model>
                </field>
                <field id="allowspecific" translate="label" type="allowspecific" sortOrder="40" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Payment from Applicable Countries</label>
                    <source_model>Magento\Payment\Model\Config\Source\Allspecificcountries</source_model>
                </field>
                <field id="specificcountry" translate="label" type="multiselect" sortOrder="50" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Payment from Specific Countries</label>
                    <source_model>Magento\Directory\Model\Config\Source\Country</source_model>
                    <can_be_empty>1</can_be_empty>
                </field>
                <field id="instructions" translate="label" sortOrder="60" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Instructions</label>
                </field>
                <field id="sort_order" translate="label" type="text" sortOrder="70" showInDefault="1" showInWebsite="1" showInStore="0">
                    <label>Sort Order</label>
                </field>
            </group>
        </section>
    </system>
</config>
```

## Step 4: Payment Model

**Model/Payment.php**
```php
<?php
namespace YourVendor\CustomPayment\Model;

use Magento\Payment\Model\Method\AbstractMethod;
use Magento\Payment\Model\Method\Online\GatewayInterface;
use Magento\Framework\Exception\LocalizedException;

class Payment extends AbstractMethod
{
    /**
     * Payment method code
     */
    const CODE = 'custompayment';

    /**
     * Payment method code
     */
    protected $_code = self::CODE;

    /**
     * Availability option
     */
    protected $_isOffline = false;

    /**
     * Payment Method feature
     */
    protected $_canAuthorize = true;
    protected $_canCapture = true;
    protected $_canCapturePartial = true;
    protected $_canRefund = true;
    protected $_canRefundInvoicePartial = true;
    protected $_canVoid = true;
    protected $_canUseCheckout = true;
    protected $_canUseForMultishipping = true;
    protected $_canUseInternal = true;
    protected $_isInitializeNeeded = false;

    /**
     * Authorize payment abstract method
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws LocalizedException
     */
    public function authorize(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canAuthorize()) {
            throw new LocalizedException(__('The authorize action is not available.'));
        }

        // Your authorization logic here
        $payment->setTransactionId('AUTH-' . time())
                ->setIsTransactionClosed(false);

        return $this;
    }

    /**
     * Capture payment abstract method
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws LocalizedException
     */
    public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canCapture()) {
            throw new LocalizedException(__('The capture action is not available.'));
        }

        // Your capture logic here
        $payment->setTransactionId('CAPTURE-' . time())
                ->setIsTransactionClosed(true);

        return $this;
    }

    /**
     * Refund specified amount for payment
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws LocalizedException
     */
    public function refund(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canRefund()) {
            throw new LocalizedException(__('The refund action is not available.'));
        }

        // Your refund logic here
        $payment->setTransactionId('REFUND-' . time())
                ->setIsTransactionClosed(true);

        return $this;
    }

    /**
     * Void payment abstract method
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @return $this
     * @throws LocalizedException
     */
    public function void(\Magento\Payment\Model\InfoInterface $payment)
    {
        if (!$this->canVoid()) {
            throw new LocalizedException(__('The void action is not available.'));
        }

        // Your void logic here
        $payment->setTransactionId('VOID-' . time())
                ->setIsTransactionClosed(true);

        return $this;
    }
}
```

## Step 5: Frontend Dependency Injection

**etc/frontend/di.xml**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <virtualType name="CustomPaymentFacade" type="Magento\Payment\Model\Method\Adapter">
        <arguments>
            <argument name="code" xsi:type="const">YourVendor\CustomPayment\Model\Payment::CODE</argument>
            <argument name="formBlockType" xsi:type="string">Magento\Payment\Block\Form</argument>
            <argument name="infoBlockType" xsi:type="string">YourVendor\CustomPayment\Block\Info</argument>
            <argument name="valueHandlerPool" xsi:type="object">CustomPaymentValueHandlerPool</argument>
            <argument name="commandPool" xsi:type="object">CustomPaymentCommandPool</argument>
        </arguments>
    </virtualType>

    <!-- Value handlers infrastructure -->
    <virtualType name="CustomPaymentValueHandlerPool" type="Magento\Payment\Gateway\Config\ValueHandlerPool">
        <arguments>
            <argument name="handlers" xsi:type="array">
                <item name="default" xsi:type="string">CustomPaymentDefaultValueHandler</item>
            </argument>
        </arguments>
    </virtualType>
    
    <virtualType name="CustomPaymentDefaultValueHandler" type="Magento\Payment\Gateway\Config\ConfigValueHandler">
        <arguments>
            <argument name="configInterface" xsi:type="object">CustomPaymentConfig</argument>
        </arguments>
    </virtualType>
    
    <virtualType name="CustomPaymentConfig" type="Magento\Payment\Gateway\Config\Config">
        <arguments>
            <argument name="methodCode" xsi:type="const">YourVendor\CustomPayment\Model\Payment::CODE</argument>
        </arguments>
    </virtualType>

    <!-- Commands infrastructure -->
    <virtualType name="CustomPaymentCommandPool" type="Magento\Payment\Gateway\Command\CommandPool">
        <arguments>
            <argument name="commands" xsi:type="array">
                <item name="authorize" xsi:type="string">CustomPaymentAuthorizeCommand</item>
                <item name="capture" xsi:type="string">CustomPaymentCaptureCommand</item>
                <item name="void" xsi:type="string">CustomPaymentVoidCommand</item>
                <item name="refund" xsi:type="string">CustomPaymentRefundCommand</item>
            </argument>
        </arguments>
    </virtualType>

    <!-- Authorize command -->
    <virtualType name="CustomPaymentAuthorizeCommand" type="Magento\Payment\Gateway\Command\GatewayCommand">
        <arguments>
            <argument name="requestBuilder" xsi:type="object">CustomPaymentAuthorizationRequest</argument>
            <argument name="handler" xsi:type="object">CustomPaymentResponseHandlerComposite</argument>
            <argument name="transferFactory" xsi:type="object">CustomPaymentTransferFactory</argument>
            <argument name="client" xsi:type="object">CustomPaymentHttpClient</argument>
        </arguments>
    </virtualType>
</config>
```

## Step 6: Block for Payment Information Display

**Block/Info.php**
```php
<?php
namespace YourVendor\CustomPayment\Block;

use Magento\Payment\Block\Info;

class Info extends \Magento\Payment\Block\Info
{
    /**
     * @var string
     */
    protected $_template = 'YourVendor_CustomPayment::info.phtml';

    /**
     * Get some specific information in format of array($label => $value)
     *
     * @return array
     */
    public function getSpecificInformation()
    {
        return [
            'Payment Method' => $this->getInfo()->getMethod(),
            'Transaction ID' => $this->getInfo()->getLastTransId()
        ];
    }
}
```

## Step 7: Frontend Templates

**view/frontend/templates/info.phtml**
```html
<?php
/** @var \YourVendor\CustomPayment\Block\Info $block */
?>
<div class="payment-method-info">
    <h4><?= $block->escapeHtml(__('Payment Information')) ?></h4>
    <?php foreach ($block->getSpecificInformation() as $label => $value): ?>
        <?php if ($value): ?>
            <p><strong><?= $block->escapeHtml(__($label)) ?>:</strong> <?= $block->escapeHtml($value) ?></p>
        <?php endif; ?>
    <?php endforeach; ?>
</div>
```

**view/adminhtml/templates/info.phtml**
```html
<?php
/** @var \YourVendor\CustomPayment\Block\Info $block */
?>
<table class="admin__table-secondary">
    <tr>
        <th><?= $block->escapeHtml(__('Payment Method')) ?></th>
        <td><?= $block->escapeHtml($block->getInfo()->getMethod()) ?></td>
    </tr>
    <?php if ($block->getInfo()->getLastTransId()): ?>
    <tr>
        <th><?= $block->escapeHtml(__('Transaction ID')) ?></th>
        <td><?= $block->escapeHtml($block->getInfo()->getLastTransId()) ?></td>
    </tr>
    <?php endif; ?>
</table>
```

## Step 8: Checkout Layout and JavaScript

**view/frontend/layout/checkout_index_index.xml**
```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout.payment.methods">
            <block class="Magento\Framework\View\Element\Template" 
                   name="checkout.payment.method.custompayment" 
                   template="YourVendor_CustomPayment::payment/custompayment.phtml"/>
        </referenceBlock>
    </body>
</page>
```

**view/frontend/web/js/view/payment/method-renderer/custompayment.js**
```javascript
define([
    'Magento_Checkout/js/view/payment/default',
    'mage/url'
], function (Component, url) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'YourVendor_CustomPayment/payment/custompayment'
        },

        /**
         * Get payment method code
         */
        getCode: function () {
            return 'custompayment';
        },

        /**
         * Get payment method data
         */
        getData: function () {
            return {
                'method': this.item.method,
                'additional_data': {
                    'custom_field': 'custom_value'
                }
            };
        },

        /**
         * Get payment method instructions
         */
        getInstructions: function () {
            return window.checkoutConfig.payment.instructions[this.item.method];
        },

        /**
         * Place order
         */
        placeOrder: function (data, event) {
            if (event) {
                event.preventDefault();
            }
            var self = this,
                placeOrder;

            if (this.validate() && this.isPlaceOrderActionAllowed() === true) {
                this.isPlaceOrderActionAllowed(false);
                placeOrder = this.placeOrderAction(data, false, {});

                placeOrder.done(function () {
                    self.afterPlaceOrder();
                    if (self.redirectAfterPlaceOrder) {
                        window.location.replace(url.build('checkout/onepage/success/'));
                    }
                }).fail(function () {
                    self.isPlaceOrderActionAllowed(true);
                });
                return true;
            }
            return false;
        },

        /**
         * After place order callback
         */
        afterPlaceOrder: function () {
            // Custom logic after placing order
        }
    });
});
```

**view/frontend/web/template/payment/custompayment.html**
```html
<!-- ko foreach: getRegion('messages') -->
<!-- ko template: getTemplate() --><!-- /ko -->
<!-- /ko -->

<div class="payment-method" data-bind="css: {'_active': (getCode() == isChecked())}">
    <div class="payment-method-title field choice">
        <input type="radio"
               name="payment[method]"
               class="radio"
               data-bind="attr: {'id': getCode()}, value: getCode(), checked: isChecked, click: selectPaymentMethod, visible: isRadioButtonVisible()"/>
        <label data-bind="attr: {'for': getCode()}" class="label">
            <span data-bind="text: getTitle()"></span>
        </label>
    </div>

    <div class="payment-method-content">
        <!-- ko if: getInstructions() -->
        <div class="payment-method-note" data-bind="html: getInstructions()"></div>
        <!-- /ko -->

        <div class="payment-method-billing-address">
            <!-- ko foreach: $parent.getRegion(getBillingAddressFormName()) -->
            <!-- ko template: getTemplate() --><!-- /ko -->
            <!-- /ko -->
        </div>

        <div class="checkout-agreements-block">
            <!-- ko foreach: $parent.getRegion('before-place-order') -->
            <!-- ko template: getTemplate() --><!-- /ko -->
            <!-- /ko -->
        </div>

        <div class="actions-toolbar">
            <div class="primary">
                <button class="action primary checkout"
                        type="submit"
                        data-bind="
                        click: placeOrder,
                        attr: {title: $t('Place Order')},
                        css: {disabled: !isPlaceOrderActionAllowed()},
                        enable: (getCode() == isChecked())
                        "
                        disabled>
                    <span data-bind="text: $t('Place Order')"></span>
                </button>
            </div>
        </div>
    </div>
</div>
```

## Step 9: Observer for Data Assignment

**Observer/DataAssignObserver.php**
```php
<?php
namespace YourVendor\CustomPayment\Observer;

use Magento\Framework\Event\Observer;
use Magento\Payment\Observer\AbstractDataAssignObserver;
use Magento\Quote\Api\Data\PaymentInterface;

class DataAssignObserver extends AbstractDataAssignObserver
{
    const CUSTOM_FIELD = 'custom_field';

    protected $additionalInformationList = [
        self::CUSTOM_FIELD
    ];

    /**
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer)
    {
        $data = $this->readDataArgument($observer);
        $additionalData = $data->getData(PaymentInterface::KEY_ADDITIONAL_DATA);
        
        if (!is_array($additionalData)) {
            return;
        }

        $paymentInfo = $this->readPaymentModelArgument($observer);

        foreach ($this->additionalInformationList as $additionalInformationKey) {
            if (isset($additionalData[$additionalInformationKey])) {
                $paymentInfo->setAdditionalInformation(
                    $additionalInformationKey,
                    $additionalData[$additionalInformationKey]
                );
            }
        }
    }
}
```

## Step 10: Register Payment Method Renderer

Create **view/frontend/web/js/view/payment/custompayment.js**:
```javascript
define([
    'uiComponent',
    'Magento_Checkout/js/model/payment/renderer-list'
], function (Component, rendererList) {
    'use strict';

    rendererList.push({
        type: 'custompayment',
        component: 'YourVendor_CustomPayment/js/view/payment/method-renderer/custompayment'
    });

    return Component.extend({});
});
```

## Step 11: Enable and Configure

After creating all files:

1. **Run setup commands:**
   ```bash
   php bin/magento setup:upgrade
   php bin/magento setup:di:compile
   php bin/magento setup:static-content:deploy
   php bin/magento cache:clean
   ```

2. **Configure in Admin:**
   - Go to Stores > Configuration > Sales > Payment Methods
   - Find your "Custom Payment Method" section
   - Enable it and configure the settings

3. **Test the payment method:**
   - Add products to cart
   - Go to checkout
   - Your custom payment method should appear in the payment methods list

## Additional Considerations

### Security Best Practices
- Always validate and sanitize input data
- Use HTTPS for all payment-related communications
- Implement proper error handling and logging
- Follow PCI DSS compliance requirements if handling credit card data

### Performance Optimization
- Implement caching where appropriate
- Use database transactions for payment operations
- Consider asynchronous processing for time-consuming operations

### Testing
- Create unit tests for your payment model
- Test all payment scenarios (authorize, capture, refund, void)
- Test error conditions and edge cases
- Perform integration testing with your payment gateway

### Documentation
- Document your API endpoints and request/response formats
- Create configuration guides for merchants
- Maintain changelog for updates

This comprehensive guide provides the foundation for creating a custom payment method in Magento 2. You can extend this base implementation with additional features like webhooks, multiple payment options, or integration with specific payment gateways.