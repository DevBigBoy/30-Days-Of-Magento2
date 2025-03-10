# Declare a new Magento 2 module

- Creating a custom module in Magento 2 involves several structured steps to ensure that the module is properly defined, registered, and activated.

- In Magento 2, a module is a standalone unit where we write our functionalities or business logic to achieve our goal.

- It is a group of directories that contain the blocks, controllers, helpers, and models needed to create a specific store feature. It is the unit of customization in the Magento 2 platform.

- You can create Magento 2 modules to perform various functions, such as influencing user experience and changing the storefront appearance. They can install, deleted, or disabled.

## Here’s a detailed guide on how to create a simple custom module.

### Steps to Create a Module :

- Create the module folder.
- Create the registration.php file.
- Create the module.xml file.
- Run the command: php bin/magento setup:upgrade
- Run the command: php bin/magento setup:di:compile

### Create the module Folder :

To create the module we have to create a vendor folder first with the vendor name (which is Bigboy in our case) under the app/code folder.


### Create the registration.php file :

```php
<?php
use \Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'Bigboy_JobManager',
    __DIR__
);
```

### Create the module.xml file. :

To create the module.xml file, we first need to create an etc folder under the directory (app/code/Webkul/Blogmanager), and in the etc folder we need to create a module.xml file with the following content.

- Code for etc/module.xml file

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Bigboy_JobManager">
    </module>
</config>
```
