# Creating a RabbitMQ Module for Magento 2

I'll help you create a Magento 2 module that integrates with RabbitMQ to log product creation events. This is a great way to learn about Magento's message queue framework.

Let's implement this step by step:

## Module Structure Overview

First, let's create the basic structure for your module. We'll name it `Vendor_RabbitMQLogger`.

Now, let's create each file for our module:

## Step 1: Create module registration files

## Step 2: Create the Interface for the Logger

## Step 3: Create the Logger Implementation

## Step 4: Create the Message Publisher

## Step 5: Create the Observer for Product Save Event

## Step 6: Create Console Command to Manually Consume Messages

## Step 7: Configure RabbitMQ Integration

Let's create the necessary configuration files:

## Step 8: Installation and Testing

Now that we have created all the necessary files for our module, let's go through the steps to install and test it:

1. **Install the module:**

```bash
# Copy the module files to your Magento installation
cp -r app/code/Vendor/RabbitMQLogger [your-magento-root]/app/code/

# Enable the module
bin/magento module:enable Vendor_RabbitMQLogger

# Update the database schema
bin/magento setup:upgrade

# Compile the code (Production mode)
bin/magento setup:di:compile

# Deploy static content (Production mode)
bin/magento setup:static-content:deploy

# Clear the cache
bin/magento cache:clean
bin/magento cache:flush
```

2. **Ensure RabbitMQ is configured in your Magento installation:**

Make sure that RabbitMQ is properly configured in your `env.php` file:

```php
'queue' => [
    'amqp' => [
        'host' => 'localhost',
        'port' => '5672',
        'user' => 'guest',
        'password' => 'guest',
        'virtualhost' => '/',
        'ssl' => ''
    ]
],
```

3. **Start the consumer:**

You can start the consumer manually using the command we created:

```bash
bin/magento vendor:rabbitmq:consume-product-log
```

Or use Magento's built-in consumer runner:

```bash
bin/magento queue:consumers:start vendor.product.log.consumer
```

4. **Test the module by creating a product:**

Create a new product in the Magento admin panel or using the API. Check your system log files (typically in `var/log/system.log`) to see if the product creation event was logged.

## How It Works

Let's explain how this module works:

1. When a product is saved in Magento, the `catalog_product_save_after` event is triggered.
2. Our `ProductSaveAfter` observer catches this event and extracts product information.
3. The observer uses the `Publisher` class to publish a message to the `vendor.product.log` topic.
4. The message is sent to RabbitMQ based on the exchange and queue configuration.
5. The consumer processes messages from the queue and calls the `processProductLog` method of our `ProductLogger` class.
6. The `ProductLogger` writes the product information to the Magento log file.

## Key Learning Points

1. **Message Queue Framework**: You've learned how Magento 2 integrates with message queues like RabbitMQ.
2. **Module Structure**: You've created a complete module with all necessary components.
3. **Event Observers**: You've implemented an observer for a Magento event.
4. **Service Contracts**: You've defined and implemented a service contract (interface and implementation).
5. **Console Commands**: You've created a custom console command.
6. **Dependency Injection**: You've used Magento's DI system throughout the module.

Would you like me to explain any particular part of this module in more detail?