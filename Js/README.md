# 🧠 What is RequireJS?

**RequireJS** is a **JavaScript file and module loader**. It allows you to define and load JavaScript modules asynchronously, which improves performance and modularity.

Magento 2 uses RequireJS to manage JavaScript dependencies in a clean, maintainable way, without blocking the page load.

---

## 📦 Why Magento 2 Uses RequireJS

1. **Asynchronous Loading** – Faster page loads by loading scripts only when needed.
2. **Dependency Management** – Clear definition of which script depends on what.
3. **Modularity** – Keeps JS code organized in modules.
4. **Customization** – Let’s override or extend JS behavior without modifying core files.

---

## 🗂️ The `requirejs-config.js` File

In Magento 2, `requirejs-config.js` is the file where you declare your custom RequireJS configurations, such as:

- Adding custom JS modules
- Setting paths/aliases
- Creating shims for non-AMD modules
- Mapping/overriding existing modules

It’s commonly placed in:

```
app/code/Vendor/Module/view/frontend/requirejs-config.js
```

You can also define it in the `view/adminhtml` folder if your module targets the admin panel.

---

## 🏗️ Basic Structure of `requirejs-config.js`

Here’s a skeleton with all the main parts:

```js
var config = {
    paths: {
        'aliasName': 'Vendor_Module/js/filename'
    },
    shim: {
        'nonAmdScript': {
            deps: ['jquery'],
            exports: 'nonAmdScript'
        }
    },
    map: {
        '*': {
            'originalModule': 'Vendor_Module/js/custom-module'
        }
    },
    config: {
        mixins: {
            'Magento_Ui/js/view/messages': {
                'Vendor_Module/js/messages-mixin': true
            }
        }
    }
};
```

Let’s break it all down 👇

---

## 🧭 `paths`

The `paths` section creates **shortcuts (aliases)** for module paths. Useful for organizing your code or referencing JS files without long paths.

```js
paths: {
    'customScript': 'Vendor_Module/js/custom-script'
}
```

Usage:

```js
define(['customScript'], function (customScript) {
    // your logic here
});
```

---

## 🧰 `shim`

This is used for **non-AMD** scripts (scripts that don’t define themselves using `define()`).

```js
shim: {
    'legacyScript': {
        deps: ['jquery'],
        exports: 'Legacy'
    }
}
```

This tells RequireJS:
- “Before loading `legacyScript`, load `jquery`.”
- “This script exposes a global object called `Legacy`.”

---

## 🔄 `map`

Allows **remapping module paths** globally or for specific modules.

```js
map: {
    '*': {
        'Magento_Checkout/js/view/shipping': 'Vendor_Module/js/view/shipping-override'
    }
}
```

This will **override** the default shipping JS component across the site.

---

## 🧬 `config > mixins`

Use this for **mixins**, which let you extend core JS components **without rewriting** the whole thing.

```js
config: {
    mixins: {
        'Magento_Ui/js/form/form': {
            'Vendor_Module/js/form-mixin': true
        }
    }
}
```

Your `form-mixin.js` file should look like:

```js
define([], function () {
    'use strict';
    return function (target) {
        return target.extend({
            // Override or add methods
            initialize: function () {
                this._super();
                console.log('Form mixin initialized!');
                return this;
            }
        });
    };
});
```

---

## ✅ Step-by-Step Example

Let’s say you want to add a new JS file called `custom-alert.js`.

### 1. Create the file:

```
app/code/Vendor/Module/view/frontend/web/js/custom-alert.js
```

```js
define(['jquery'], function ($) {
    'use strict';
    return function () {
        alert('Hello from custom script!');
    };
});
```

### 2. Register it in `requirejs-config.js`:

```js
var config = {
    paths: {
        'customAlert': 'Vendor_Module/js/custom-alert'
    }
};
```

### 3. Use it in a `.phtml` file:

```php
<script type="text/javascript">
    require(['customAlert'], function (customAlert) {
        customAlert();
    });
</script>
```

---

## 🔁 Deploying Changes

Don’t forget to deploy static files and clear cache:

```bash
php bin/magento setup:upgrade
php bin/magento setup:static-content:deploy -f
php bin/magento cache:clean
```

---

## 🔍 Debugging Tips

- Use browser dev tools' **Network tab** to see if modules are loaded correctly.
- Use **`require(['module'], function () { ... });`** in browser console to test modules.
- Enable `developer` mode for easier debugging:
  ```bash
  php bin/magento deploy:mode:set developer
  ```

