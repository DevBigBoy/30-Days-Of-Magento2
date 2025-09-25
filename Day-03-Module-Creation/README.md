# Day 03: Module Creation

## Overview

Modules are the fundamental building blocks of Magento 2. This guide covers everything you need to know about creating, registering, and managing custom modules.

## Table of Contents

1. [What is a Module?](#what-is-a-module)
2. [Module Structure](#module-structure)
3. [Creating Your First Module](#creating-your-first-module)
4. [Module Registration](#module-registration)
5. [Module Declaration](#module-declaration)
6. [Module Dependencies](#module-dependencies)
7. [Enabling and Disabling Modules](#enabling-and-disabling-modules)
8. [Best Practices](#best-practices)
9. [Practical Examples](#practical-examples)

## What is a Module?

A **module** is a structural element of Magento 2 that contains all the logic related to a specific business feature. Magento 2 itself is built from modules - the core functionality is divided into over 200 modules.

### Module Characteristics

- **Self-contained**: Contains all code for a specific feature
- **Reusable**: Can be installed in different Magento instances
- **Modular**: Can be enabled/disabled independently
- **Upgradeable**: Supports versioning and upgrades
- **Configurable**: Can have dependencies on other modules

### Types of Modules

1. **Core Modules**: Located in `vendor/magento/module-*`
2. **Custom Modules**: Located in `app/code/{Vendor}/{ModuleName}`
3. **Third-party Modules**: Installed via Composer in `vendor/`

## Module Structure

A typical Magento 2 module follows this directory structure:

```
app/code/Vendor/ModuleName/
├── Api/                          # Service contracts (interfaces)
│   ├── Data/                     # Data interfaces
│   └── ProductRepositoryInterface.php
├── Block/                        # Block classes (View layer)
│   └── Product/
│       └── View.php
├── Console/                      # CLI commands
│   └── Command/
│       └── CustomCommand.php
├── Controller/                   # Controllers (handle HTTP requests)
│   ├── Adminhtml/               # Admin controllers
│   └── Index/                   # Frontend controllers
│       └── Index.php
├── Cron/                        # Cron job classes
│   └── CustomCron.php
├── etc/                         # Configuration files
│   ├── adminhtml/               # Admin-specific config
│   │   ├── menu.xml
│   │   ├── routes.xml
│   │   └── system.xml
│   ├── frontend/                # Frontend-specific config
│   │   └── routes.xml
│   ├── acl.xml                  # Access Control List
│   ├── config.xml               # Default config values
│   ├── crontab.xml              # Cron configuration
│   ├── di.xml                   # Dependency injection
│   ├── events.xml               # Event observers
│   ├── module.xml               # Module declaration
│   └── webapi.xml               # Web API routes
├── Helper/                      # Helper classes (utility functions)
│   └── Data.php
├── i18n/                        # Translation files
│   └── en_US.csv
├── Model/                       # Business logic layer
│   ├── ResourceModel/           # Database operations
│   │   ├── Product.php
│   │   └── Product/
│   │       └── Collection.php
│   └── Product.php
├── Observer/                    # Event observers
│   └── ProductSaveObserver.php
├── Plugin/                      # Plugins/Interceptors
│   └── ProductPlugin.php
├── Setup/                       # Installation/upgrade scripts
│   ├── InstallData.php
│   ├── InstallSchema.php
│   ├── UpgradeData.php
│   ├── UpgradeSchema.php
│   ├── Patch/                   # Declarative schema patches
│   │   ├── Data/
│   │   └── Schema/
│   └── Recurring.php
├── Test/                        # Unit and integration tests
│   ├── Unit/
│   └── Integration/
├── Ui/                          # UI components
│   └── Component/
├── view/                        # View layer files
│   ├── adminhtml/               # Admin templates/layouts
│   │   ├── layout/
│   │   ├── templates/
│   │   └── ui_component/
│   ├── frontend/                # Frontend templates/layouts
│   │   ├── layout/
│   │   ├── templates/
│   │   ├── web/
│   │   │   ├── css/
│   │   │   ├── js/
│   │   │   └── images/
│   │   └── requirejs-config.js
│   └── base/                    # Shared between admin/frontend
├── composer.json                # Composer package definition
└── registration.php             # Module registration

```

### Required Files

Every module must have these two files:

1. **registration.php** - Registers the module with Magento
2. **etc/module.xml** - Declares the module and its version

## Creating Your First Module

Let's create a simple "Hello World" module step by step.

### Step 1: Create Module Directory Structure

```bash
# Navigate to Magento root
cd /path/to/magento

# Create module directories
mkdir -p app/code/Vendor/HelloWorld/etc
```

**Naming Convention:**
- **Vendor**: Your company/organization name (e.g., `Acme`, `MyCompany`)
- **ModuleName**: The module name in PascalCase (e.g., `HelloWorld`, `ProductReview`)

### Step 2: Create registration.php

Create `app/code/Vendor/HelloWorld/registration.php`:

```php
<?php
/**
 * Copyright © Vendor. All rights reserved.
 */

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Vendor_HelloWorld',
    __DIR__
);
```

**Explanation:**
- `ComponentRegistrar::MODULE` - Indicates this is a module (not theme or language)
- `'Vendor_HelloWorld'` - Module name in `Vendor_ModuleName` format
- `__DIR__` - Current directory path

### Step 3: Create module.xml

Create `app/code/Vendor/HelloWorld/etc/module.xml`:

```xml
<?xml version="1.0"?>
<!--
/**
 * Copyright © Vendor. All rights reserved.
 */
-->
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_HelloWorld" setup_version="1.0.0">
        <!-- Module dependencies go here -->
    </module>
</config>
```

**Explanation:**
- `name="Vendor_HelloWorld"` - Module name (must match registration.php)
- `setup_version="1.0.0"` - Module version (deprecated in 2.3+, but still used)

### Step 4: Enable the Module

```bash
# Enable the module
bin/magento module:enable Vendor_HelloWorld

# Run setup upgrade to register the module
bin/magento setup:upgrade

# Clear cache
bin/magento cache:flush

# Verify module is enabled
bin/magento module:status Vendor_HelloWorld
```

**Expected Output:**
```
Module is enabled
```

## Module Registration

The `registration.php` file registers the module with Magento's component registration system.

### Registration File Template

```php
<?php
/**
 * Copyright © [Year] [Vendor]. All rights reserved.
 */

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,      // Component type
    'Vendor_ModuleName',              // Component name
    __DIR__                           // Component path
);
```

### Component Types

Magento supports multiple component types:

```php
ComponentRegistrar::MODULE    // For modules
ComponentRegistrar::THEME     // For themes
ComponentRegistrar::LANGUAGE  // For language packs
ComponentRegistrar::LIBRARY   // For libraries
```

### How Registration Works

1. Magento scans the following directories during bootstrap:
   - `app/code/*/*/registration.php`
   - `vendor/*/*/registration.php`
   - `app/design/*/*/*/registration.php`

2. Each `registration.php` file registers its component

3. The component list is cached for performance

### Common Registration Errors

❌ **Wrong module name format:**
```php
// WRONG
ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'VendorHelloWorld',  // Missing underscore
    __DIR__
);
```

✅ **Correct:**
```php
ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Vendor_HelloWorld',  // Correct format
    __DIR__
);
```

## Module Declaration

The `etc/module.xml` file declares the module's metadata and dependencies.

### Basic module.xml

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_ModuleName" setup_version="1.0.0"/>
</config>
```

### Module Version

**Note**: In Magento 2.3+, `setup_version` is deprecated in favor of `composer.json` version. However, it's still commonly used.

```xml
<module name="Vendor_ModuleName" setup_version="1.0.0"/>
```

**Version Format**: `MAJOR.MINOR.PATCH`
- **MAJOR**: Incompatible API changes
- **MINOR**: Backward-compatible functionality additions
- **PATCH**: Backward-compatible bug fixes

### Complete module.xml Example

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_CustomModule" setup_version="1.2.5">
        <sequence>
            <module name="Magento_Catalog"/>
            <module name="Magento_Customer"/>
            <module name="Vendor_AnotherModule"/>
        </sequence>
    </module>
</config>
```

## Module Dependencies

Dependencies ensure modules load in the correct order.

### Declaring Dependencies

Use the `<sequence>` node in `module.xml`:

```xml
<module name="Vendor_CustomModule">
    <sequence>
        <module name="Magento_Catalog"/>
        <module name="Magento_Customer"/>
    </sequence>
</module>
```

**What this means:**
- `Vendor_CustomModule` depends on `Magento_Catalog` and `Magento_Customer`
- Those modules will be loaded before `Vendor_CustomModule`
- If any dependency is missing or disabled, this module won't load

### Types of Dependencies

#### 1. Hard Dependencies (module.xml)

```xml
<sequence>
    <module name="Magento_Catalog"/>
</sequence>
```
- Module won't work without this dependency
- Used for modules you extend or modify

#### 2. Soft Dependencies (composer.json)

```json
{
    "suggest": {
        "magento/module-catalog": "For product features"
    }
}
```
- Module can work without it
- Used for optional integrations

### Dependency Example

If you're creating a module that adds custom attributes to products:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_ProductAttributes" setup_version="1.0.0">
        <sequence>
            <!-- Required because we're extending product functionality -->
            <module name="Magento_Catalog"/>
            <module name="Magento_Eav"/>
        </sequence>
    </module>
</config>
```

### Circular Dependencies

❌ **AVOID circular dependencies:**

Module A depends on Module B, and Module B depends on Module A.

This will cause errors. Always ensure one-way dependency chains.

## Enabling and Disabling Modules

### Check Module Status

```bash
# Check specific module
bin/magento module:status Vendor_ModuleName

# List all modules
bin/magento module:status

# List only enabled modules
bin/magento module:status --enabled

# List only disabled modules
bin/magento module:status --disabled
```

### Enable Module

```bash
# Enable single module
bin/magento module:enable Vendor_ModuleName

# Enable multiple modules
bin/magento module:enable Vendor_ModuleOne Vendor_ModuleTwo

# Enable and run setup upgrade
bin/magento module:enable Vendor_ModuleName && bin/magento setup:upgrade
```

### Disable Module

```bash
# Disable single module
bin/magento module:disable Vendor_ModuleName

# Disable multiple modules
bin/magento module:disable Vendor_ModuleOne Vendor_ModuleTwo

# Disable and clear generated code
bin/magento module:disable Vendor_ModuleName && bin/magento setup:upgrade
```

### Clear Cache After Enable/Disable

```bash
# Clear all cache
bin/magento cache:flush

# Or specific cache types
bin/magento cache:clean config
```

### Module Status in Configuration

Module status is stored in `app/etc/config.php`:

```php
<?php
return [
    'modules' => [
        'Magento_AdminAnalytics' => 1,  // Enabled
        'Magento_Store' => 1,           // Enabled
        'Vendor_HelloWorld' => 1,       // Enabled
        'Vendor_DisabledModule' => 0,   // Disabled
    ]
];
```

**Values:**
- `1` = Enabled
- `0` = Disabled

### Manual Enable/Disable

You can manually edit `app/etc/config.php`:

```php
'modules' => [
    'Vendor_ModuleName' => 1,  // Change to 0 to disable
]
```

Then run:
```bash
bin/magento setup:upgrade
bin/magento cache:flush
```

## Best Practices

### 1. Naming Conventions

✅ **Do:**
```
Vendor: Acme (company name)
Module: ProductReview (PascalCase, descriptive)
Full name: Acme_ProductReview
```

❌ **Don't:**
```
Vendor: mycompany (lowercase)
Module: pr (too short)
Full name: MyCompany-ProductReview (wrong separator)
```

### 2. Module Scope

**Keep modules focused:**
- ✅ One module for product reviews
- ✅ One module for custom shipping method
- ❌ One module for "everything custom"

### 3. File Organization

**Organize files by responsibility:**
```
✅ app/code/Vendor/ProductReview/
   ├── Block/Review/
   ├── Controller/Review/
   └── Model/Review/

❌ app/code/Vendor/ProductReview/
   ├── AllBlocks.php
   └── AllControllers.php
```

### 4. Documentation

**Always include:**
- Copyright notice in files
- README.md with module description
- Comments for complex logic

### 5. Version Management

**Update version when:**
- Adding database changes
- Modifying module structure
- Making breaking changes

### 6. Dependencies

**Only declare necessary dependencies:**
- Include in `<sequence>` only modules you directly use
- Don't declare transitive dependencies

### 7. Backward Compatibility

**Maintain compatibility:**
- Don't remove public methods
- Don't change method signatures
- Use deprecation notices before removal

## Practical Examples

### Example 1: Simple Custom Module

**Purpose**: Add a "Featured" flag to products

```bash
# Create module structure
mkdir -p app/code/Acme/FeaturedProduct/etc
```

**app/code/Acme/FeaturedProduct/registration.php**:
```php
<?php
use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Acme_FeaturedProduct',
    __DIR__
);
```

**app/code/Acme/FeaturedProduct/etc/module.xml**:
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Acme_FeaturedProduct" setup_version="1.0.0">
        <sequence>
            <module name="Magento_Catalog"/>
        </sequence>
    </module>
</config>
```

**Enable:**
```bash
bin/magento module:enable Acme_FeaturedProduct
bin/magento setup:upgrade
bin/magento cache:flush
```

### Example 2: Module with Multiple Dependencies

**Purpose**: Custom checkout that depends on multiple modules

**app/code/Acme/CustomCheckout/etc/module.xml**:
```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Acme_CustomCheckout" setup_version="1.0.0">
        <sequence>
            <module name="Magento_Checkout"/>
            <module name="Magento_Quote"/>
            <module name="Magento_Sales"/>
            <module name="Magento_Customer"/>
            <module name="Magento_Payment"/>
        </sequence>
    </module>
</config>
```

### Example 3: Module with Composer

**app/code/Acme/BlogModule/composer.json**:
```json
{
    "name": "acme/module-blog",
    "description": "Blog functionality for Magento 2",
    "type": "magento2-module",
    "version": "1.0.0",
    "license": "proprietary",
    "authors": [
        {
            "name": "Acme Developer",
            "email": "developer@acme.com"
        }
    ],
    "require": {
        "php": "~8.1.0||~8.2.0||~8.3.0",
        "magento/framework": "103.0.*",
        "magento/module-cms": "104.0.*",
        "magento/module-store": "101.1.*"
    },
    "autoload": {
        "files": [
            "registration.php"
        ],
        "psr-4": {
            "Acme\\BlogModule\\": ""
        }
    }
}
```

## Module Creation Checklist

Before deploying your module, verify:

- [ ] `registration.php` created with correct module name
- [ ] `etc/module.xml` created with correct version
- [ ] Module dependencies declared in `<sequence>`
- [ ] Module enabled via CLI
- [ ] `setup:upgrade` executed successfully
- [ ] No errors in logs (`var/log/`)
- [ ] Module appears in admin: Stores → Configuration → Advanced → Advanced
- [ ] Cache cleared and site works correctly

## Troubleshooting

### Module Not Showing Up

**Check:**
```bash
# 1. Verify file paths are correct
ls -la app/code/Vendor/ModuleName/

# 2. Check registration.php exists
cat app/code/Vendor/ModuleName/registration.php

# 3. Check module.xml exists
cat app/code/Vendor/ModuleName/etc/module.xml

# 4. Run module discovery
bin/magento module:status Vendor_ModuleName
```

### Module Enabled But Not Working

```bash
# 1. Check for errors
tail -f var/log/system.log
tail -f var/log/exception.log

# 2. Recompile
bin/magento setup:di:compile

# 3. Clear all caches
bin/magento cache:flush
rm -rf generated/* var/cache/* var/page_cache/*

# 4. Upgrade
bin/magento setup:upgrade
```

### Dependency Errors

If you see "Module X depends on disabled module Y":

```bash
# Enable the dependency first
bin/magento module:enable Vendor_DependencyModule

# Then enable your module
bin/magento module:enable Vendor_YourModule

# Run upgrade
bin/magento setup:upgrade
```

## Next Steps

- [Day 04: Dependency Injection](../Day-04-Dependency-Injection/README.md)
- Learn about routing and controllers
- Create your first custom functionality

## Additional Resources

- [Magento DevDocs: Build a Module](https://devdocs.magento.com/videos/fundamentals/create-a-new-module/)
- [Module Development Guide](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/bk-extension-dev-guide.html)
- [Component Registration](https://devdocs.magento.com/guides/v2.4/extension-dev-guide/build/component-registration.html)
- [Module Dependencies](https://devdocs.magento.com/guides/v2.4/architecture/archi_perspectives/components/modules/mod_depend.html)
