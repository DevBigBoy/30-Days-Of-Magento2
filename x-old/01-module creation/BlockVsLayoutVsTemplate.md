# Create Magento 2 frontend Layout - Block - Template

- In Magento 2.4, **block creation**, **layout creation**, and **template creation** are essential concepts for building a fully functioning, customizable front-end. These concepts work together to control what is displayed on the frontend and how it looks. Let’s break them down in a beginner-friendly way, using real examples.

## 1. Block Creation

### What is a Block?

In Magento 2, a block is a PHP class that handles the logic of your view. It acts as a bridge between the layout (XML) and the template (HTML/PHTML). The block contains the logic necessary for preparing the data, which will be used in your template files for rendering content.

#### Steps to Create a Block

##### Example: Creating a Custom Block to Show a Simple Message

A - **Create a Block Class:**
   Now, let’s create a block class that will output the message. In `app/code/Tutorial/HelloWorld/Block/Hello.php`:

   ```php
   <?php
   namespace Tutorial\HelloWorld\Block;

   use Magento\Framework\View\Element\Template;

   class Hello extends Template
   {
       public function getHelloWorldTxt()
       {
           return 'Hello World!';
       }
   }
   ```

   This block contains a simple method, `getHelloWorldTxt()`, which returns the message "Hello World!".

---

### 2. Layout Creation

#### What is a Layout?

A layout in Magento 2 is an XML file that defines the structure of the page. It maps the blocks and templates to specific positions on the page and allows you to control how content is organized and displayed.

- You can associate routes and block thanks to the layout. Its name is very important and must be composed with: RouterName_ControllerName_ActionName

#### Steps to Create a Layout

1. **Create a Layout XML File:**
   In Magento 2, layout files are stored in the `view/frontend/layout` folder of your module.

   Let’s say you want to display your block on the homepage. Create the following layout file in `app/code/Tutorial/HelloWorld/view/frontend/layout/cms_index_index.xml`:

   ```xml
   <?xml version="1.0"?>
   <page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
       <body>
           <referenceContainer name="content">
               <block class="Tutorial\HelloWorld\Block\Hello" name="hello.world.block" template="Tutorial_HelloWorld::hello.phtml"/>
           </referenceContainer>
       </body>
   </page>
   ```

   Explanation:
   - `<referenceContainer name="content">` tells Magento where to place the block. In this case, it is in the `content` section.
   - The block’s class is `Tutorial\HelloWorld\Block\Hello`, and its template is `hello.phtml`.

---

### 3. Template Creation

#### What is a Template?

A template is the actual HTML and PHP file that gets rendered in the browser. It pulls data from the block and formats it for display on the page.

#### Steps to Create a Template

1. **Create a Template File:**
   Now, let’s create the `hello.phtml` template file. The location of template files is specified relative to the `view/frontend/templates` directory.

   In `app/code/Tutorial/HelloWorld/view/frontend/templates/hello.phtml`:

   ```php
   <div>
       <h1><?php echo $block->getHelloWorldTxt(); ?></h1>
   </div>
   ```

   Explanation:
   - `<?php echo $block->getHelloWorldTxt(); ?>` calls the method from the block class to display the "Hello World!" message.

---

### Putting it All Together

Here’s a quick overview of how the pieces work together:

- The **Block (Hello.php)** prepares the data and logic.
- The **Layout (cms_index_index.xml)** maps the block to a specific area (container) in the page.
- The **Template (hello.phtml)** renders the output on the frontend.

When you navigate to the homepage (`cms_index_index.xml` controls the layout of the homepage), the page will show a simple "Hello World!" message.

---

### Testing Your Module

1. **Enable the Module:**
   Run the following commands to enable and register the module in Magento 2.

   ```bash
   bin/magento setup:upgrade
   bin/magento setup:di:compile
   bin/magento cache:flush
   ```

2. **Check the Frontend:**
   Go to your Magento site’s homepage, and you should see the "Hello World!" message displayed.

---

### Conclusion

- **Blocks** handle logic and data retrieval.
- **Layouts** organize blocks and templates on the page.
- **Templates** display the content.

By understanding how blocks, layouts, and templates interact in Magento 2.4, you can customize and build complex frontend functionality step by step.
