# SVED architectural pattern
In an API-Management context the SVED Architectural pattern helps to reliably process incoming requests. SVED stands for 
* **Store** - persist the request for operational process and further usage
* **Validate** - ensure that the request conforms to the API-Spec - provide early feedback to the caller
* **Enhance** - give the request all the additional data which is required for further processing. The caller might not know all internals of your company
* **Distribute** - an incoming call most of the time holds some information which should be provided (distributed) to some downstream systems 

# Challenges:
There are some general challenges the services provider faces, when processing incoming requests
 * Support operational processes by STORE
    * service provider needs a data basis for further processing
    * service provider needs to be able to provide received data in case of errors   
 * Force high data quality by VALIDATION
    * a service provider wants only to accept high quality data
    * a service consumer wants to have early feedback - and thus a chance to early correct potentially invalid data
 * Accept different bounded contexts through ENHANCEMENT:
   * a service provider receives a request with data from the context of the consumer. 
   * a consumer only can provide data which is available in the context of the consumer.
   the service provider often needs more data than only the provided one. So the request needs to be enhanced in a structured way for further processing
 * Reliably deliver all downstream systems with expected data through DISTRIBUTION:
   after validation and a potential enhancement, what usually happens with an incoming request? It has do be provided to one or more downstream systems, where the real business processing is being done
   a unique internal identification might be created, a CRM system delivered with the new partner, a price calculated and so on. But how to we ensure, that the distribution happened relialby, which means that all required downstream systems successfully processed the call?    
             
# Context:
We discovered this pattern in a context, where Baloise Insurance worked in cooperation with a SaaS-Provider. The SaaS-Provider hosts software for Baloise Insurance which allows a customer to buy an insurance contract. 
The data of the insurance contract is out in the wild on some arbitrary cloud provider which might change tomorrow.
This represents an obligation of Baloise Insurance to cover risk for a customer. Therefore Baloise Insurance is required by legal restrictions to have the contract data our own systems.
While this pattern is applicable for any request which provides data to downstream systems for processing we believe, that it should be assessed if this pattern might help.

The following questions can help
## is the processing and/or the response of high relevance? 
Obviously if you receive data from external partners, which you have to process in a reliable manner this pattern applies very well. But how about GET requests - where the user expects a simple response?
Again the central question to ask is the relevance. If we think about a weather API that provides private persons with a information about the current situation and forecast. Should we store the request, validate it afterwards, provide early feedback
about correctness of the request, enhance it and the distribute? This sounds like an overengineered solution as in a private context usually its of not high relevance, if the weather information can be provided now.
But what if we think about weather forecasts in professional situations like e.g. a flight from Zurich to New York. 
What if the weather forecast fails or even tells wrong local information? This might lead to serious situations during the flight which could endanger many people. 
In this case the whole processing of the request and creation of the response is of high relevance as it might be responsible for an accident due to wrong weather forecast data. 
## do we perceive high complexity?  
Do we have a complex API with a lot of arguments? Do we have to coordinate many downstream systems to process the request? Those are good indicators that the pattern will help you.
 
   
# Forces: 
Main forces are:
 - completentess and correctness - all downstream systems must reliably be provided with calls
 - extensiblity, evolvibility - this pattern should be applicable for any API-Management solution agnostic of business aspects. So no matter if the business is insurance, automotive or healthcare or something else it should make sense to use
 - performance - it's critical for callers to have early feedback for not being blocked. 
 - scalability horizontal and vertical
    - vertical: we all hope for growing business, so this pattern should scale well for high volume
    - horizontal: any consumer being active in the same business context should be able to reuse the API
 - reliability, availability - in the context of API-Management these attributes are crucial for success
 
# Solution: 
the solution is implemented using JavaEE technology. We will not set the focus on technology but briefly explain how we solved it. 
## store 
Store has the goal to provide the request for reconciliation and error analysis reasons. Therefore the request should be persisted in raw format. 
A common technical solution today for Web APIs is to use REST and JSON format. As JSON-Documents might fail to be parsed to the target format (in our implementation Java objects), its recommended to store the request in raw format.
After storing in raw format we transform the JSON representation to Java using Jackson. The result is a domain model object for the bounded context of data processing. Why mapping and not reusing the API-Model throughout the processing?
The reasoning behind can be found in domain driven design. Every object has a bounded context. And the API Model has a different one than the processing domain model. 
The processing domain model, although being similar to a high degree, differs in terms of additional attributes which are not necessary or available on an API level. Those one include provider internal IDs of objects or providers audit information.
After mapping it into providers domain model it is persisted for further processing. 

## validate
Validation has two aspects in it: 
* basic formal validation (length, type, reg exp etc.)
* business specific validation rules 

A common way of implementing basic formal validation is following the JSR303 Bean validation approach. Standard validation rules like e.g. string length, not null or regex based can be used as well as own ones can be implemented according to JSR 303. This allows for a high cohesive, symmetric validation approach 
It`s good practice to encapsulate downstream systems (a.k.a core systems) through services. This keeps agility high in the core systems while consumers can rely on solid service contracts. 
The business specific validation rules usually are implemented by the downstream systems services as the services have to shield the core systems from bad data quality.
In order not to repeat ourselves validation services of the downstream systems can be used in this step.

The validate step allows to provide early response for the caller. After validation succeeded the caller should receive status ok. All following steps are irrelevant for the caller as only the internal processing of the provider is concerned. 
But there are still situations, where the provider is responsible for handling validation issues. Thus the validation step can be split into:
* earlyValidation: issues the consumer should handle
* lateValidation: issues the provider should handle

After earlyValidation the caller should receive an OK response while the providers processing can continue asynchronously.   

## enhance
Enhance should complete the data which was provided in the request with internal data the consumer does not know. Examples are an internal contract number for an insurance contract or an id of the insurance product or partner on which downstream systems rely.
Why should we design "enhance" in a dedicated step and not gather the data in the corresponding place where data is needed? Enhancing in a dedicated architectural component helps for better semantics and less redundancy.
E.g. the contract number needs to be available in several processing parts. If the first processing part would provide the enhance data this would result in less flexibility of changing processing sequence or might result in redundancy.  
    
## distribute

Distribution of an incoming request should support the following requirements:
* reliability - every downstream system must participate reliably
* mapping to downstream systems
* extensibility - a further downstream system should be easily integratable
* potentially downstream systems need to be called in a certain sequence

The requirement for reliability reminds us of the transactional concept of databases. In data storage we want to be sure that all data is persisted or nothing at all, the transaction should be rolled back.
We see some similarities here. Several downstream systems are part of the distribution. This compares well to tables. But as we live in a distributed world with loosely coupled http protocol there is no technical transaction available to ensure data consistency accross systems.
How can we control, that all downstream systems are called successfully and receive the data, they should? Therefore we introduced the notion of a "Business Transaction" in this pattern. 
A business transaction consists of n loosely coupled tasks. Each downstream system call is one task. A business transaction is complete if and only if every downstream system call is completed successfully.
Otherwise something needs to be done - either a technical retry or a human being has to fix something. 
But as this pattern supports for a structured way of straight through processing without human interaction it makes sense to supervice the status of the BusinessTransaction and its corresponding tasks for early information in error situations. 

Design:
- a BusinessTransaction aggregates Tasks
- the BusinessTransaction status is the sum of the Tasks status (init, inError, complete). Only if all Tasks are complete, the BusinessTransaction is complete  
- downstream systems are encapsulated by adapters.
- every adapater checks if it should participate in a BusinessTransaction and if yes, it registers itself as a BusinessTransactionTask.
- an adapter transforms the contract processing domain model to a downstream domain model and calls the service 

Extensibility: simply implement a new DistributionAdapter and let CDI discover it.
Sequence: annotate adapters with a priority. During CDI discovery of the adapters sort them accordingly. No priority annotated means, this adapter can be called at any time in parallel with others 

#Implementation
While some pattern relevant implementation details are mentioned in the corresponding chapters above, this chapter holds information that goes beyond the central aspects of the pattern which might help in some situations
## DomainModel
All domain objects are _Processable_. An example for a domain object is _Contract_ which holds the contract data of the API-Model and the additional fields relevant for the bounded context of contract processing.
## Steps and their Coordination
### Steps
Every step (store, validate, enhance, distribute) is implemented as a dedicated type. After successfully processing a step the _SVEDEngine_ will push the status forward towards the final destination status "PROCESSED". 

### Status
Every _Processable_ has a _ProcessStatus_. The _SVEDEngine_ checks, that e.g. *Enhance* step can only be started for a contract-data being in the status *validated*

### Coordination by _SVEDEngine_
The coordination of the four steps is done by the _SVEDEngine_ type. This _SVEDEngine_ is generically typed to a _Processable_, which means that every class that implements this interface can be processed by _SVEDEngine_.

## Downstream system encapsulation
Every downstream system is encapsulated by an adapter. Every adapter can participate in any of the steps *validate*, *enhance*, *distribute*. For each step a list of adapters is created by the 
_AdapterRegistry_. _AdapterRegistry_ is instantiated at runtime via CDI. CDI injects all implementations of an interface to the corresponding list of adapters. 
E.g. the list of ValidationAdapters is built by CDI by scanning the classpath for all types implementing _ValidationAdapter_. 
This supports easy extensibility with respect for the open close principle. We can easily add new adapter while the whole processing logic stays unchangeable. 
An adapter can implement several interfaces if it should participate in several steps.
  
## JSON-Storage
In case of wrong JSON-syntax mapping to Java fails. If we have to store the original JSON request before mapping to Java we have to interfere on an http-level.
JavaEE supports WebFilters which allow to interfere with http-requests before calling the service implementation. We implemented a webfilter which stores the JSON request to a database table.

## Idempotency
An idempotent call is a second call with the exact same parameters as the first call. There are two different notions of idempotency: 
* explicit idempotency: a part of the request is explicitly responsible for identifying a call. This is required if the business information in the payload is not unique, like e.g. in a money transfer between two bank accounts.
* implicit idempotency: a call is unique simply by the business data it consists. In the example here, where the SaaS provider creates a new contract and part of the payload (the external contract number) can be used to identify a unique call. 
In order to not polluting our domain database we implemented a filter as well, which would recognise an idempotent call.

## Dashboard
As the solution is for straight-through processing it does not require a UI by nature. But for administration reasons we implemented a web dashboard for business and technical supporters

## Monitoring and alarming
We centralized exception handling to very few methods. Those methods are monitored at runtime and in case of a call to an error handler operation we get notified in our Slackchannel.





