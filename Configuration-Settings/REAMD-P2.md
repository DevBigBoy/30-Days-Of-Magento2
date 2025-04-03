# 🛠️ Magento 2: Create a System.xml Configuration

This guide will walk you through **creating configuration fields** in the Magento 2 admin panel using `system.xml`.  
By the end, you’ll have your very own custom settings visible and usable inside **Stores > Configuration**!

---

## 📋 Table of Contents

- [🛠️ Magento 2: Create a System.xml Configuration](#️-magento-2-create-a-systemxml-configuration)
  - [📋 Table of Contents](#-table-of-contents)
  - [📖 Introduction](#-introduction)
  - [🛠️ Step 1: Create `system.xml`](#️-step-1-create-systemxml)
  - [⚙️ Step 2: Set Default Value](#️-step-2-set-default-value)
  - [♻️ Step 3: Flush Magento Cache](#️-step-3-flush-magento-cache)
  - [📥 Step 4: Get Value From Configuration](#-step-4-get-value-from-configuration)
    - [4.1 Simple Calling:](#41-simple-calling)
    - [4.2 Create a Helper File (Recommended)](#42-create-a-helper-file-recommended)
  - [🎯 Conclusion](#-conclusion)
- [🔥 Bonus Tips](#-bonus-tips)

---

## 📖 Introduction

The Magento 2, the `system.xml` is a configuration file that is used to create fields in Magento 2 System Configuration system.xml. You will need system.xml if your module has some settings which the admin needs to set.

---

## 🛠️ Step 1: Create `system.xml`

Create the file inside your module at:

```plaintext
app/code/Vendor/Module/etc/adminhtml/system.xml
```

Sample `system.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Config:etc/system_file.xsd">
    <system>
        <section id="vendor_section" translate="label" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="1">
            <label>Vendor Settings</label>
            <tab>general</tab> <!-- You can also create a new tab -->
            
            <resource>Vendor_Module::config</resource>

            <group id="general_group" translate="label" type="text" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
                <label>General Settings</label>

                <field id="enable" translate="label" type="select" sortOrder="10" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Enable Module</label>
                    <source_model>Magento\Config\Model\Config\Source\Yesno</source_model>
                </field>

                <field id="custom_text" translate="label" type="text" sortOrder="20" showInDefault="1" showInWebsite="1" showInStore="1">
                    <label>Custom Text</label>
                    <validate>required-entry</validate>
                </field>

            </group>
        </section>
    </system>
</config>
```

✅ **Important Notes**:

- `section` → Represents a top-level group in the config (e.g., your own module settings).
- `group` → Organizes related fields together.
- `field` → Represents a single input (text field, select, etc.).

---

## ⚙️ Step 2: Set Default Value

Create the file:

```plaintext
app/code/Vendor/Module/etc/config.xml
```

Sample `config.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/config.xsd">
    <default>
        <vendor_section>
            <general_group>
                <enable>1</enable> <!-- 1 = Yes, 0 = No -->
                <custom_text>Hello Magento!</custom_text>
            </general_group>
        </vendor_section>
    </default>
</config>
```

This sets **default values** for your fields.

---

## ♻️ Step 3: Flush Magento Cache

After creating `system.xml` and `config.xml`, you MUST flush the cache so Magento picks up the new settings.

Run:

```bash
php bin/magento cache:flush
php bin/magento setup:upgrade
```

🔵 **Tip**: If your field is still not showing, double-check:

- File paths
- XML structure
- Permissions
- Correct `tab` and `resource` values

---

## 📥 Step 4: Get Value From Configuration

You can **fetch the value** you saved in `system.xml` like this:

---

### 4.1 Simple Calling:

```php
$enable = $this->scopeConfig->getValue('vendor_section/general_group/enable', \Magento\Store\Model\ScopeInterface::SCOPE_STORE);

$customText = $this->scopeConfig->getValue('vendor_section/general_group/custom_text', \Magento\Store\Model\ScopeInterface::SCOPE_STORE);
```

- `vendor_section` = Section ID
- `general_group` = Group ID
- `enable` or `custom_text` = Field ID

---

### 4.2 Create a Helper File (Recommended)

It’s **better practice** to create a Helper class for fetching settings.

Create:

```plaintext
app/code/Vendor/Module/Helper/Data.php
```

Sample Helper:

```php
<?php

namespace Vendor\Module\Helper;

use Magento\Framework\App\Helper\AbstractHelper;
use Magento\Store\Model\ScopeInterface;

class Data extends AbstractHelper
{
    const XML_PATH_ENABLE = 'vendor_section/general_group/enable';
    const XML_PATH_CUSTOM_TEXT = 'vendor_section/general_group/custom_text';

    public function isEnabled()
    {
        return $this->scopeConfig->isSetFlag(self::XML_PATH_ENABLE, ScopeInterface::SCOPE_STORE);
    }

    public function getCustomText()
    {
        return $this->scopeConfig->getValue(self::XML_PATH_CUSTOM_TEXT, ScopeInterface::SCOPE_STORE);
    }
}
```

Now, simply inject the helper and use:

```php
protected $helper;

public function __construct(
    \Vendor\Module\Helper\Data $helper
) {
    $this->helper = $helper;
}

// Usage
if ($this->helper->isEnabled()) {
    echo $this->helper->getCustomText();
}
```

---

## 🎯 Conclusion

And that's it!  
Now you have your own **Magento 2 Admin Configuration Settings** working perfectly with proper structure, default values, and helper methods.

---

# 🔥 Bonus Tips

- Always use `scope="store"` in `system.xml` unless you have a strong reason not to.
- If you need a **dropdown** with custom options, create a **source model**.
- Don't forget to manage ACL (permissions) if you want fine-grained access control!
