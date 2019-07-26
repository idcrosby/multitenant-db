# multitenant-db

A tutorial for how to setup multitenancy in a Database.

Specifically we will look at using hosted (AWS) SQL database as the first case.

All code and config examples will be hosted in this repo.

## Outline

### Intro

Why do we want a multitenant database?

### Use Cases

Specific examples of use cases for multitenancy in a Database

### Architecture Overview

Describe the architecture: Kubernetes (EKS) and Postgres (RDS)

[Architecture Diagram]

### Levels of Access Control

Describe the various levels of access control in place (IAM roles, RBAC, Database Users...)

### Factors to Condsider

- Security
- Performance
- Scale
- Cost
- Complexity

### Implementation

How to isolate clients using database schemas and dynamically created users.

Step By Step Guide to setup.

### Alternatives

Other options:

- Database per client
- Using a proxy
- IAM authentication
- Using other Databases?

### Conclusion
