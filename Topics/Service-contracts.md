# Service contracts

- A service contract is a set of PHP interfaces that is defined by a module.
This contract comprises data interfaces and service interfaces.
The role of the data interface is to preserve data integrity, while the role of
the service interface is to hide the business logic details from service
consumers.
Data interfaces define various functions, such as validation, entity
information, search related functions, and so on. They are defined within
the Api/Data directory of an individual module. To better understand the
actual meaning of it, let's take a look at the data interfaces for the
Magento_Cms module. In the vendor/magento/module-cms/Api/Data/
directory, there are four interfaces defined, as follows:
BlockInterface.php
BlockSearchResultsInterface.php
PageInterface.php
PageSearchResultsInterface.php
The CMS module actually deals with two entities, one being Block and the
other one being Page. Looking at the interfaces defined in the preceding
code, we can see that we have separate data interface for the entity itself
and separate data interface for search results.
