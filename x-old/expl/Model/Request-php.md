This `Request` model class looks like it's part of the **CrocoIT_Gdpr** module, and it's implementing the `RequestInterface` that you've defined earlier. Let me break this down for you:

### 1. **Class Definition**
```php
class Request extends AbstractModel implements RequestInterface
```
This class extends Magento’s `AbstractModel` and implements the `RequestInterface`. By extending `AbstractModel`, this class inherits key functionality for interacting with Magento's model system (such as CRUD operations and data management). Implementing `RequestInterface` ensures that this model adheres to the contract for accessing and setting data related to a GDPR request.

### 2. **Constructor Method**
```php
protected function _construct()
{
    $this->_init(ResourceModel\Request::class);
}
```
The `protected function _construct()` method initializes the model by telling Magento which **resource model** to use. The resource model (`ResourceModel\Request`) is responsible for handling the database interaction, such as saving, updating, and loading data from the `crocoit_gdpr_request` table.

- **Why use `_init()`**: This is the Magento way of linking a model with its corresponding resource model, ensuring that the model has the necessary methods to interact with the database.

### 3. **Getters and Setters**
These methods manage the **properties** of the `Request` model, which are defined in the `RequestInterface`.

#### `getCustomerId()`
```php
public function getCustomerId()
{
    return $this->getData(self::CUSTOMER_ID);
}
```
This method retrieves the `customer_id` value from the model’s data array.

#### `setCustomerId($value)`
```php
public function setCustomerId($value)
{
    return $this->setData(self::CUSTOMER_ID, $value);
}
```
This method sets the `customer_id` value in the model’s data array. It uses the constant `CUSTOMER_ID` from `RequestInterface` to ensure consistency across the codebase.

#### `getCreatedAt()`
```php
public function getCreatedAt()
{
    return $this->getData(self::CREATED_AT);
}
```
This method retrieves the `created_at` timestamp value. 

#### `getType()`
```php
public function getType()
{
    return $this->getData(self::TYPE);
}
```
This method retrieves the `type` of the GDPR request, which is defined in `RequestInterface` as a constant (`TYPE_ANONYMIZE`, `TYPE_PROVIDE_DATA`, `TYPE_REMOVE`).

#### `setType($value)`
```php
public function setType($value)
{
    return $this->setData(self::TYPE, $value);
}
```
This method sets the `type` value, ensuring that the request type is set correctly (whether it's "anonymize", "provide_data", or "remove").

#### `getStatus()`
```php
public function getStatus()
{
    return $this->getData(self::STATUS);
}
```
This method retrieves the current status of the GDPR request (such as `pending`, `processing`, `completed`, etc.).

#### `setStatus($value)`
```php
public function setStatus($value)
{
    return $this->setData(self::STATUS, $value);
}
```
This method sets the `status` value, allowing updates to the state of the request.

---

### Summary

- **Purpose**: The `Request` class represents a GDPR request record. It is used to manage the various properties of a request, such as the customer ID, type of request, and the request's status.
  
- **Data Management**: The model interacts with data using the `getData()` and `setData()` methods, which manage the actual data values in the model's internal data array. These are part of the `AbstractModel` class, allowing easy management of properties and their persistence in the database.

- **Database Interaction**: The resource model (`ResourceModel\Request`) handles the database interactions, while this class focuses on manipulating and accessing data for the request.

### Clarifications/Things to Check
- **Resource Model**: The `Request` model depends on a resource model (`ResourceModel\Request`), which you would need to define as part of your module. This resource model will handle database CRUD operations for the `crocoit_gdpr_request` table.
  
- **Interface**: The `RequestInterface` defines constants and method signatures, which is useful for ensuring that the model adheres to a contract, especially when dealing with multiple implementations or external integrations.

If you need to see the resource model or have any specific questions on how things work together, feel free to ask!