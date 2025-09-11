# Magento 2 Architecture Diagrams

## Complete Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        WEB/API REQUEST                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     FRONT CONTROLLER                                │
│                  (Magento\Framework\App)                            │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        ROUTERS                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │ Standard │ │   Admin  │ │   API    │ │   CMS    │              │
│  │  Router  │ │  Router  │ │  Router  │ │  Router  │              │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   PRESENTATION LAYER                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │   Controllers    │  │     Blocks       │  │    Templates     │ │
│  │  (Action Class)  │  │  (Data Prepare)  │  │   (PHTML/HTML)   │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│  ┌──────────────────┐  ┌──────────────────┐                       │
│  │     Layouts      │  │  UI Components   │                       │
│  │      (XML)       │  │   (Complex UI)   │                       │
│  └──────────────────┘  └──────────────────┘                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     SERVICE LAYER                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │               Service Contracts (APIs)                       │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │  │
│  │  │  Repositories   │  │ Data Interfaces │  │  Search API  │ │  │
│  │  └─────────────────┘  └─────────────────┘  └──────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  Web API Layer                               │  │
│  │        REST API    │    SOAP API    │    GraphQL API        │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DOMAIN LAYER                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │     Models       │  │ Resource Models  │  │   Collections    │ │
│  │ (Business Logic) │  │  (DB Abstraction)│  │  (Model Groups)  │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
│  ┌──────────────────┐  ┌──────────────────┐                       │
│  │    Helpers       │  │   Factories      │                       │
│  │  (Utility Code)  │  │ (Object Creation)│                       │
│  └──────────────────┘  └──────────────────┘                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   PERSISTENCE LAYER                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Database (MySQL)                          │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐            │  │
│  │  │ EAV Tables │  │ Flat Tables│  │Index Tables│            │  │
│  │  └────────────┘  └────────────┘  └────────────┘            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Cache Storage                             │  │
│  │        Redis / Memcached / Varnish / File System             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  File System / Media Storage                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## Module Interaction Flow

```
┌──────────────┐
│   Module A   │
└──────┬───────┘
       │
       │ Uses Service Contract
       ▼
┌──────────────────────────┐
│  Product Repository      │ ◄── Service Contract Interface
│  Interface               │
└──────┬───────────────────┘
       │
       │ Implementation
       ▼
┌──────────────────────────┐
│  Product Repository      │
│  (Module B)              │
└──────┬───────────────────┘
       │
       │ Uses Model
       ▼
┌──────────────────────────┐
│  Product Model           │
│  (Module B)              │
└──────┬───────────────────┘
       │
       │ Uses Resource Model
       ▼
┌──────────────────────────┐
│  Product Resource Model  │
│  (Module B)              │
└──────┬───────────────────┘
       │
       │ Database Query
       ▼
┌──────────────────────────┐
│      Database            │
└──────────────────────────┘
```

## Plugin (Interceptor) Flow

```
                    ┌─────────────────┐
                    │  Original Class │
                    │   public method │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌──────────────┐    ┌──────────────┐
│ before Plugin │    │    around    │    │after Plugin  │
│   (Plugin 1)  │    │    Plugin    │    │  (Plugin 1)  │
└───────┬───────┘    │  (Plugin 2)  │    └──────▲───────┘
        │            └──────┬───────┘           │
        │                   │                   │
        │                   ▼                   │
        │            ┌──────────────┐           │
        │            │   proceed()  │           │
        │            │   Original   │           │
        │            │    Method    │───────────┘
        │            └──────────────┘
        │                   │
        └───────────────────┘
```

## Event & Observer Pattern

```
┌─────────────────────────────────────────────┐
│          Controller / Model / Block         │
└──────────────────┬──────────────────────────┘
                   │
                   │ 1. Dispatches Event
                   ▼
┌─────────────────────────────────────────────┐
│          Event Manager                      │
└──────┬──────────┬───────────┬───────────────┘
       │          │           │
       │ 2. Notifies Observers
       │          │           │
       ▼          ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Observer 1│ │Observer 2│ │Observer 3│
│(Module A)│ │(Module B)│ │(Module C)│
└──────────┘ └──────────┘ └──────────┘
       │          │           │
       │ 3. Execute logic
       │          │           │
       ▼          ▼           ▼
  [Action]   [Action]    [Action]
```

## Dependency Injection Container

```
┌──────────────────────────────────────────┐
│        Application Bootstrap             │
└──────────────────┬───────────────────────┘
                   │
                   │ 1. Loads di.xml files
                   ▼
┌──────────────────────────────────────────┐
│      Dependency Injection Container      │
│  ┌────────────────────────────────────┐  │
│  │    Class Preferences               │  │
│  │    Virtual Types                   │  │
│  │    Constructor Arguments           │  │
│  │    Plugins Configuration           │  │
│  └────────────────────────────────────┘  │
└──────────────────┬───────────────────────┘
                   │
                   │ 2. Object creation requested
                   ▼
┌──────────────────────────────────────────┐
│         Object Manager                   │
│  - Creates instances                     │
│  - Injects dependencies                  │
│  - Applies plugins                       │
└──────────────────┬───────────────────────┘
                   │
                   │ 3. Returns instance
                   ▼
┌──────────────────────────────────────────┐
│       Your Class Instance                │
│    (with all dependencies injected)      │
└──────────────────────────────────────────┘
```

## Full Request-Response Cycle

```
1. HTTP Request
   │
   ▼
2. pub/index.php
   │
   ▼
3. Bootstrap Application
   │
   ├─► Load Configuration
   ├─► Initialize DI Container
   ├─► Setup Error Handler
   └─► Create Application Instance
   │
   ▼
4. Front Controller
   │
   ├─► Load Routes
   └─► Match URL to Controller
   │
   ▼
5. Dispatch to Router
   │
   ├─► Standard Router (frontend/customer URLs)
   ├─► Admin Router (admin URLs)
   ├─► API Router (rest/V1/... URLs)
   └─► CMS Router (CMS pages)
   │
   ▼
6. Execute Controller Action
   │
   ├─► Validate Request
   ├─► Check Permissions (ACL)
   ├─► Call Models/Repositories
   ├─► Dispatch Events
   └─► Return Result Object
   │
   ▼
7. Result Object Processing
   │
   ├─► Page Result → Load Layout
   ├─► JSON Result → Return JSON
   ├─► Redirect Result → Send Headers
   └─► Forward Result → Forward to Another Action
   │
   ▼
8. Layout Processing (if Page Result)
   │
   ├─► Load Layout XML files
   ├─► Merge layout updates
   ├─► Create Block instances
   └─► Generate structure
   │
   ▼
9. Block Rendering
   │
   ├─► Call Block methods
   ├─► Load Templates
   ├─► Render child blocks
   └─► Apply plugins
   │
   ▼
10. Template Rendering
   │
   ├─► Execute PHP code
   ├─► Escape output
   └─► Generate HTML
   │
   ▼
11. HTTP Response
   │
   ├─► Set Headers
   ├─► Set Cookies
   ├─► Output Content
   └─► Send to Browser
```

## Cache Flow

```
┌──────────────┐
│   Request    │
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│  Full Page Cache?    │
│     (Varnish)        │
└──────┬───────────────┘
       │
       ├─► HIT → Return cached page
       │
       └─► MISS
           │
           ▼
   ┌───────────────────┐
   │  Block Cache?     │
   └───────┬───────────┘
           │
           ├─► HIT → Use cached block
           │
           └─► MISS → Generate block
                      └─► Cache block with tags
```
