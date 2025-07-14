To create a custom payment method in Magento 2, you'll need to follow these key steps:

## 1. Create Module Structure

First, create your module directory structure:
```
app/code/YourCompany/CustomPayment/
├── etc/
│   ├── module.xml
│   ├── config.xml
│   └── payment.xml
├── Model/
│   └── Payment.php
├── view/frontend/
│   ├── layout/
│   │   └── checkout_index_index.xml
│   └── web/
│       └── js/
│           └── view/
│               └── payment/
│                   └── method-renderer/
│                       └── custompayment.js
└── registration.php
```

## 2. Module Registration

**registration.php:**
```php
<?php
\Magento\Framework\Component\ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'YourCompany_CustomPayment',
    __DIR__
);
```

**etc/module.xml:**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="YourCompany_CustomPayment" setup_version="1.0.0">
        <sequence>
            <module name="Magento_Sales"/>
            <module name="Magento_Payment"/>
            <module name="Magento_Checkout"/>
        </sequence>
    </module>
</config>
```

## 3. Payment Method Model

**Model/Payment.php:**
```php
<?php
namespace YourCompany\CustomPayment\Model;

class Payment extends \Magento\Payment\Model\Method\AbstractMethod
{
    const CODE = 'custompayment';
    
    protected $_code = self::CODE;
    protected $_isOffline = true;
    
    public function isAvailable(\Magento\Quote\Api\Data\CartInterface $quote = null)
    {
        return parent::isAvailable($quote);
    }
    
    public function authorize(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        // Your authorization logic here
        return $this;
    }
    
    public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        // Your capture logic here
        return $this;
    }
}
```

## 4. Configuration Files

**etc/config.xml:**
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <payment>
            <custompayment>
                <active>1</active>
                <model>YourCompany\CustomPayment\Model\Payment</model>
                <title>Custom Payment Method</title>
                <allowspecific>0</allowspecific>
                <group>offline</group>
            </custompayment>
        </payment>
    </default>
</config>
```

**etc/payment.xml:**
```xml
<?xml version="1.0"?>
<payment xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Payment:etc/payment.xsd">
    <methods>
        <method name="custompayment">
            <allow_multiple_address>1</allow_multiple_address>
        </method>
    </methods>
</payment>
```

## 5. Frontend JavaScript Component

**view/frontend/web/js/view/payment/method-renderer/custompayment.js:**
```javascript
define([
    'Magento_Checkout/js/view/payment/default'
], function (Component) {
    'use strict';

    return Component.extend({
        defaults: {
            template: 'YourCompany_CustomPayment/payment/custompayment'
        },

        getCode: function() {
            return 'custompayment';
        },

        isActive: function() {
            return true;
        }
    });
});
```

## 6. Layout and Template

**view/frontend/layout/checkout_index_index.xml:**
```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout.root">
            <arguments>
                <argument name="jsLayout" xsi:type="array">
                    <item name="components" xsi:type="array">
                        <item name="checkout" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="steps" xsi:type="array">
                                    <item name="children" xsi:type="array">
                                        <item name="billing-step" xsi:type="array">
                                            <item name="children" xsi:type="array">
                                                <item name="payment" xsi:type="array">
                                                    <item name="children" xsi:type="array">
                                                        <item name="renders" xsi:type="array">
                                                            <item name="children" xsi:type="array">
                                                                <item name="custompayment" xsi:type="array">
                                                                    <item name="component" xsi:type="string">YourCompany_CustomPayment/js/view/payment/method-renderer/custompayment</item>
                                                                    <item name="methods" xsi:type="array">
                                                                        <item name="custompayment" xsi:type="array">
                                                                            <item name="isBillingAddressRequired" xsi:type="boolean">true</item>
                                                                        </item>
                                                                    </item>
                                                                </item>
                                                            </item>
                                                        </item>
                                                    </item>
                                                </item>
                                            </item>
                                        </item>
                                    </item>
                                </item>
                            </item>
                        </item>
                    </item>
                </argument>
            </arguments>
        </referenceBlock>
    </body>
</page>
```

## 7. Enable and Deploy

After creating these files:

1. Enable the module: `php bin/magento module:enable YourCompany_CustomPayment`
2. Run setup upgrade: `php bin/magento setup:upgrade`
3. Deploy static content: `php bin/magento setup:static-content:deploy`
4. Clear cache: `php bin/magento cache:flush`

The payment method will now appear in the admin configuration under Stores > Configuration > Sales > Payment Methods, and customers will see it during checkout.

For more advanced implementations, you might need to add system configuration fields, custom validation, API integrations, or database tables depending on your specific payment gateway requirements.