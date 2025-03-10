# Understanding the Platform Architecture

- Magento is a powerful, highly scalable, and highly customizable e-commerce platform that can be used to build web shops and, if needed, some non-e-commerce sites. It provides a large number of e-commerce features out of the box.

- Features such as product inventory, shopping cart, support for numerous payment and shipment methods, promotion rules, content management, multiple currencies, multiple languages, multiple websites, and so on make it a great choice for merchants. On the other hand, developers enjoy the full set of merchant-relevant features plus all the things related to actual development. This chapter will touch upon the topic of robust Web API support, extensible administration interface, modules, theming, embedded testing frameworks, and much more.

Depending upon your role and purpose for learning more about Magento 2, there are several different ways to view Magento 2 architecture.

For example, a developer who wants to create new modules or perhaps customize an existing module needs to understand the architecture of a module itself.

It also needs to understand how it fits into the larger view, with the Magento 2 framework and other components.

However, a merchant who wants to quickly build an online storefront wants to view the collection of components from a higher level.

It also needs to understand the components that impact the look, feel, and user interaction components.

## Architectural layers overview

At its highest level, the Adobe Commerce and Magento 2 Open Source framework (Commerce framework) architecture consists of the core product code plus optional modules.

These optional modules enhance or replace the basic product code.

If you are substantially customizing the basic Adobe Commerce or Magento 2 Open Source product, module development will be your central focus.

Modules organize code that supports a particular task or feature.

A module can include code to change the look and feel of your storefront as well as its fundamental behavior.

Your modules function with the core product code, which is organized into layers.

Understanding layered software patterns is essential for understanding basic Adobe Commerce and Magento 2 Open Source product organization.

## Advantages of layered application design

- Separation of business logic from presentation logic simplifies the customization process which helps you to alter your storefront appearance without affecting any of the backend business logic.
- Clear organization of code predictably points extension developers to code location.

## The architecture of Magento 2
Magento 2 has a Model View ViewModel (MVVM) architecture.

This MVVM architecture provides a much more robust separation between the Model and View layer, as it is closely related to the Model View Controller (MVC).

A brief description of MVVC is given below:

## Model

## View

## ViewModel


## Layers in Magento 2

## Presentation Layer

### Conclusion

Understanding the architecture of Magento 2 is crucial for anyone looking to effectively utilize the platform.

Whether you are a developer aiming to create or customize modules, or a merchant focused on building a visually appealing online storefront.

The layered architecture of Magento 2 comprising the Presentation, Service, Domain, and Persistence layers facilitates a clear separation of concerns.

It allowing for easier customization and enhanced maintainability.

The MVVM pattern further strengthens this architecture by delineating the roles of the Model, View, and ViewModel, promoting a more organized approach to code development.

By grasping these architectural concepts, users can better navigate the complexities of Magento 2, ensuring a more efficient and effective implementation of their e-commerce solutions.

Ultimately, this foundational knowledge empowers users to leverage Magento 2’s capabilities to meet their specific needs, driving successful online commerce.