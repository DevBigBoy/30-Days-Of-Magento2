# Complete Module Example Files

This document provides complete, ready-to-use files for creating a Magento 2 module.

## Example 1: Hello World Module

### Directory Structure

```
app/code/Vendor/HelloWorld/
├── etc/
│   └── module.xml
└── registration.php
```

### registration.php

**File**: `app/code/Vendor/HelloWorld/registration.php`

```php
<?php
/**
 * Copyright © Vendor. All rights reserved.
 * See COPYING.txt for license details.
 */

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Vendor_HelloWorld',
    __DIR__
);
```

### module.xml

**File**: `app/code/Vendor/HelloWorld/etc/module.xml`

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Vendor. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_HelloWorld" setup_version="1.0.0"/>
</config>
```

### Installation Commands

```bash
# Navigate to Magento root
cd /path/to/magento

# Create directories
mkdir -p app/code/Vendor/HelloWorld/etc

# Create files (paste content above)
# Then enable module
bin/magento module:enable Vendor_HelloWorld
bin/magento setup:upgrade
bin/magento cache:flush
```

## Example 2: Module with Dependencies

### Directory Structure

```
app/code/Acme/ProductExtension/
├── etc/
│   └── module.xml
├── registration.php
└── composer.json
```

### registration.php

**File**: `app/code/Acme/ProductExtension/registration.php`

```php
<?php
/**
 * Acme Product Extension Module
 *
 * @category  Acme
 * @package   Acme_ProductExtension
 * @author    Acme Developer
 * @copyright Copyright (c) 2025 Acme Corp
 */

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Acme_ProductExtension',
    __DIR__
);
```

### module.xml with Dependencies

**File**: `app/code/Acme/ProductExtension/etc/module.xml`

```xml
<?xml version="1.0"?>
<!--
/**
 * Acme Product Extension Module
 *
 * @category  Acme
 * @package   Acme_ProductExtension
 * @author    Acme Developer
 * @copyright Copyright (c) 2025 Acme Corp
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Acme_ProductExtension" setup_version="1.0.0">
        <sequence>
            <!-- Core modules this module depends on -->
            <module name="Magento_Catalog"/>
            <module name="Magento_Eav"/>
            <module name="Magento_Backend"/>
        </sequence>
    </module>
</config>
```

### composer.json

**File**: `app/code/Acme/ProductExtension/composer.json`

```json
{
    "name": "acme/module-product-extension",
    "description": "Extends Magento product functionality",
    "type": "magento2-module",
    "version": "1.0.0",
    "license": [
        "proprietary"
    ],
    "authors": [
        {
            "name": "Acme Developer",
            "email": "developer@acme.com",
            "homepage": "https://www.acme.com",
            "role": "Developer"
        }
    ],
    "require": {
        "php": "~8.1.0||~8.2.0||~8.3.0",
        "magento/framework": "103.0.*",
        "magento/module-catalog": "104.0.*",
        "magento/module-eav": "102.1.*",
        "magento/module-backend": "102.0.*"
    },
    "autoload": {
        "files": [
            "registration.php"
        ],
        "psr-4": {
            "Acme\\ProductExtension\\": ""
        }
    }
}
```

## Example 3: Complete Module with Controller

### Directory Structure

```
app/code/Acme/CustomPage/
├── Controller/
│   └── Index/
│       └── Index.php
├── etc/
│   ├── frontend/
│   │   └── routes.xml
│   └── module.xml
├── view/
│   └── frontend/
│       ├── layout/
│       │   └── customppage_index_index.xml
│       └── templates/
│           └── page.phtml
├── composer.json
└── registration.php
```

### registration.php

**File**: `app/code/Acme/CustomPage/registration.php`

```php
<?php
/**
 * Copyright © Acme. All rights reserved.
 */

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Acme_CustomPage',
    __DIR__
);
```

### module.xml

**File**: `app/code/Acme/CustomPage/etc/module.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Acme_CustomPage" setup_version="1.0.0"/>
</config>
```

### routes.xml

**File**: `app/code/Acme/CustomPage/etc/frontend/routes.xml`

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Acme. All rights reserved.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="custompage" frontName="custompage">
            <module name="Acme_CustomPage"/>
        </route>
    </router>
</config>
```

### Controller

**File**: `app/code/Acme/CustomPage/Controller/Index/Index.php`

```php
<?php
/**
 * Copyright © Acme. All rights reserved.
 */

namespace Acme\CustomPage\Controller\Index;

use Magento\Framework\App\Action\Action;
use Magento\Framework\App\Action\Context;
use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\View\Result\PageFactory;

/**
 * Custom Page Controller
 */
class Index extends Action implements HttpGetActionInterface
{
    /**
     * @var PageFactory
     */
    protected $resultPageFactory;

    /**
     * Constructor
     *
     * @param Context $context
     * @param PageFactory $resultPageFactory
     */
    public function __construct(
        Context $context,
        PageFactory $resultPageFactory
    ) {
        $this->resultPageFactory = $resultPageFactory;
        parent::__construct($context);
    }

    /**
     * Execute action
     *
     * @return \Magento\Framework\View\Result\Page
     */
    public function execute()
    {
        $resultPage = $this->resultPageFactory->create();
        $resultPage->getConfig()->getTitle()->set(__('Custom Page'));
        return $resultPage;
    }
}
```

### Layout XML

**File**: `app/code/Acme/CustomPage/view/frontend/layout/custompage_index_index.xml`

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Acme. All rights reserved.
 */
-->
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <head>
        <title>Custom Page</title>
    </head>
    <body>
        <referenceContainer name="content">
            <block class="Magento\Framework\View\Element\Template"
                   name="custom.page"
                   template="Acme_CustomPage::page.phtml"/>
        </referenceContainer>
    </body>
</page>
```

### Template

**File**: `app/code/Acme/CustomPage/view/frontend/templates/page.phtml`

```php
<?php
/**
 * Copyright © Acme. All rights reserved.
 *
 * @var \Magento\Framework\View\Element\Template $block
 */
?>
<div class="custom-page">
    <h1><?= $block->escapeHtml(__('Welcome to Custom Page')) ?></h1>
    <p><?= $block->escapeHtml(__('This is a custom page created by Acme_CustomPage module.')) ?></p>
    <p><?= $block->escapeHtml(__('URL: /custompage')) ?></p>
</div>
```

### composer.json

**File**: `app/code/Acme/CustomPage/composer.json`

```json
{
    "name": "acme/module-custom-page",
    "description": "Custom Page Module",
    "type": "magento2-module",
    "version": "1.0.0",
    "license": "proprietary",
    "require": {
        "php": "~8.1.0||~8.2.0||~8.3.0",
        "magento/framework": "103.0.*"
    },
    "autoload": {
        "files": [
            "registration.php"
        ],
        "psr-4": {
            "Acme\\CustomPage\\": ""
        }
    }
}
```

### Installation & Testing

```bash
# Enable module
bin/magento module:enable Acme_CustomPage

# Run setup upgrade
bin/magento setup:upgrade

# Compile (if in production mode)
bin/magento setup:di:compile

# Deploy static content (if in production mode)
bin/magento setup:static-content:deploy -f

# Clear cache
bin/magento cache:flush

# Access the page
# URL: http://your-magento-site.com/custompage
```

## Example 4: Module with Admin Menu

### Additional Files Needed

```
app/code/Acme/AdminModule/
├── etc/
│   ├── adminhtml/
│   │   ├── menu.xml
│   │   └── routes.xml
│   ├── acl.xml
│   └── module.xml
└── ...
```

### menu.xml

**File**: `app/code/Acme/AdminModule/etc/adminhtml/menu.xml`

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Acme. All rights reserved.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Backend:etc/menu.xsd">
    <menu>
        <!-- Add new menu item under Content -->
        <add id="Acme_AdminModule::custom_menu"
             title="Custom Menu"
             module="Acme_AdminModule"
             sortOrder="100"
             parent="Magento_Backend::content"
             action="adminmodule/index/index"
             resource="Acme_AdminModule::custom_menu"/>
    </menu>
</config>
```

### acl.xml

**File**: `app/code/Acme/AdminModule/etc/acl.xml`

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Acme. All rights reserved.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="Magento_Backend::content">
                    <resource id="Acme_AdminModule::custom_menu"
                              title="Custom Menu"
                              sortOrder="100"/>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

## Quick Start Template

Use this as a starting point for any new module:

```bash
#!/bin/bash
# Quick module creation script

VENDOR="YourVendor"
MODULE="YourModule"
MODULE_PATH="app/code/${VENDOR}/${MODULE}"

# Create directories
mkdir -p "${MODULE_PATH}/etc"

# Create registration.php
cat > "${MODULE_PATH}/registration.php" << 'EOF'
<?php
use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'VENDOR_MODULE',
    __DIR__
);
EOF

# Create module.xml
cat > "${MODULE_PATH}/etc/module.xml" << 'EOF'
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="VENDOR_MODULE" setup_version="1.0.0"/>
</config>
EOF

# Replace placeholders
sed -i "s/VENDOR_MODULE/${VENDOR}_${MODULE}/g" "${MODULE_PATH}/registration.php"
sed -i "s/VENDOR_MODULE/${VENDOR}_${MODULE}/g" "${MODULE_PATH}/etc/module.xml"

echo "Module created at: ${MODULE_PATH}"
echo "Enable with: bin/magento module:enable ${VENDOR}_${MODULE}"
```

## Testing Your Module

### Verification Script

```bash
#!/bin/bash
# verify-module.sh

MODULE_NAME="Vendor_ModuleName"

echo "Checking module: ${MODULE_NAME}"
echo "================================"

# Check if module is registered
echo -n "Module registered: "
bin/magento module:status ${MODULE_NAME} > /dev/null 2>&1 && echo "✓ YES" || echo "✗ NO"

# Check if module is enabled
echo -n "Module enabled: "
bin/magento module:status ${MODULE_NAME} 2>&1 | grep -q "Module is enabled" && echo "✓ YES" || echo "✗ NO"

# Check for errors in log
echo -n "Errors in log: "
grep -q "${MODULE_NAME}" var/log/system.log 2>/dev/null && echo "⚠ FOUND" || echo "✓ NONE"

# Check in admin
echo -n "Visible in admin: "
bin/magento module:status --enabled | grep -q "${MODULE_NAME}" && echo "✓ YES" || echo "✗ NO"

echo "================================"
```

## Common Module Templates

### Empty Module (Minimal)
- registration.php
- etc/module.xml

### Frontend Module (with routing)
- Add: etc/frontend/routes.xml
- Add: Controller/Index/Index.php
- Add: view/frontend/layout/*.xml
- Add: view/frontend/templates/*.phtml

### Admin Module (with menu)
- Add: etc/adminhtml/routes.xml
- Add: etc/adminhtml/menu.xml
- Add: etc/acl.xml
- Add: Controller/Adminhtml/Index/Index.php

### API Module (with REST/GraphQL)
- Add: Api/Data/*.php (interfaces)
- Add: Api/*RepositoryInterface.php
- Add: etc/webapi.xml
- Add: etc/schema.graphqls

## Next Steps

After creating your module:

1. Add functionality (controllers, models, etc.)
2. Create database tables (if needed)
3. Add configuration options
4. Write tests
5. Create documentation
6. Package for distribution

## Additional Resources

- [Module File Structure](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/build/module-file-structure.html)
- [Create a Component](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/build/create_component.html)
- [Magento Coding Standards](https://developer.adobe.com/commerce/php/coding-standards/)
