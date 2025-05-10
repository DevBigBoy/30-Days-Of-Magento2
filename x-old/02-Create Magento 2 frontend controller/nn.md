To implement CRUD operations in Magento 2.4 for two models, such as `News` and `Categories`, you’ll need to follow the process for creating routes, controllers, models, and views. Below is a detailed explanation of how to implement these models with all CRUD operations.

### Step-by-Step Guide for `News` and `Categories` CRUD Operations

#### 1. **Folder Structure**

The folder structure for your module should look like this:

```
app/code/Croco/NewsModule/
├── Block/
├── Controller/
│   ├── Adminhtml/
│   │   ├── News/
│   │   ├── Category/
│   ├── Index/
├── etc/
│   ├── adminhtml/
│   │   ├── routes.xml
│   ├── frontend/
│   │   ├── routes.xml
├── Model/
├── view/
│   ├── adminhtml/
│   │   ├── templates/
│   │   ├── layout/
│   ├── frontend/
│   │   ├── templates/
│   │   ├── layout/
```

#### 2. **Create Database Setup for `News` and `Categories`**

You’ll need to create an `InstallSchema.php` file to define the database structure for both `news` and `categories` tables.

```php
<?php
// app/code/Croco/NewsModule/Setup/InstallSchema.php

namespace Croco\NewsModule\Setup;

use Magento\Framework\Setup\InstallSchemaInterface;
use Magento\Framework\Setup\ModuleContextInterface;
use Magento\Framework\Setup\SchemaSetupInterface;
use Magento\Framework\DB\Ddl\Table;

class InstallSchema implements InstallSchemaInterface
{
    public function install(SchemaSetupInterface $setup, ModuleContextInterface $context)
    {
        $installer = $setup;
        $installer->startSetup();

        // Create the news table
        if (!$installer->tableExists('news')) {
            $table = $installer->getConnection()->newTable(
                $installer->getTable('news')
            )->addColumn(
                'news_id',
                Table::TYPE_INTEGER,
                null,
                ['identity' => true, 'nullable' => false, 'primary' => true],
                'News ID'
            )->addColumn(
                'title',
                Table::TYPE_TEXT,
                255,
                ['nullable' => false],
                'News Title'
            )->addColumn(
                'content',
                Table::TYPE_TEXT,
                '64k',
                ['nullable' => false],
                'News Content'
            )->addColumn(
                'status',
                Table::TYPE_INTEGER,
                null,
                ['nullable' => false, 'default' => 1],
                'Status'
            )->setComment('News Table');
            $installer->getConnection()->createTable($table);
        }

        // Create the categories table
        if (!$installer->tableExists('categories')) {
            $table = $installer->getConnection()->newTable(
                $installer->getTable('categories')
            )->addColumn(
                'category_id',
                Table::TYPE_INTEGER,
                null,
                ['identity' => true, 'nullable' => false, 'primary' => true],
                'Category ID'
            )->addColumn(
                'name',
                Table::TYPE_TEXT,
                255,
                ['nullable' => false],
                'Category Name'
            )->addColumn(
                'description',
                Table::TYPE_TEXT,
                '64k',
                ['nullable' => true],
                'Category Description'
            )->addColumn(
                'status',
                Table::TYPE_INTEGER,
                null,
                ['nullable' => false, 'default' => 1],
                'Status'
            )->setComment('Categories Table');
            $installer->getConnection()->createTable($table);
        }

        $installer->endSetup();
    }
}
```

#### 3. **Define Routes for Both Models**

##### Frontend Routes (`etc/frontend/routes.xml`):

```xml
<!-- app/code/Croco/NewsModule/etc/frontend/routes.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="news" frontName="news">
            <module name="Croco_NewsModule"/>
        </route>
        <route id="category" frontName="category">
            <module name="Croco_NewsModule"/>
        </route>
    </router>
</config>
```

##### Admin Routes (`etc/adminhtml/routes.xml`):

```xml
<!-- app/code/Croco/NewsModule/etc/adminhtml/routes.xml -->
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="admin">
        <route id="news" frontName="news">
            <module name="Croco_NewsModule"/>
        </route>
        <route id="category" frontName="category">
            <module name="Croco_NewsModule"/>
        </route>
    </router>
</config>
```

#### 4. **Create Models and Resource Models**

##### `News` Model

```php
<?php
// app/code/Croco/NewsModule/Model/News.php

namespace Croco\NewsModule\Model;

use Magento\Framework\Model\AbstractModel;

class News extends AbstractModel
{
    protected function _construct()
    {
        $this->_init(\Croco\NewsModule\Model\ResourceModel\News::class);
    }
}
```

##### `Categories` Model

```php
<?php
// app/code/Croco/NewsModule/Model/Category.php

namespace Croco\NewsModule\Model;

use Magento\Framework\Model\AbstractModel;

class Category extends AbstractModel
{
    protected function _construct()
    {
        $this->_init(\Croco\NewsModule\Model\ResourceModel\Category::class);
    }
}
```

##### Resource Models for `News` and `Categories`

```php
<?php
// app/code/Croco/NewsModule/Model/ResourceModel/News.php

namespace Croco\NewsModule\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class News extends AbstractDb
{
    protected function _construct()
    {
        $this->_init('news', 'news_id');
    }
}
```

```php
<?php
// app/code/Croco/NewsModule/Model/ResourceModel/Category.php

namespace Croco\NewsModule\Model\ResourceModel;

use Magento\Framework\Model\ResourceModel\Db\AbstractDb;

class Category extends AbstractDb
{
    protected function _construct()
    {
        $this->_init('categories', 'category_id');
    }
}
```

##### Collection Models for `News` and `Categories`

```php
<?php
// app/code/Croco/NewsModule/Model/ResourceModel/News/Collection.php

namespace Croco\NewsModule\Model\ResourceModel\News;

use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;

class Collection extends AbstractCollection
{
    protected $_idFieldName = 'news_id';
    protected function _construct()
    {
        $this->_init('Croco\NewsModule\Model\News', 'Croco\NewsModule\Model\ResourceModel\News');
    }
}
```

```php
<?php
// app/code/Croco/NewsModule/Model/ResourceModel/Category/Collection.php

namespace Croco\NewsModule\Model\ResourceModel\Category;

use Magento\Framework\Model\ResourceModel\Db\Collection\AbstractCollection;

class Collection extends AbstractCollection
{
    protected $_idFieldName = 'category_id';
    protected function _construct()
    {
        $this->_init('Croco\NewsModule\Model\Category', 'Croco\NewsModule\Model\ResourceModel\Category');
    }
}
```

#### 5. **Create CRUD Controllers for Admin**

For both `News` and `Category` models, create CRUD controllers in the `Adminhtml` folder. Let's take the example of creating controllers for `News`. Similarly, you can create CRUD operations for `Categories`.

##### `Adminhtml/News/NewAction.php`

```php
<?php
// app/code/Croco/NewsModule/Controller/Adminhtml/News/NewAction.php

namespace Croco\NewsModule\Controller\Adminhtml\News;

use Magento\Backend\App\Action;

class NewAction extends Action
{
    public function execute()
    {
        // Redirect to the edit page for adding new news
        $this->_forward('edit');
    }
}
```

##### `Adminhtml/News/Edit.php`

```php
<?php
// app/code/Croco/NewsModule/Controller/Adminhtml/News/Edit.php

namespace Croco\NewsModule\Controller\Adminhtml\News;

use Magento\Backend\App\Action;
use Croco\NewsModule\Model\NewsFactory;
use Magento\Framework\View\Result\PageFactory;

class Edit extends Action
{
    protected $newsFactory;
    protected $resultPageFactory;

    public function __construct(
        Action\Context $context,
        NewsFactory $newsFactory,
        PageFactory $resultPageFactory
    ) {
        parent::__construct($context);
        $this->newsFactory = $newsFactory;
        $this->resultPageFactory = $resultPageFactory;
    }

    public function execute()
    {
        $newsId = $this->getRequest()->getParam('id');
        $news = $this->newsFactory->create();

        if ($newsId) {
            $news->load($newsId);
            if (!$news->getId()) {
                $this->

messageManager->addErrorMessage(__('This news no longer exists.'));
                return $this->_redirect('*/*/');
            }
        }

        $this->_getSession()->setFormData($news->getData());

        $resultPage = $this->resultPageFactory->create();
        $resultPage->getConfig()->getTitle()->prepend($newsId ? __('Edit News') : __('New News'));

        return $resultPage;
    }
}
```

##### `Adminhtml/News/Save.php`

```php
<?php
// app/code/Croco/NewsModule/Controller/Adminhtml/News/Save.php

namespace Croco\NewsModule\Controller\Adminhtml\News;

use Magento\Backend\App\Action;
use Croco\NewsModule\Model\NewsFactory;

class Save extends Action
{
    protected $newsFactory;

    public function __construct(
        Action\Context $context,
        NewsFactory $newsFactory
    ) {
        parent::__construct($context);
        $this->newsFactory = $newsFactory;
    }

    public function execute()
    {
        $data = $this->getRequest()->getPostValue();

        if (!$data) {
            return $this->_redirect('*/*/');
        }

        $news = $this->newsFactory->create();

        if (isset($data['news_id']) && $data['news_id']) {
            $news->load($data['news_id']);
        }

        try {
            $news->setData($data);
            $news->save();
            $this->messageManager->addSuccessMessage(__('News has been saved.'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->_redirect('*/*/');
    }
}
```

##### `Adminhtml/News/Delete.php`

```php
<?php
// app/code/Croco/NewsModule/Controller/Adminhtml/News/Delete.php

namespace Croco\NewsModule\Controller\Adminhtml\News;

use Magento\Backend\App\Action;
use Croco\NewsModule\Model\NewsFactory;

class Delete extends Action
{
    protected $newsFactory;

    public function __construct(
        Action\Context $context,
        NewsFactory $newsFactory
    ) {
        parent::__construct($context);
        $this->newsFactory = $newsFactory;
    }

    public function execute()
    {
        $newsId = $this->getRequest()->getParam('id');

        try {
            $news = $this->newsFactory->create()->load($newsId);
            if (!$news->getId()) {
                throw new \Magento\Framework\Exception\LocalizedException(__('This news no longer exists.'));
            }

            $news->delete();
            $this->messageManager->addSuccessMessage(__('News has been deleted.'));
        } catch (\Exception $e) {
            $this->messageManager->addErrorMessage($e->getMessage());
        }

        return $this->_redirect('*/*/');
    }
}
```

Repeat similar CRUD operations for the `Category` model.

#### 6. **Frontend Views**

You can create `.phtml` templates and corresponding layout files for the frontend display of `News` and `Categories`.

##### Example: List News Articles (`joblist.phtml`)

```php
<!-- app/code/Croco/NewsModule/view/frontend/templates/joblist.phtml -->
<h1>News Articles</h1>
<ul>
    <?php foreach ($block->getNews() as $news): ?>
        <li>
            <h2><?= $news->getTitle(); ?></h2>
            <p><?= $news->getContent(); ?></p>
        </li>
    <?php endforeach; ?>
</ul>
```

You will also need to create a block for fetching the `News` data and loading it in the template.

With this setup, you will have full CRUD routes and operations for both `News` and `Categories` models, including frontend and backend functionality.