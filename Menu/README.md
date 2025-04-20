# Menu

## Step 1: Set Up Your Custom Module

Before creating the admin menu, you need to create a custom module in Magento 2. If you already have a module, you can skip this step. Here's how you can create a basic custom module:

1. **Create the Module Directory Structure:**
   Create the following directories in your Magento installation:

   ```bash
   app/code/Vendor/ModuleName/
   ├── Controller/
   ├── etc/
   ├── Model/
   ├── view/
   ├── registration.php
   └── etc/module.xml
   ```

2. **registration.php:**

   ```php
   <?php
   use Magento\Framework\Component\ComponentRegistrar;

   ComponentRegistrar::register(
       ComponentRegistrar::MODULE,
       'Vendor_ModuleName',
       __DIR__
   );
   ```

3. **module.xml:**

   In `app/code/Vendor/ModuleName/etc/module.xml`, define the module's information:

   ```xml
   <?xml version="1.0"?>
   <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
       <module name="Vendor_ModuleName" setup_version="1.0.0" />
   </config>
   ```

4. **Enable the Module:**
   After creating the module files, you need to enable the module.

   ```bash
   php bin/magento setup:upgrade
   php bin/magento module:enable Vendor_ModuleName
   php bin/magento setup:di:compile
   ```

### Step 2: Define Admin Menu Structure

Now, let’s define the admin menu for this module. Magento 2’s admin menu is defined in an XML file called `adminhtml/menu.xml`.

1. **Create the `menu.xml` File:**

   In `app/code/Vendor/ModuleName/etc/adminhtml/menu.xml`, define the menu structure.

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <menu xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Menu/etc/menu.xsd">
       <add id="Vendor_ModuleName::menu_root"
            title="My Custom Module"
            module="Vendor_ModuleName"
            sortOrder="10"
            action="Vendor_ModuleName::index"
            resource="Vendor_ModuleName::menu_root" />
       <add id="Vendor_ModuleName::sub_menu"
            title="Sub Menu"
            module="Vendor_ModuleName"
            sortOrder="20"
            parent="Vendor_ModuleName::menu_root"
            action="Vendor_ModuleName::subindex"
            resource="Vendor_ModuleName::sub_menu" />
   </menu>
   ```

   #### Explanation

   - **`id`**: A unique identifier for the menu item.
   - **`title`**: The label that appears in the admin panel.
   - **`module`**: The module name for this menu item.
   - **`sortOrder`**: Defines the order of the menu items.
   - **`action`**: The controller action that is invoked when the menu item is clicked.
   - **`parent`**: The parent menu item (if the menu item is a sub-menu).
   - **`resource`**: Defines the ACL (Access Control List) permissions required to view the menu item.

2. **Create the Controller Action:**

   You need to define controller actions to handle the menu item clicks.

   In `app/code/Vendor/ModuleName/Controller/Adminhtml/Index/Index.php`, create the action for the root menu.

   ```php
   <?php
   namespace Vendor\ModuleName\Controller\Adminhtml\Index;

   use Magento\Backend\App\Action;
   use Magento\Backend\App\Action\Context;

   class Index extends Action
   {
       const ADMIN_RESOURCE = 'Vendor_ModuleName::menu_root';

       protected function _construct()
       {
           $this->_controller = 'adminhtml_index';
           $this->_blockGroup = 'Vendor_ModuleName';
           $this->_viewCategory = 'Adminhtml';
           parent::_construct();
       }

       public function execute()
       {
           $this->_view->loadLayout();
           $this->_view->renderLayout();
       }
   }
   ```

3. **Create a Block for the Page Layout:**

   Create a block that will be rendered when the menu item is clicked. In `app/code/Vendor/ModuleName/Block/Adminhtml/Index/Index.php`:

   ```php
   <?php
   namespace Vendor\ModuleName\Block\Adminhtml\Index;

   use Magento\Backend\Block\Template;

   class Index extends Template
   {
       public function _prepareLayout()
       {
           $this->setTemplate('Vendor_ModuleName::index.phtml');
       }
   }
   ```

4. **Create the Template File:**

   In `app/code/Vendor/ModuleName/view/adminhtml/templates/index.phtml`, create a simple template for the page:

   ```php
   <h1>Welcome to My Custom Admin Page</h1>
   ```

### Step 3: Create ACL for Menu Access Control

In Magento 2, the admin menu items are also controlled by ACL (Access Control List). To ensure that only users with the correct permissions can see the menu, you need to define ACL rules.

1. **Define ACL Rules in `acl.xml`:**

   In `app/code/Vendor/ModuleName/etc/acl.xml`, define the ACL rules:

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <acl xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
       <resources>
           <resource id="Magento_Backend::admin">
               <resource id="Vendor_ModuleName::menu_root" title="My Custom Module" />
               <resource id="Vendor_ModuleName::sub_menu" title="Sub Menu" />
           </resource>
       </resources>
   </acl>
   ```

   #### Explanation

   - The ACL rules ensure that only authorized users can access the menu item based on the permissions defined.
   - The `id` values in the ACL must match the `resource` values defined in the `menu.xml`.

### Step 4: Add Sub-menu Items (Optional)

You can add sub-menu items under the main menu. For instance, if you want to add a "Sub Menu" under "My Custom Module," you would modify the `menu.xml` as shown above in the example.

Sub-menus work the same way as root menu items. The only difference is that they have the `parent` attribute pointing to another menu item.

### Step 5: Run Magento Upgrade and Clear Cache

Once you’ve defined the menu and its resources, run the following commands to upgrade the Magento setup and clear the cache:

```bash
php bin/magento setup:upgrade
php bin/magento cache:flush
```

### Step 6: Accessing the Admin Menu

After completing the above steps, go to your Magento 2 admin panel. You should see "My Custom Module" in the admin menu, and if you click it, it will take you to the `Index` controller action. If you added a sub-menu, you’ll also see it under the main menu item.

### Example of More Complex Menus

If you have multiple sub-menu items, you can nest them like so:

```xml
<add id="Vendor_ModuleName::menu_root"
     title="Main Menu"
     module="Vendor_ModuleName"
     sortOrder="10"
     action="Vendor_ModuleName::index"
     resource="Vendor_ModuleName::menu_root" />

<add id="Vendor_ModuleName::sub_menu_1"
     title="Sub Menu 1"
     parent="Vendor_ModuleName::menu_root"
     action="Vendor_ModuleName::subindex1"
     resource="Vendor_ModuleName::sub_menu_1" />

<add id="Vendor_ModuleName::sub_menu_2"
     title="Sub Menu 2"
     parent="Vendor_ModuleName::menu_root"
     action="Vendor_ModuleName::subindex2"
     resource="Vendor_ModuleName::sub_menu_2" />
```

In this case, when you click "Main Menu," you will have the option to choose between "Sub Menu 1" and "Sub Menu 2."

