# Magento Component Isolation

Magento Commerce was designed as a monolithic modular application: all codebase is split to functional modules but is deployed together. This has following implications:

* Application has to be deployed as a whole. No ability to deploy updated versions of separate components independently
* Application has to be scaled as a whole. No ability to scale separate components independently
* Even though modular code structure groups related behavior, it is easy to introduce an undesired dependency between application components as components are not deployed and tested independently. Only static modularity analysis is performed.

## Current state
Introduction of service contracts in Magento 2.0 was the first step towards component isolation:

* Every module defines its API (service contracts) – PHP interfaces that can be called either from within PHP process or remotely through REST, SOAP or AMQP APIs
* Every module is allowed to call other modules only through their service contracts
* Implementation of any service contract can be replaced with a proxy that does a call to remote module through REST or SOAP API

In consequent releases more service contracts were introduced, and more modules were switched to service contract communication, but legacy undesired dependencies (direct inter-module model-to-model and presentation-to-model dependencies) still exist. This fact does not allow true module separation. Most of undesired inter-module dependencies reside in UI (presentation-to-model).

On data level, ability to split checkout and order management databases was introduced. This improved scalability of Magento instances.

![Current State](component-isolation/current-state.png)

### PWA & Component isolation
The decision to move all UI to browser as a part of PWA effort significantly reduces number of undesired dependencies in codebase: most dependencies reside in UI, and will not be present in PWA implementation.

## Desired state
Magento _components_ (Catalog, Checkout, Order Management, ...) are isolated and only communicate through service contracts.

A _component_ is a part of Magento application that consists of one or more modules and is responsible for an distinct business behavior.

Examples of components:
   
- Catalog
    - Magento/Catalog
    - Magento/CatalogGraphql
    - Magento/ConfigurableProduct
    - Magento/Downloadable
    - Magento/BundleProduct
    - ...
- Checkout
    - Magento/Checkout
    - Magento/Quote
    - Magento/Multishipping
    - Magento/CheckoutAgreements
    - ...
    
Unlike in the initial plan, the requirement to communicate through service contracts only applies to components, not modules. This significantly reduces the scope of required modifications and makes implementations and customizations easier. 

Upsides:

* scalability – components can be scaled independently on Magento instances
* deployment – components can be deployed independently
* replaceability – it is easier for System Integrators to replace Magento built-in components with third-party systems
* teams have clear ownership boundaries
* easier to comprehend components
* clear contracts between components
* granular releases - components can be released independently
* in isolated components it is easier to use data storage with more appropriate storage models.
* easier to experiment and replace components

Downsides:

* Performance hit of inter-component communication
* Instance maintenance – it is harder to manage distributed Magento instance. This needs to be compensated with improved logging, tracking, debugging, and deployment capabilities.
* Any component can be replaced with a remote implementation

![Desired State](component-isolation/desired-state.png)

Current deployment model (single app) must be supported for clients that don't require advanced scalability.

Following application components are identified to be isolated as of now (the list is not final):

![Isolatd magento components](component-isolation/magento-components.png) 

## Implementation
To achieve the desired state two sets of changes are required: platform modifications and component isolation.

### Platform modifications

To support current ecosystem the platform and main APIs will have to stay the same (PHP, Magento Framework).

Part of platform work is required to allow independent component deployment (module separation, application framework, configurable service invokers, backends for frontends, support for split extensions in marketplace), other part is good to have to make independently deployed components manageable (cloud native, development environments).

### Application framework

Following library components are currently implemented as Magento modules. Many modules depend on these library components. It clutters module dependency graph. Effectively they will be required for most application components. So these library components (or application-agnostic parts of them) should be moved to `Magento\Framework` to reduce clutter in module dependencies:

* Amqp
* Config
* Cron
* Deploy
* EAV
* GraphQL
* Indexer
* MessageQueue
* MysqlMq
* WebAPI

### Configurable service contract invocations

As of 2.3-develop, Magento service contracts can be called in process, or through REST, SOAP, and AMQP APIs by remote clients. But remote calls require manual implementation of client. Magento should be able to invoke service contracts either locally or remotely (if a service whose contract is called is deployed remotely). Two options to implement this are discussed:

* Introduce service contract invoker that should be used by every client to invoke service contract. For local calls such invoker would just call a service contract, for remote components it would do a remote call. This option gives more control over service contract invocation, but requires modification of all current service contract clients

* Auto-generated remote call proxies for service contracts that represent remote components. This option does not require code modifications but will produce harder to debug code

Remote invoker must contain improved logging, timeouts, retry logic, and circuit breakers to avoid failure state propagation between components.

First iteration of remote implementation should be based on existing synchronous APIs.

Long-term goal: To avoid the leaky abstraction of remote invocations, new generation of service contracts has to expose asynchronous nature in APIs to avoid runtime coupling between components.

### Backends for frontends

Following entry point components need to be created for two main application clients: storefront and admin apps (admin, integrations). These endpoint components (backends for frontends or BFFs) should act as façades to corresponding functionality:

* Storefront PWA BFF
  * launch PWA Application shell with page data (product information, category information)
  * routing to storefront scenarios only
* Storefront GraphQL endpoint
  * Serve GraphQL requests of PWA application
  * Potentially merged with PWA BFF
* Admin backend
  * used by admin app and integrations
  * mostly CRUD operations for entities
  * token based authentication
  * ACL-based authorization.
  * REST
  
### Split modules

The described separation of UI from APIs, and admin from storefront requires extracting all UI and Admin application logic to separate modules. For storefront application UI code will be moved to separate modules in PWA.

To support real and proxied implementations of modules, the APIs should be extracted to separate modules. Webapi should be extracted to separate modules to glue API, implementation, and Webapi declarations together.

Following diagram demonstrates the split:

![Split module](component-isolation/split-modules.png)

* *CatalogPWA* - js module deployed on PWA host
* *CatalogStorefront* - deployed on GraphQL endpoint
* *CatalogGraphQL* - exposes graphql API, deployed on Storefront GraphQL endpoint
* *CatalogAdmin* - deployed on Admin BFF
* *CatalogWebapi* - deployed on Catalog service instance and depeonds on CatalogApi and Catalog modules
* *CatalogAPI* - contains module service contracts
* *Catalog* - contains implementation of Catalog; deployed on Catalog instance
* *CatalogProxy* - contains proxy implementations of Catalog services. Deployed on client services (Checkout, OMS)

### Support for split extensions on Marketplace

Current data model of Marketplace does not support multiple packages per extension. Since an extension can customize multiple components, it should be possible to create an extension that consists of multiple composer packages. Such extensions must be supported by Magento Marketplace.

Example: MyVendorShippingMethod extension modifies shipment and checkout components. MyVendorShippingMethod extension consists of 2 composer packages in Marketplace: MyVendorShipment and MyVendorCheckout.

### Cloud-native

The distributed deployment approach will require improved tooling to make it easy to deploy, monitor and troubleshoot magento application. In addition to that, Magento application will have to better follow the principles of [12-factor application](https://12factor.net/). 
Following changes are proposed:

#### Application configuration

Magento components must support centralized application configuration management.

>NOTE: supported since 2.2 with configuration stored in env.php.

#### Tracing
Correlation IDs must be added to internal calls to make tracing easier.

#### Tooling

##### Magento CLI
Current magento tool depends on application codebase being present on same machine. This makes it hard to manage a distributed instance of Magento.

New endpoint must be introduced that can be exposed on an secure network adapter and that would listen to commands from remote magento tool and execute them.

New standalone magento tool must be created. The tool should have following commands:
* ```magento instance:add [name] [url]``` - register a new managed remote instance
* ```magento instance:remove [name]``` - unregister a managed instance from the tool instance list
* ```magento instance:list``` - list registered remote instances
* ```magento instance:update``` - load list of commands supported by the instance
* ```magento context:set [name]``` - select the default instance to be used in commands

##### Instance builder

With distributed deployment and split modules, composer.json management for different services will be complicated. A tool to build instance `composer.json` files for specific services based on enabled features or installed extensions should be built.

Input: 

Output: 
* List of composer.json files that can be deployed on different instances

The tool should be able to:
* generate list of composer.json based on following input:
    * list of extensions that need to be installed
    * list of components deployed separately (`composer.json` files to generate)
* check compatibility of API versions between modules of different components
* analyse the distributed instance of magento:
    * check api version compatibility
    * display the list of distributed extensions
     
#### Schema migrations

Expansion-cleanup stages should be introduced into schema deployment tool to reduce downtime during service deployment

NOTE: declarative schema in 2.3 makes automated distinction possible

#### Development environment

To make development of distributed instances easier, new developer environment must be created.

A prototype that uses Minikube VM with Kubernetes cluster is prepared. 

>NOTE: Distributed component deployment will make current Commerce/B2B linking approach impossible, so new approach of internal development environment installation must be used: installation of Magento modules from “path” type composer repositories (prototype working with Minikube is available).

## Design principles

General principles to follow for component isolation:

* Current service contracts SHOULD BE preserved for backward compatibility
* All new service contracts MUST follow design principles described in technical guidelines + following principles:
* All operations MUST BE idempotent Network communication is unreliable. Retries will be required, and some messages will be delivered more than once.
* Sagas SHOULD BE used for consistency of distributed operations
* All new service contracts SHOULD expose asynchronous APIs
* All new state modifying operations SHOULD expose bulk APIs
* All service operations MUST BE stateless
* There MUST BE NO data dependencies between components.
* Command & query responsibility segregation – storefront APIs for data immutable in storefront (catalog) should be optimized for data retrieval

Detailed design must be prepared for every component.

## Implementation approach

Iterative approach must be used for component isolation: one component at a time.

Based on component sizes and dependencies and priorities, following isolation sequence is proposed:

1. Platform changes
1. Checkout
1. Pricing
1. Inventory
1. Tax
1. Order Management
1. Customer
1. CMS
1. Reports
1. Payments
1. Shipping
1. Promotions
1. Wishlist
1. Rewards
1. Company
1. Marketing
1. Risk