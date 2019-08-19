# multitenant-db

A tutorial for how to setup multitenancy in a Database.

Specifically we will look at using hosted (AWS) SQL database as the first case.

All code and config examples will be hosted in this repo.

## Outline

Part 1: Introduction to Multi-tenancy & Multi-tenancy models
Part 2: Single-Database Single-Schema Multi-tenancy
Part 3: Single-Database Multiple-Schema Multi-tenancy


### Intro

Why do we want a multitenant database?

#### Use Cases

Specific examples of use cases for multitenancy in a Database

#### Factors to Consider

- Security
- Performance
- Scale
- Cost
- Complexity

#### Alternatives

Other options:

- Database per client
- Using a proxy
- IAM authentication
- Using other Databases?

### Architecture Overview

Describe the architecture: Kubernetes (EKS) and Postgres (RDS)

[Architecture Diagram]

### Levels of Access Control

Describe the various levels of access control in place (IAM roles, RBAC, Database Users...)

### Implementation

How to isolate clients using database schemas and dynamically created users.

Step By Step Guide to setup.

### Conclusion
