# Configuration and customization of System.xml

## 🧠 What is a `system.xml` File?

The `system.xml` file:

- Defines **admin configuration settings** visible under **Stores > Configuration**.
- It creates sections, groups, and fields in the Magento **backend system config UI**.
- Links these settings to configuration values (e.g., `core_config_data` DB table).

---

## 🔧 Why It’s Important

| Purpose                | Description                                                                                                |
| ---------------------- | ---------------------------------------------------------------------------------------------------------- |
| 📦 Module Configuration | Allows merchants/admins to configure features like SMS gateway, OTP behavior, etc.                         |
| 🔐 Scope-Aware          | Supports global, website, or store view levels.                                                            |
| 🔄 Dynamic Logic        | You can use `depends`, `source_model`, `frontend_model`, `backend_model`, etc., to create smart behaviors. |

---

## 🧱 Structure of `system.xml`

Here’s the **core hierarchy** of this file:

```xml
<config>
  <system>
    <section>         ← A main menu item
      <group>         ← A fieldset or category inside section
        <field>       ← An individual setting
        ...
      </group>
    </section>
  </system>
</config>
```

Now let’s walk through **your file step-by-step** 🔍

---

## 🏷️ `<section id="croco">`

- This is the **main config section** (top-level menu item in admin).
- Appears in **Stores > Configuration** under the `CrocoIT` tab.
- Has the label "SMS Notification".

```xml
<section id="croco" translate="label" sortOrder="100" showInDefault="1" showInWebsite="1" showInStore="1">
```

---

### 🔹 `tab`: Specifies the top-level tab to group this section under

```xml
<tab>crocoit</tab>
```

To show under your custom tab (you'd need a `crocoit.xml` in `system.xml` format too, or Magento will use default).

---

### 🔐 `resource`: ACL path for permissions

```xml
<resource>CrocoIT_Sms::configuration</resource>
```

Magento checks if the admin user has permission to access this section.

---

## 🧩 Group: `settings`

Groups contain actual settings. You define multiple `group` blocks under a section.

```xml
<group id="settings" label="Settings" ...>
```

Inside here, you've got:

- SMS gateway options
- Admin phone config
- Duplicate number handling
- Mobile login toggle
- Verification rules

### 📥 Example Field: SMS Gateway Selector

```xml
<field id="gateway" type="select" ...>
    <label>SMS Gateway</label>
    <source_model>crocoit\Sms\Model\Config\Source\Gateways</source_model>
    <config_path>croco/settings/gateway</config_path>
</field>
```

- **type="select"**: dropdown in admin
- **source_model**: Returns an array of options
- **config_path**: Where this value is saved (in `core_config_data`)

---

## 🧾 Group: `customer_verify`

For verifying mobile numbers:

- Toggle verification on/off
- Require it at registration
- Use OTP at login or checkout

Fields here include:

```xml
<field id="verify_customer_mobile" type="select" ...>
```

This one drives conditional fields via `<depends>`.

---

## 🔐 OTP Configuration Group

This whole section lets admin configure:

- Format and length
- Expiration time
- Resend block rules

Example:

```xml
<field id="otp_length" type="text" ...>
    <frontend_class>required-entry validate-number</frontend_class>
</field>
```

✅ Adds HTML validation to force valid numbers.

---

## 📱 Input Settings Group

Controls how the mobile number input behaves in frontend:

- Country dropdown toggle
- Default country
- GeoIP DB path
- Allowed countries
- Preferred countries

This config maps into your frontend JS (`intlTelInput`) behavior.

---

## 📢 Admin Notification Group

Let admin configure SMS when:

- Customer registers
- Order is placed
- Contact form is submitted
- Review is written

Each fieldset:

```xml
<group id="customer_register" ...>
```

- Has an `enabled` toggle
- A customizable message
- Can add `depends` to hide message unless enabled

---

## 👥 Customer Notification Group

Let merchants control messages for customers:

- On registration
- Order confirmation
- Status updates
- Invoices, shipments, refunds (credit memos)

Also lets admin define **priority of mobile sources** (shipping, billing, registration).

---

## 🧪 Config Paths & Storage

Each `<field>` with a `config_path` gets saved to the **`core_config_data`** table:

```bash
SELECT * FROM core_config_data WHERE path = 'croco/settings/gateway';
```

You can access them in code like this:

```php
$this->scopeConfig->getValue('croco/settings/gateway', \Magento\Store\Model\ScopeInterface::SCOPE_STORE);
```

---

## 🧠 Summary of Key Features

| Element          | Purpose                                            |
| ---------------- | -------------------------------------------------- |
| `<section>`      | Top-level menu                                     |
| `<group>`        | Category of settings                               |
| `<field>`        | Actual setting (select, input, textarea)           |
| `source_model`   | Supplies dropdown values                           |
| `frontend_model` | Custom rendering of field                          |
| `backend_model`  | Validates/saves custom logic                       |
| `depends`        | Makes a field conditional on another field’s value |
| `config_path`    | Saves data to database                             |
| `resource`       | ACL permission control                             |

---

## 🔍 What `config_path` Does

`<config_path>` defines the **location** where the field’s value will be **stored in the `core_config_data` table**.

This means when an admin saves a value in the Magento backend (like enabling OTP login), Magento knows:
> “Hey, I should store this value under the path `croco/settings/enable_otp_login` in the config table.”

---

### 🗄️ What It Looks Like in the DB

Magento stores system config values in this table:

```bash
core_config_data
```

A row created from this field would look like:

| config_id | scope   | scope_id | path                            | value |
| --------- | ------- | -------- | ------------------------------- | ----- |
| 123       | default | 0        | croco/settings/enable_otp_login | 1     |

- `path`: Comes from `<config_path>`
- `value`: Comes from what the admin selected (`1` for Yes, `0` for No)
- `scope`: Tells Magento if it's global (`default`), per website, or per store view

---

## 🧠 Why It’s Important

| ✅ Role          | 📄 Description                                                                  |
| --------------- | ------------------------------------------------------------------------------ |
| 🧩 Save location | Tells Magento where to persist the setting in DB                               |
| 🔄 Retrievable   | You can fetch this value using Magento’s `ScopeConfigInterface`                |
| 🗺️ Scoping       | Respects global/website/store-level config                                     |
| 🛠️ Overrides     | Can be overridden per scope (if allowed via XML `showInWebsite`/`showInStore`) |

---

## 🧰 How to Read This Value in Code

In your block, helper, model, etc., you’d use this to fetch the value:

```php
use Magento\Framework\App\Config\ScopeConfigInterface;

class YourClass
{
    protected $scopeConfig;

    public function __construct(ScopeConfigInterface $scopeConfig)
    {
        $this->scopeConfig = $scopeConfig;
    }

    public function isOtpLoginEnabled(): bool
    {
        return $this->scopeConfig->isSetFlag(
            'croco/settings/enable_otp_login',
            \Magento\Store\Model\ScopeInterface::SCOPE_STORE
        );
    }
}
```

---

## 🏗️ Real-Life Scenario

Let’s say you're building a **custom login module with OTP**, and you want to **enable/disable the feature** based on admin setting.

In your controller or frontend block, you'd check the value of:

```php
croco/settings/enable_otp_login
```

If enabled (`1`), you'd:

- Display the OTP field
- Trigger the SMS logic
- Validate OTP before login

If disabled (`0`), you'd skip all of that.

---

### 🔍 What Happens When a `<field>` **Does NOT** Have a `<config_path>`?

```xml
<field id="validator" ...>
    <label>Mobile Validator</label>
    <source_model>CrocoIT\Sms\Model\Config\Source\MobileValidator</source_model>
</field>
```

### ✅ In This Case

Magento will **automatically store the value** using a **default path pattern** derived from:

```php
<section>/<group>/<field>
```

So here, assuming it's inside:

```xml
<section id="croco">
  <group id="settings">
    <field id="validator" ...>
```

The **config path** becomes:

```php
croco/settings/validator
```

Even though it’s not explicitly written in the XML, Magento **infers it**.

---

## 🧪 How Magento Figures It Out

Magento uses the logic in `Magento\Config\Model\Config\Structure` to build the final config paths at runtime.

When you save this field, Magento maps it like:

```php
$fieldPath = $sectionId . '/' . $groupId . '/' . $fieldId;
```

So you can **still retrieve it in PHP** with:

```php
$this->scopeConfig->getValue('croco/settings/validator');
```

Or using `isSetFlag()` if it's boolean-like.

---

## 🧠 Why Use `<config_path>` At All?

Explicitly defining `<config_path>` lets you:

- 🔄 **Reuse** a config field in multiple groups
- 🔗 **Map multiple fields** to the same config value
- 🛠️ Store data under a **custom key**, not just the default `section/group/field` path

---

## ✅ Summary

| Scenario | Where value is stored |
|----------|-----------------------|
| `<config_path>` is set | At the custom path you define |
| `<config_path>` is **not** set | At `section/group/field` path, automatically |

---
