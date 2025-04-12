# comparing Setup/Update Schema vs Declarative Schema in Magento 2
---

## 📄 **Documentation Outline: Setup/Update Schema vs Declarative Schema**

---

#### **1. Introduction**
- Brief explanation of what database schema is in Magento.
- Purpose of the document (comparison, use cases, recommendations).

---

#### **2. Overview of Each Approach**

**A. Setup/Update Schema (InstallSchema, UpgradeSchema)**
- Location in the code (`Setup` directory in module)
- How it works (executed during `bin/magento setup:upgrade`)
- PHP-based approach

**B. Declarative Schema**
- Introduced in Magento 2.3+
- Based on XML files (`db_schema.xml`)
- Magento compares current DB state with schema definition

---

#### **3. Use Cases**

- When to use Setup/Upgrade scripts (e.g. conditional logic, data manipulation)
- When Declarative Schema is better (e.g. schema consistency, clean versioning)
- Hybrid scenarios (mixing declarative with data patches)

---

#### **4. Pros and Cons**

| Feature                      | Setup/Upgrade Schema           | Declarative Schema             |
|-----------------------------|--------------------------------|--------------------------------|
| Complexity                  | Higher (manual control)         | Lower (Magento handles diff)   |
| Versioning                  | Manual                          | Automatic via `schema_whitelist` |
| Rollback Support            | ❌ Not supported                | ✅ Built-in rollback (`rollback` folder) |
| Migration Ease              | Difficult                       | Easier                         |
| Data Patching               | Required                        | Separate concern (`DataPatchInterface`) |
| Error Prone                 | Higher (manual coding)          | Lower (Magento validates XML) |

---

#### **5. Rollback Mechanism**

**Setup Schema:**
- No native rollback
- Must be handled manually

**Declarative Schema:**
- Supports automatic rollback via generated rollback scripts
- How to generate and use `rollback` folder

---

#### **6. Practical Example (Optional but Recommended)**  
- Add a sample use case (e.g. add column `mobile_number` to `customer_entity`)
- Show both implementations side by side:
  - PHP setup script
  - XML declarative schema

---

#### **7. Migration Path**
- How to migrate from Setup Schema to Declarative Schema
- Tools or commands needed (e.g. `bin/magento setup:db-declaration:generate-whitelist`)

---

#### **8. Best Practices**
- Use Declarative Schema for all new development
- Use Data Patches for data changes
- Avoid logic inside schema changes

---

#### **9. Conclusion & Recommendation**
- Summarize when to choose which
- Final advice for your team

---
