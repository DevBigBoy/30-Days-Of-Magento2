# 30 Days Of Magento 2

A comprehensive guide to learning Magento 2 development concepts from beginner to advanced level.

## Table of Contents

| Day | Topics | Description |
| --- | ------ | ----------- |
| **Week 1: Fundamentals** | | |
| 01 | [Introduction](./Day-01-Introduction/README.md) | What is Magento 2, Architecture Overview, MVC Pattern |
| 02 | [Installation & Setup](./Day-02-Installation/README.md) | Installation methods, System requirements, CLI commands |
| 03 | [Module Creation](./Day-03-Module-Creation/README.md) | Module structure, registration.php, module.xml, Enabling/Disabling modules |
| 04 | [Dependency Injection](./README.md) | DI concepts, di.xml, Object Manager, Preferences, Virtual Types |
| 05 | [Routing](./Routing/README.md) | Frontend/Admin routes, routes.xml, Controllers, Action classes |
| 06 | [Database Schema](./Database/README.md) | db_schema.xml, Declarative Schema, Patches, Data/Schema patches |
| 07 | [Models & Collections](./Model/README.md) | Models, Resource Models, Collections, CRUD operations |
| **Week 2: Core Concepts** | | |
| 08 | [Service Contracts](./Service%20Layer/README.md) | Repository Pattern, API interfaces, Data interfaces, SearchCriteria |
| 09 | [Data Objects](./Data-Object/README.md) | Data Transfer Objects, ExtensibleDataInterface, Custom attributes |
| 10 | [Plugins (Interceptors)](./Plugin/README.md) | Before, After, Around plugins, Plugin priority, Plugin sorting |
| 11 | [Events & Observers](./Observer-Evenet/README.md) | Event dispatch, Observer pattern, events.xml, Custom events |
| 12 | [Layouts & Blocks](./README.md) | Layout XML, Blocks, Containers, Layout handles, Layout updates |
| 13 | [Templates & UI Components](./view/README.md) | PHTML templates, Template directives, UI Components, Form/Listing components |
| 14 | [View Models](./ViewModel/README.md) | ViewModel pattern, Separating business logic from templates |
| **Week 3: Advanced Features** | | |
| 15 | [EAV Model](./README.md) | Entity-Attribute-Value pattern, Product attributes, Custom EAV entities |
| 16 | [System Configuration](./Configuration-Settings/README.md) | system.xml, config.xml, ACL for config, Store/Website/Default scope |
| 17 | [Admin Grids & Forms](./README.md) | UI Components for admin, Grid columns, Filters, Mass actions |
| 18 | [Cron Jobs](./Cron/README.md) | crontab.xml, Cron groups, Scheduling tasks, Custom cron jobs |
| 19 | [Logging & Debugging](./Logs/README.md) | Logger interface, Custom log files, Debug mode, Profiling |
| 20 | [Exception Handling](./Exception/README.md) | Exception types, Custom exceptions, Error handling best practices |
| 21 | [ACL & Permissions](./README.md) | Access Control Lists, acl.xml, Resource-based permissions, Roles |
| **Week 4: Integration & Advanced Topics** | | |
| 22 | [GraphQL](./GraphQl/README.md) | GraphQL schema, Resolvers, Mutations, Queries, Custom GraphQL endpoints |
| 23 | [REST & SOAP APIs](./README.md) | Web API configuration, webapi.xml, Authentication, Token-based auth |
| 24 | [JavaScript & RequireJS](./Js/README.md) | RequireJS modules, Mixins, jQuery widgets, Knockout JS |
| 25 | [Customer Data (Sections)](./CustomerData/README.md) | Private content, Customer sections, Cache invalidation |
| 26 | [Payment Integration](./Payments/README.md) | Payment gateway integration, Payment methods, Transaction handling |
| 27 | [Message Queue (RabbitMQ)](./RabbitMq/README.md) | Queue configuration, Publishers, Consumers, Async operations |
| 28 | [Indexers](./README.md) | Indexing process, Custom indexers, Mview, Reindex modes |
| 29 | [Cache Management](./README.md) | Cache types, Cache tags, Full Page Cache, Varnish, Redis |
| 30 | [Testing & Deployment](./README.md) | Unit tests, Integration tests, Static tests, Deployment strategies |

## Additional Important Topics

### Design Patterns
- [Design Patterns in Magento 2](./Design%20Patterns/README.md)
  - Factory Pattern
  - Repository Pattern
  - Proxy Pattern
  - Strategy Pattern
  - Singleton Pattern
  - Object Pool Pattern

### Product Management
- [Product Types](./Products/README.md)
  - Simple, Configurable, Bundle, Grouped, Virtual, Downloadable
  - Custom product types
  - Product attributes and attribute sets

### Performance & Optimization
- Code compilation and generation
- Static content deployment
- Database optimization
- Query optimization
- Production mode vs Developer mode

### Best Practices
- Coding standards (PHPCS, PHPMD)
- Security best practices
- Upgrade compatibility
- Backward compatibility
- Extension attributes vs Custom attributes

### DevOps & Tools
- Composer dependency management
- Git workflow
- CI/CD pipelines
- Docker for Magento 2
- Cloud deployments (Adobe Commerce Cloud)

### Customization Techniques
- Theme creation and customization
- Custom modules vs Core modifications
- Upgrade-safe customization
- Multi-store/Multi-website setup

## Quick Reference

### Essential CLI Commands
```bash
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento setup:static-content:deploy
bin/magento cache:clean
bin/magento cache:flush
bin/magento indexer:reindex
bin/magento module:enable ModuleName
bin/magento module:disable ModuleName
```

### Useful Resources
- [Official Magento DevDocs](https://devdocs.magento.com/)
- [Adobe Commerce Developer Guide](https://experienceleague.adobe.com/docs/commerce.html)
- [Magento Stack Exchange](https://magento.stackexchange.com/)
- [Magento GitHub Repository](https://github.com/magento/magento2)

## Contributing
Feel free to contribute to this learning repository by creating pull requests with additional examples, corrections, or improvements.

## License
This project is open source and available for educational purposes.