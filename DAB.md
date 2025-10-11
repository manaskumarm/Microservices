# Data API Builder (DAB), GraphQL & Cosmos DB – A Complete Overview

This article explores how to build a scalable, flexible, and efficient API layer using **Data API Builder (DAB)** integrated with **GraphQL** and **Azure Cosmos DB**. It covers architectural patterns, implementation guidelines, advantages, and real-world use cases.

---

## 📖 Table of Contents

- [Introduction](#introduction)
- [What is Data API Builder (DAB)?](#what-is-data-api-builder-dab)
- [GraphQL and its Role](#graphql-and-its-role)
- [Why Cosmos DB?](#why-cosmos-db)
- [Architecture Overview](#architecture-overview)
- [Setting Up DAB with Cosmos DB](#setting-up-dab-with-cosmos-db)
- [Exposing GraphQL Endpoints](#exposing-graphql-endpoints)
- [Handling Authentication and Security](#handling-authentication-and-security)
- [Performance and Scaling Considerations](#performance-and-scaling-considerations)
- [Common Use Cases](#common-use-cases)
- [Challenges and Best Practices](#challenges-and-best-practices)
- [Conclusion](#conclusion)

---

## 🟠 Introduction

Modern applications require fast, flexible, and data-driven APIs that can evolve without the need to rewrite the backend constantly. **Data API Builder (DAB)** provides a configuration-driven approach to expose APIs over databases like **Cosmos DB**. Combined with **GraphQL**, it offers developers a powerful way to query data efficiently while reducing server-side complexity.

---

## ✅ What is Data API Builder (DAB)?

**Data API Builder (DAB)** is an open-source framework that allows you to quickly expose your database as REST or GraphQL endpoints without writing backend code.

### Key Features:
- Configuration-driven API generation
- Support for GraphQL and REST endpoints
- Integration with various databases, including Azure Cosmos DB, SQL Server, PostgreSQL, etc.
- Authentication and authorization capabilities
- Suitable for rapid prototyping and production-ready solutions

**Benefits:**
- Reduces boilerplate code
- Improves developer productivity
- Seamless integration with modern frontend frameworks

---

## ✅ GraphQL and its Role

**GraphQL** is a query language for APIs that enables clients to request only the data they need.

### Advantages:
- Efficient data fetching
- Strong typing and validation
- Introspection for self-documentation
- Simplified client-server interactions

With DAB, GraphQL endpoints are automatically generated, enabling seamless interaction between the client and database.

---

## ✅ Why Cosmos DB?

**Azure Cosmos DB** is a globally distributed, multi-model database service designed for high availability and low-latency access.

### Features:
- Horizontal scaling and global replication
- Multi-model support (SQL, MongoDB API, Cassandra, Gremlin, Table)
- Serverless or provisioned throughput options
- Strong consistency and tunable consistency models

It pairs perfectly with DAB for cloud-native applications requiring fast, scalable data access.

---

## 🏗 Architecture Overview

```text
Client (Web/Mobile)
      ↓
DAB GraphQL Endpoint
      ↓
Cosmos DB (SQL API)
```

## 🏗 Implementation

**schema.graphql**
```
type Store @model(name: "Store") {
  id: ID!
  storeId: String!
  name: String
  status: Boolean
}
```

**dab-config.json**
```
{
  "$schema": "https://github.com/Azure/data-api-builder/releases/download/v1.5.56/dab.draft.schema.json",
  "data-source": {
    "database-type": "cosmosdb_nosql",
	//"connection-string": "${COSMOS_CONNECTION_STRING}", TODO: Container Apps → Your App → Settings → Environment Variables.
    "connection-string": "AccountEndpoint=https://dev-cosmosdb.documents.azure.com:443/;AccountKey=;",
    "options": {
      "database": "servesync",
      "schema": "schema.graphql"
    }
  },
  "runtime": {
    "rest": {
      "enabled": false,
      "path": "/api",
      "request-body-strict": true
    },
    "graphql": {
      "enabled": true,
      "path": "/graphql",
      "allow-introspection": true
    },
    "host": {
      "cors": {
        "origins": [],
        "allow-credentials": false
      },
      "authentication": {
        "provider": "StaticWebApps",
        "allowedRoles": ["anonymous"]
      },
      "mode": "development"
    },
    "telemetry": {
      "application-insights": {
        "enabled": false,
        "connection-string": ""
      }
    }
  },
  "entities": {
    "Store": {
      "source": {
        "object": "stores",
        "key": "storeId"
      },
      "graphql": {
        "enabled": true,
        "type": {
          "singular": "Store",
          "plural": "Stores"
        }
      },
      "rest": {
        "enabled": false
      },
      "permissions": [
        {
          "role": "anonymous",
          "actions": [
            { "action": "*" }
          ]
        }
      ]
    }
  }
}
```

**dockerfile**
```
If you want to run your DAB as a container in AKS or ACA then you can follow otherwise it is optional.
FROM mcr.microsoft.com/azure-databases/data-api-builder:latest

WORKDIR /App

COPY dab-config.json /App/dab-config.json
COPY schema.graphql /App/schema.graphql

EXPOSE 5000

CMD ["start", "--config", "/App/dab-config.json"]

Note: Keep above files in one directory.
```
**Installation**
Download and install the DAB cli in your local system(if you do not use dockerfile otherwise docker will take care). 
To verify if it is installed properly, execute command in powershell like:
```
dab --version
```
Once you see the version properly, it is confirmed that it installed properly. Next, initialize the DAB component(as mentioned earlier put config, schema in same directory and go to same directory) 

```
dab init
```

<img width="954" height="464" alt="image" src="https://github.com/user-attachments/assets/46108a6a-a001-47c9-bcf7-798cdc6f7282" />

You may encounter with cosmos db SSL issues because in local cosmosdb runs in https port with self signed certificate. 
To resolve the issue you may use Azure URL or export the self signed cert from cosdmosb to your container.

**References**
https://learn.microsoft.com/en-us/azure/data-api-builder/deployment/how-to-publish-container-apps
