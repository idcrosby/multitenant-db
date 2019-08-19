# Relational Database Multitenancy in the Cloud – A Guide

## Motivation

Engineering for multitenancy is an often overlooked subject in application architecture design. Yet when it comes to building entreprise software, this can often be a key requirement for clients. Unfortunately, too little resources regarding how to implement multitenancy are available online, and as such, this tutorial ought to provide you with a better comprehension of the subject, as well as reproducible steps to bring your own multi-tenant architecture to life.

## Series Overview

* **Part I: Introduction (this post):** This post is designed as a high-level introductory post covering the general concepts around multitenancy. We will explore the concept & jargon of multitenancy, its motivations and use cases, as well as the key points to address when designing for it.
* **Part II: Single-Schema Single-Instance multitenancy on AWS with IAM, Kubernetes & RDS (TODO:):** Presents an implementation of multitenancy that uses Row-Level Security coupled with virtual table views and AWS IAM/RDS to manage the access to the said views. Discusses the advantages and disadvantages of this family of solutions.
* **Part III: Multi-Schema Single-Instance multitenancy on AWS with IAM & RDS (TODO:):** Presents an implementation of multitenancy that uses the concept of schemas to logically isolate tenants, and discusses the advantages and disadvantage of this family of solutions.
* **Part IV: Database-Per-Tenant multitenancy on AWS with Amazon Aurora Serverless (TODO:):** Presents an implementation that leverages the advances of the serverless technologies to implement a concrete physical & logical separation between each tenant by hosting one serverless instance per tenant.

## Contributing

This tutorial is an open-source collaboration between several contributors. We are happy to welcome your contributions to this tutorial, whether through corrections, improvement & updates to the proposed solutions or new sections of the tutorial altogether (looking at you, Google Cloud and Azure people!). We are also looking for wonderful contributors who would like to volunteer Infrastructure as Code implementations of the infrastructures done in this tutorial :)

To contribute, fork [the repository](https://github.com/idcrosby/multitenant-db) and issue a PR! 🎉

---


# Introduction

## What is multitenancy?

Multitenancy, as defined in the [Gartner IT Glossary](https://www.gartner.com/it-glossary/multitenancy), is:

> _[A] reference to the mode of operation of software where multiple independent instances of one or multiple applications operate in a shared environment. The instances (tenants) are logically isolated, but physically integrated. The degree of logical isolation must be complete, but the degree of physical integration will vary. The more physical integration, the harder it is to preserve the logical isolation. The tenants (application instances) can be representations of organizations that obtained access to the multitenant application (this is the scenario of an ISV offering services of an application to multiple customer organizations). The tenants may also be multiple applications competing for shared underlying resources (this is the scenario of a private or public cloud where multiple applications are offered in a common cloud environment)._

## What are the core desired properties of multitenant architectures?

At it's core, multitenant architectures are about **cost**, **security**, **access control**, and **isolation of resources**, whether data or infrastructural in nature. Moreover, like any data system, multitenant architectures should also have the desired properties of big data systems[1](https://www.oreilly.com/library/view/big-data-principles/9781617290343/)[2](https://dataintensive.net/)

* Reliability
* Scalability
* Generalization
* Minimal development complexity
* Minimal operational complexity
* Minimal maintenance
* Extensibility
* Debuggability
* Support for ad-hoc queries
* Low latency read and updates

We will explore the **security**, **access control**, **isolation of resources**, **development complexity**, **operational complexity**, **scalability** and **per tenant cost** aspects of these solutions.

## When should multitenancy be considered?

Multitenancy architecture principally serve two purposes – reduce the cost per tenant and isolate your tenants from each other to keep them from accessing each other's data.

In short story, as soon as you have multiple users using the same application running on shared resources (servers, databases, etc.), you are using multitenancy.

For instance, if you are building a B2C application communicating with the cloud, your application is most likely a multitenant application, where a single database serves all of your users and stores all of their information. In these cases the isolation of tenants is oftentimes a second concern rather than a priority, and databases are accessed directly by a single connection and a single user. No real security is enforced, and it is the application programmer's responsibility to avoid disasters by ensuring that the user only has access to their data and the common data.

In B2B contexts, strong security and isolation are very often required, as each tenant may have sensitive information that they do not want to be divulgated. Prime use cases are when dealing with protected health, legal, financial, or otherwise sensitive information whose leak could cause mild to serious damages to the affected parties, whether it is the service provider or the owner of the data. Sometimes however, the need for multitenancy can be mostly cost-driven, as we will discuss in Part II's use case.

This series of articles discusses implementations of the latter case where implementing stronger security mechanisms, ensuring isolation of tenants and proper Role Based Access Control (RBAC) are paramount.

## Considerations when choosing a multitenancy model

Choosing a multitenancy model requires to take considerations of data governance, operability and requirements into account. As outlined by [Flater on Software Engineering Stack Exchange](https://softwareengineering.stackexchange.com/a/380432), you should consider the following when designing for multitenancy:

* When upgrading your database schema, do you wish to give your tenant the choice of upgrade or
will you enforce upgrade for all of your tenants?
* When a customer require a restore of their data, do you want to have the ability to restore only that
customer's data or will you need to refresh all of your tenant simultaneously?
* Will you host common data across your tenants? Given a concept of schema versioning and a state where
your customers may be at different versions of your schema, how do you handle the common data?
* How do you handle failures and rollbacks over all of your tenants data?

Additionally, the following concerns apply:

* When a customer requires you to purge their data according to data privacy laws, do you want to be able to remove that customer's data indepedently of other customers? Do you want to anonymize said data?
* How do you ensure that your data accesses and changes are auditable in the case of an audit?
* Do you want to do market-wide analytics and monitoring?
* How important is it to your tenants that their information be physically isolated? Are they ready to shoulder the extra cost associated with this choice?

## Database Multitenancy models

This section is greatly inspired by Microsoft's marvelous article on the subject. For another perspective on
multitenancy, go read:

https://docs.microsoft.com/en-us/azure/sql-database/saas-tenancy-app-design-patterns

### Database-Per-Tenant Multitenancy

Database-Per-Tenant Multitenancy is the simplest approach to multitenancy. It offers a strong guarantee of isolation
through physical isolation. In this model, each of your customers have their own database. 

This solution eases pretty much every aspect of development and operations. The physical isolation eases per-tenant operations like configuration, optimization and customization of the databases and the data schema, as well as data governance, access control, security, etc.

On the flipside, this solution increases the complexity of market-wide analytic delivery to your customers. Indeed you will need to load market-wide pre-aggregations in each of your instances, and potentially maintain another data store earlier in the ETL pipeline in order to backfill the data over time.

When it comes to the cost, Database-Per-Tenant Multitenancy is the most expensive type of multitenancy due to the need for one instance per tenant, throwing any possible economies of scale to the trash. To fill this gap, the Microsoft article above advises to use resource pooling. The problem is, if you use resource pooling chances are that you are managing your own database instances by yourself, sacrificing the advantages of managed services like AWS RDS and Amazon Aurora. Now your problem of cost becomes one of DevOps and operational complexity, which in most cases is not a problem you want to have on your hands. 

Another solution to the cost problem is to opt for a serverless option. In his opiniated piece [Does multi-tenancy really matter anymore?](https://diginomica.com/does-multi-tenancy-really-matter-anymore), Claus Jepsen, member of the Forbes Technology Council explains why he thinks that multitenancy solutions at the application level have become outdated:

"Today we have huge clouds offering cheap infrastructure and tooling for automatic scaling in and out, creating new instances and tearing them down again without any of the manual interventions that used to be necessary. There are new services popping up every day, like databases being charged per transaction rather than per instance, which means you do not have to pay in the event the tenant is not using the database. [...] The economic necessity for multi-tenancy at the application tier has gone away."

Granted, the cloud services pricing landscape has greatly evolved over the last decade, but the fact is, so has the application market. Nowadays, non-enterprise applications sell at a fraction of what applications used to cost, with most prices ranging between $0.99 and $15.99 monthly. With this price range, you simply cannot afford a database that costs $14.40 per month _per user_ (AWS RDS, db.t3.micro, smallest at a rate of $0.02 per hour).

"But what about serverless?" says the SF trendy tech hipster.

Well serverless is great for a lot of things, but it also has its downsides. Anyone who has been bitten by the coldstart issue with AWS Lambdas or Firebase functions know what I'm talking about. As of August 2019, serverless technologies are still insufficient performance-wise to answer client-facing applications. For instance Aurora Serverless takes between 30-60 seconds to wake up from its slumber when coldstarted ([source one](https://www.jeremydaly.com/aurora-serverless-the-good-the-bad-and-the-scalable/), [source two](https://www.percona.com/blog/2019/01/09/amazon-aurora-serverless-the-sleeping-beauty/)). _30 to 60 freaking seconds!!!_ And that's without talking about the rest of the latency that you are adding from the rest of your application.

If you can predict your queries ahead of time, preloading your caches with the results of the queries could drastically reduce the chances of this issue being problematic, but you'd have to do a major job at making sure that at no point your users ever have to reach the database.

If that doesn't stop you, and you want to benefit from the obvious cost advantages of serverless, I greatly recommend the following article for a price comparison and more in-depth analysis of AWS's Serverless RDBMS, Amazon Aurora Serverless:

https://www.jeremydaly.com/aurora-serverless-the-good-the-bad-and-the-scalable/

If this is a solution that suits your needs, you will find your joy in [Part IV](TODO:)

In summary, the Database-Per-Tenant Multitenancy model is the simplest to implement and operate. It offers the maximum isolation and allows each of your clients to have their own resources and sandbox to play with. Cost-wise, it is the most expensive, and may require you to pull some tricks to make it affordable. Another option to curb the cost is to pass the bill to your customers directly, either as a premium or as part of your pricing.

### Single-schema single-instance multitenancy

Single-schema single-instance multitenancy (SS-SI) is the most common type of multitenancy. As explained above, it is the default mode of multitenancy that most applications use to reduce the cost-per-tenant. It is the cheapest option as you only need to have a single instance running and scale according to your load and storage needs. However this comes at the cost of isolation, for this is the model with the least physical and logical isolation of all the models. 

In this model, your users all share the same tables, with their records intertwining between each other. The discriminant factor is a column that is common to all tables and that specifies the unique identifier of each tenant. A single-schema single-instance database looks like the following:

<TODO: Insert image here>

To compensate for the reduced physical and logical isolation, Row Level Security (RLS) has to be implemented. Row Level Security, as its name suggests, allows to control access to records in a database table. Some RDBMS support Row Level Security natively, like SQL Server with the `CREATE SECURITY POLICY` SQL statement, while some others don't. As you will see in Part II, Row Level Security can be implemented through the use of VIEWs and appropriate user access control, and be coupled with the Cloud Provider's identity management services to reinforce security and improve monitoring, debuggability and operability.

Besides cost-reduction, single-schema single-instance multitenancy's biggest selling point is the ability to store all of your customers data in a single instance, thereby making it easier to do market-wide analytics and ad-hoc queries on your data, and serve it to your customers at no extra cost.

Single-schema single-instance multitenancy also improves the data schema migration time and thus your ability to deploy. For more details on how delays in migration can affect your team, please read [this wonderful article](https://influitive.io/our-multi-tenancy-journey-with-postgres-schemas-and-apartment-6ecda151a21f#.vfxsvbsrg) by Brad Robertson on his experience with Multi-schema single-instance multitenancy at Influitive.io.

This model is not all flowers and rainbows (rainbows! 🌈) however, as it makes it harder for about every other aspect of development, operationability, debuggability and data governance. For one, it makes it much harder to create setups where different clients may have different versions of the data schema running in the database. Second, it makes virtually all single-tenant operations like restoring data, purging data, extracting data for audits and exports much harder than the Database-Per-Tenant approach. Restoring previous version is especially a nightmare if there is no immutable data store from which to backfill because you can't simply rollback your instance to a previous snapshot in time without affecting your other tenants.

Both single-schema single-instance multitenancy and multiple-schema single-instance multitenancy come with an other, generally less well known issue: resource sharing. Contrarily to the Database-Per-Tenant model, given that all of your users share common resources, if some of them hoard or abuse the said resources, other users' service quality may drastically drop.

Horizontal scaling can be approached by sharding your dataset according to the values in the tenant identification column. As to whether scaling and sharding your database can affect the ability of the setup to perform market-wide analytics, I (@philippefutureboy) do not know sufficiently on the subject to give you an answer. It is very likely that a solution like duplication of market-wide preaggregated, larger grained tables across all the shards may solve the problem. If you know more about sharding and query performance, feel free to let us know in the comments or to contribute to the repository 🚀

As for performance, it is unclear if single-schema single instance multitenancy performs better or worse than its brother multi-schema single-instance.

In summary, single-schema single-instance multitenancy provides a cheaper alternative to the Database-Per-Tenant approach by sacrificing the physical isolation and increasing the development and operability complexity of the solution. By nature, it eases market-wide analytics and ad-hoc queries by using a centralized data schema and data store. Its Row Level Security model allows for a more finely grained security and access control, but increases the complexity of any operation other than routine CRUD operations.

### Multi-schema single-instance multitenancy

Multi-schema single-instance multitenancy (MS-SI) is the brother of single-schema single-instance multitenancy and sports most of the same characteristics. MS-SI comes with the same reductions in costs by centralizing all of your tenants in a single instance (or multiple as you scale). Similarly, it does not provide physical isolation, but still slightly improve on SS-SI's isolation by providing a logical level isolation in the form of having one schema per tenant:

<TODO: Insert image here>

Separation by schemas allow to create small universes within your instances that are completely separated from each other, and can easily have their own specific data schema and access control rules. With this separation, you can treat individual schemas as if they were physical instances, thereby gaining a lot of the advantages of physical instances.

Namely, CRUD operations become easier to manage, different schemas can have different data schemas, be on different versions, or be optimized individually. Data governance and operability greatly improves also because you can simply act at the schema level and process each schema individually. Scalability also becomes easier because you can split your database simply by moving schemas around between instances.

But there are tradeoffs. If you didn't read [Brad Robertson's article](https://influitive.io/our-multi-tenancy-journey-with-postgres-schemas-and-apartment-6ecda151a21f#.vfxsvbsrg) linked above by now, well here's the TL;DR: _It seems like RDBMS are not made to host 100s or 1000s of schemas and much less 100 of thousands or even millions of tables. They are just not made for that._ Performance-wise, that's not what they have been optimized for, and if you want to maximize your performance on CRUD operations as well as development operations, you'll probably need a very very good DBA, and maybe to step out of managed services altogether. Don't take my word for it, but be aware that this is a very real risk that you can have while scaling up with schemas. Another potential solution to this problem would be to have less tenants by instance, and find the best point satisfying both cost and performance constraints.

Market-wide analytic also become more complex. You will either need to load each schema with its own copy of the market-wide aggregated data, centralize the data in a single "common" schema, or do your queries across schemas. From my research, it seems like queries across schemas are just as performant as queries across large tables, but I don't have any performance data to back up my claims.

In summary, multi-schema single-instance multitenancy provides a solution that attempts to get the best of both of its brothers, but that can be held back by performance issues. These performance issues can be mitigated through careful optimization or by reducing the number of tenants by instance, thereby reducing the cost benefits of this model. Its schema level security allows to gain all the benefits of the previously discussed Row Level Security model with less complexity.

### Comparison

| Measure | Database-Per-Tenant | Single-Schema Single Instance | Multiple-Schema Single Instance |
|---------|---------------------|-------------------------------|---------------------------------|
| Scale | Very high, most simple | Unlimited, most complex | Unlimited, medium complexity |
| Isolation | Very High | Low, needs lots of extra work | Medium, can require lots of extra work |
| Cost per tenant | Very High to Medium (serverless) | Very low | Very low |
| Performance | Very Good | Very Good, needs horizontal scaling & sharding | Great performance at lower scales, undetermined at higher scales and loads |
| Development Complexity | Low | High | Medium |
| Operational Complexity | Low | High | Low |
| Data Governance & Audit | Low | High | Medium |
| Market-wide Analytics | Complex | Easy, may get more complex when scaling. | Medium. May get more complex when scaling |

# Wrapping Up Relational Database Multitenancy in the Cloud – Part I

In this post, we learned about the concept of multitenancy and how it allows us to reduce the cost of operations of an application hosting multiple tenants by leveraging different models of multitenancy. We discussed the considerations of multitenancy designs, its important aspects, and we dove deep into each of the models, gaining a better understanding of their advantages and disadvantages and relative tradeoffs.

In Part II, we will dive into the implementation of Single-Schema Single-Instance multitenancy on AWS with IAM, Kubernetes & RDS. Specifically, we will provide a step by step tutorial explaining how to build this solution on Amazon Web Services.

Thank you for reading this post, and see you in [Part II](TODO:) 🚀