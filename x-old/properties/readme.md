#

This file is quite comprehensive, and it implements a lot of functionality related to custom login management for Magento 2. I'll break it down into smaller pieces, explaining each part step by step to help you understand its purpose and structure.

---

## **Overview of the Class**

This class, `CustomLoginManagement`, implements the `CustomLoginManagementInterface` and provides the logic for custom login functionalities such as:

- Customer login
- Account creation
- Sending OTP (One-Time Password) via SMS
- Password reset
- Guest registration and cart handling
- Activating accounts

---

### **Step 1: Class Properties**

The class uses numerous properties, most of which are injected via Dependency Injection (DI). Here's what some of them do:

- **`$customerRegistry`**: Manages customer data securely, especially for retrieving and modifying sensitive attributes.
- **`$responseResultFactory`**: Creates response objects, typically for API responses.
- **`$stringHelper`**: Utility for working with strings (e.g., checking length, trimming).
- **`$customerHelper`**: Custom helper class for additional logic.
- **`$mathRandom`**: Utility for generating random numbers or strings.
- **`$ActivateOtpFactory` and `$activateotpcollectionFactory`**: Factories for managing OTP entities in the database.
- **`$tokenModelFactory` and `$_tokenService`**: Handle customer tokens for authentication.
- **`$scopeConfig`**: Reads configuration values from Magento's system settings.
- **`$storeManager`**: Provides information about the current store and website.
- **`$encryptor`**: Encrypts sensitive data like passwords.

---

### **Step 2: Constructor and Dependency Injection**

The constructor is used to inject dependencies. These dependencies are automatically managed by Magento's DI container.

```php
public function __construct(
    \Magento\Newsletter\Model\SubscriberFactory $subscriberFactory,
    \Magento\Customer\Model\CustomerRegistry $customerRegistry,
    ...
    DateTime $date,
    \Magento\Framework\App\ResourceConnection $resouceConnection,
    CollectionFactory $visitorCollectionFactory = null,
    SessionManagerInterface $sessionManager = null
)
```

- **DI in Action**:
  Magento passes the required dependencies into this class via the constructor. If a dependency is optional (like `$visitorCollectionFactory`), the code uses `ObjectManager` to instantiate it if it's not provided.

---

### **Step 3: Core Functionalities**

#### **3.1 Login Logic: `CustomLogin`**

This method handles customer login based on email or mobile number:

```php
public function CustomLogin($username, $password)
{
    ...
    // Check if username is an email or mobile number
    $validator = new \Zend\Validator\EmailAddress();
    if ($validator->isValid($username)) {
        // Fetch customer by email
        $customer = $this->_customerRepo->get($username);
    } else {
        // Fetch customer by mobile number using SearchCriteria
        $this->_searchBuilder->addFilter('themecafe_mobile', $username);
        $searchCriteria = $this->_searchBuilder->create();
        $resault = $this->_customerRepo->getList($searchCriteria);
        $customer = $resault->getItems()[0];
    }

    // Validate mobile number and verify account activation
    if ($customer->getCustomAttribute('themecafe_mobile_verify')->getValue() == 0) {
        throw new \Magento\Framework\Exception\LocalizedException(
            __('Account not activated.')
        );
    }

    // Generate a token for the customer
    $token = $this->_tokenService->createCustomerAccessToken($customer->getEmail(), $password);
    $customLogin->setToken($token);

    // Set additional login details
    $customLogin->setCartId($this->createEmptyCartQuoteForCustomer($customer->getId())->getId());
    ...
}
```

- **How it Works**:
  - Determines whether the username is an email or mobile number.
  - Fetches the customer using the appropriate identifier.
  - Checks if the account is activated.
  - Generates an access token for the customer and sets login details (cart, token, etc.).

---

#### **3.2 Account Creation: `createAccount`**

This method creates a new customer account and sends an OTP for activation.

```php
public function createAccount($firstname, $lastname, $email, $mobile, $phoneCode, $password, $confirmPassword)
{
    ...
    // Validate email and password
    $this->checkPasswordStrength($password);
    if ($password !== $confirmPassword) {
        throw new \Magento\Framework\Exception\LocalizedException(__('Password mismatch.'));
    }

    // Check if mobile number is already used
    if ($this->checkIsPhoneAddedTocustomer($mobile)) {
        throw new \Magento\Framework\Exception\LocalizedException(__('Phone number already used.'));
    }

    // Create customer object and set attributes
    $customer = $this->customerFactory->create();
    $customer->setEmail($email);
    $customer->setFirstname($firstname);
    $customer->setLastname($lastname);
    $customer->setCustomAttribute('themecafe_mobile', $mobile);

    // Save customer with hashed password
    $hash = $this->createPasswordHash($password);
    $this->_accountManagment->createAccountWithPasswordHash($customer, $hash);

    // Send OTP and save activation details
    $random_number = $this->generatedRandomNumbers();
    $this->callSmsApi($mobile, $random_number);
    ...
}
```

- **How it Works**:
  - Validates inputs like mobile number, email, and password.
  - Checks for duplicate mobile numbers in the database.
  - Creates a new customer with the provided details.
  - Sends an OTP for account activation.

---

#### **3.3 Sending OTP: `callSmsApi`**

This method sends an SMS using a configured vendor (e.g., Cequens or Unifonic).

```php
public function callSmsApi($mobile, $random_number)
{
    $smsMessage = "Verification code: " . $random_number;
    $phoneNumber = '+2' . $mobile;

    // Use Cequens API for SMS
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => 'https://apis.cequens.com/sms/v1/messages',
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'senderName' => 'YourApp',
            'messageType' => 'text',
            'messageText' => $smsMessage,
            'recipients' => $phoneNumber,
        ]),
        CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $authToken],
    ]);
    ...
    $response = curl_exec($ch);
    curl_close($ch);

    if ($response['success']) {
        $ActivateOtp = $this->ActivateOtpFactory->create();
        $ActivateOtp->setOtpCode($random_number);
        $ActivateOtp->setMobile($mobile);
        $ActivateOtp->save();
    }
}
```

- **How it Works**:
  - Uses cURL to send an HTTP request to the selected SMS vendor's API.
  - Logs the OTP in the database for future validation.

---

#### **3.4 Password Reset: `resetPassword`**

Resets a customer’s password using email or mobile for verification.

```php
public function resetPassword($password, $confirmPass, $identity)
{
    $this->checkPasswordStrength($password);
    if ($password !== $confirmPass) {
        throw new \Magento\Framework\Exception\LocalizedException(__('Password mismatch.'));
    }

    // Fetch customer by email or mobile
    $validator = new \Zend\Validator\EmailAddress();
    if ($validator->isValid($identity)) {
        $customer = $this->_customerRepo->get($identity);
    } else {
        $searchCriteria = $this->_searchBuilder->addFilter('themecafe_mobile', $identity)->create();
        $customer = $this->_customerRepo->getList($searchCriteria)->getItems()[0];
    }

    // Update password
    $customerSecure = $this->customerRegistry->retrieveSecureData($customer->getId());
    $customerSecure->setPasswordHash($this->encryptor->getHash($password, true));
    $this->_customerRepo->save($customer);
}
```

- **How it Works**:
  - Fetches the customer by email or mobile.
  - Validates the password and updates it in the database.

---

### **Step 4: Utility Methods**

The class includes helper methods for common tasks:

- **`ValidateMobileNumber`**: Ensures the mobile number is valid.
- **`generatedRandomNumbers`**: Generates OTP codes.
- **`checkPasswordStrength`**: Validates password complexity.
- **`createEmptyCartQuoteForCustomer`**: Creates an empty cart for the customer.
- **`destroyCustomerSessions`**: Logs the customer out from all active sessions.

---

### **Step 5: Guest Handling**

The class also manages guest customer flows, such as:

- **`Guestregister`**: Registers a guest account and sends verification.
- **`sendverficationviamobilegist`**: Handles OTP for guest users.

---

This detailed breakdown covers the essential parts of the class. Let me know if you'd like further details on specific methods! 😊
