# The configuration array used with Magento 2’s `EavSetup::addAttribute()`

---

## 🧾 Comprehensive List of Attribute Properties

Here's a detailed explanation of each property in your array, followed by some additional ones you can use.

---

### 🔹 `type`

- **Description**: The backend storage data type in the database.
- **Accepted Values**: `'int'`, `'varchar'`, `'text'`, `'decimal'`, `'datetime'`
- **Example**: `'varchar'` stores short strings (e.g., names, labels).
- **Mapped Table**: For products → `catalog_product_entity_varchar`

---

### 🔹 `label`

- **Description**: The human-readable label shown in the admin panel.
- **Example**: `'My Custom Attribute'`
- **Note**: This will appear as the field’s name in the product edit form.

---

### 🔹 `input`

- **Description**: Specifies the frontend input type (i.e., form element).
- **Common Values**:
  - `'text'` – Input field
  - `'textarea'` – Multiline text box
  - `'select'` – Dropdown
  - `'multiselect'` – Multi-select list
  - `'boolean'` – Yes/No toggle
  - `'date'` – Date picker
  - `'price'` – Price input
  - `'media_image'` – Image uploader
- **Example**: `'text'`

---

### 🔹 `required`

- **Description**: Whether the field is required in the admin form.
- **Type**: Boolean (`true` or `false`)
- **Example**: `false`

---

### 🔹 `sort_order`

- **Description**: Determines the position of the attribute in the form (within its group).
- **Type**: Integer
- **Example**: `100` → Lower numbers appear higher.

---

### 🔹 `global`

- **Description**: Scope of the attribute value.
- **Possible Constants**:
  - `SCOPE_STORE` – Store View
  - `SCOPE_WEBSITE` – Website
  - `SCOPE_GLOBAL` – Global (default across all stores)
- **Namespace**: `\Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface`
- **Example**: `SCOPE_STORE`

---

### 🔹 `visible`

- **Description**: Whether this attribute is visible in the admin product edit form.
- **Type**: Boolean
- **Example**: `true`

---

### 🔹 `user_defined`

- **Description**: Marks the attribute as user-defined, as opposed to system-defined.
- **Type**: Boolean
- **Example**: `true`

---

### 🔹 `group`

- **Description**: The attribute group under which this field appears in the admin form.
- **Type**: String (must match an existing attribute group or it will create one)
- **Example**: `'General'`

---

## ✅ Additional Commonly Used Properties

Here are more properties you can define to control various behaviors of the attribute:

---

### 🔸 `default`

- **Description**: Default value for the attribute.
- **Example**: `'Default Value'`

---

### 🔸 `visible_on_front`

- **Description**: Show attribute on the frontend (e.g., on product page).
- **Type**: Boolean
- **Example**: `true`

---

### 🔸 `used_in_product_listing`

- **Description**: Whether the attribute should be available in product listings (e.g., API, listing blocks).
- **Type**: Boolean
- **Example**: `true`

---

### 🔸 `searchable`

- **Description**: Whether the attribute should be searchable in site search.
- **Type**: Boolean
- **Example**: `true`

---

### 🔸 `filterable`

- **Description**: Whether the attribute should be filterable in layered navigation.
- **Type**: Integer (0 = No, 1 = Filterable, 2 = Filterable with results)
- **Example**: `1`

---

### 🔸 `comparable`

- **Description**: Whether the attribute can be used in product comparison.
- **Type**: Boolean
- **Example**: `true`

---

### 🔸 `used_for_sort_by`

- **Description**: Allows sorting products in catalog using this attribute.
- **Type**: Boolean
- **Example**: `true`

---

### 🔸 `is_html_allowed_on_front`

- **Description**: Whether HTML tags are allowed in frontend display.
- **Type**: Boolean
- **Example**: `true`

---

### 🔸 `note`

- **Description**: Adds a note under the field in the admin form.
- **Example**: `'Only use lowercase letters.'`

---

## 🧠 Pro Tip

When using `select`, `multiselect`, or `boolean`, you’ll often need to provide **source models** and **option sets**. For example:

```php
'source' => \Magento\Eav\Model\Entity\Attribute\Source\Boolean::class
```

Or define a custom source model for options.

---