# GraphQL Implementation in .NET Core – A Complete Guide

This article explains how to implement **GraphQL** in a **.NET Core** application. It covers the basics of GraphQL, its advantages, and provides a step-by-step guide for setting up and building a GraphQL API using .NET Core.

---

## 📖 Table of Contents

- [Introduction](#introduction)
- [What is GraphQL and its Advantages](#what-is-graphql-and-its-advantages)
- [Implementation with .NET Core (Step by Step)](#implementation-with-net-core-step-by-step)
- [References](#references)
- [Conclusion](#conclusion)

---

## 🟠 Introduction

With the increasing demand for efficient and flexible APIs, **GraphQL** has emerged as a powerful alternative to traditional REST APIs. It allows clients to request exactly the data they need, making API interactions faster and more efficient. In this article, we will walk through how to implement GraphQL in a **.NET Core** application, enabling developers to build scalable, maintainable, and high-performance APIs.

---

## ✅ What is GraphQL and its Advantages

**GraphQL** is a query language for APIs that enables clients to specify the structure of the response data. It was developed by Facebook and is widely adopted for its efficiency and flexibility.

### Key Features:
- **Client-driven data requests** – Clients request exactly what they need.
- **Strong typing** – The schema defines types and relationships.
- **Single endpoint** – No need for multiple REST endpoints.
- **Real-time updates** – Supports subscriptions for live data changes.
- **Introspection** – Built-in documentation and query exploration tools.

### Advantages:
✔ Reduces over-fetching and under-fetching of data  
✔ Improves performance by minimizing unnecessary data transfer  
✔ Easier to evolve APIs without breaking existing clients  
✔ Provides self-documenting APIs  
✔ Supports real-time communication with subscriptions

---

## ✅ Implementation with .NET Core (Step by Step)

We will use the **Hot Chocolate** GraphQL library, one of the most popular libraries for GraphQL in .NET.

### 🟢 Step 1 – Create a New .NET Core Project
```
dotnet new webapi -n GraphQLDemo
cd GraphQLDemo
```
### 🟢 Step 2 – Add Required Packages
```
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.AspNetCore.Playground
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```
Note: Check if you need all packages.

### 🟢 Step 3 – Define the Model
```
namespace GraphQLDemo.Models
{
    public class Book
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Author { get; set; }
    }
}
```
### 🟢 Step 4 – Setup the DbContext

Create Data/AppDbContext.cs:
```
using GraphQLDemo.Models;
using Microsoft.EntityFrameworkCore;

namespace GraphQLDemo.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
        public DbSet<Book> Books { get; set; }
    }
}
```


### 🟢 Step 5 – Define the GraphQL Schema

Create GraphQL/Query.cs:
```
using GraphQLDemo.Data;
using GraphQLDemo.Models;
using HotChocolate;
using HotChocolate.Data;
using System.Linq;

namespace GraphQLDemo.GraphQL
{
    public class Query
    {
        [UseDbContext(typeof(AppDbContext))]
        [UseFiltering]
        [UseSorting]
        public IQueryable<Book> GetBooks([ScopedService] AppDbContext context)
        {
            return context.Books;
        }
    }
}

```

### 🟢 Step 8 – Run and Test
```
dotnet run
```

Navigate to https://localhost:5001/graphql and use the GraphQL Playground to test:
```
// To get all books
query {
  books {
    id
    title
    author
  }
}

// To get all books which title contains 1984
query {
  books(where: { title: { contains: "1984" } }) {
    id
    title
    author
  }
}

```

**program.cs**
```
using GraphQLDemo.Data;
using GraphQLDemo.GraphQL;

var builder = WebApplication.CreateBuilder(args);
// Register services
builder.Services.AddSingleton<OrderService>();
// GraphQL setup
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>();
    
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseInMemoryDatabase("BooksDb"));
    
var app = builder.Build();

// GraphQL API & UI at /api/books
app.MapGraphQL("/api/books");

// Conditionally expose Playground UI only in development
if (app.Environment.IsDevelopment())
{
    // Option 1: Direct Playground mapping
    app.MapGraphQLPlayground("/playground");
}

app.Run();

// 👇 Add this so WebApplicationFactory can find Program
public partial class Program { }

//You may change little bit in program.cs based on mutation(AddMutationType), querytype(AddQueryType) file you creates.
```

**dockerfile**
```
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build

WORKDIR /app

COPY . .

RUN dotnet tool install -g Microsoft.DataApiBuilder

ENV PATH="${PATH}:/root/.dotnet/tools"

CMD ["dab", "start"]
```
**Client using React axios library**
```
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const BookList = () => {
  const [books, setBooks] = useState([]);
  const [error, setError] = useState(null);

  // URL of the GraphQL API
  const GRAPHQL_ENDPOINT = "https://localhost:5001/api/books"; // Adjust port as needed

  useEffect(() => {
    const fetchBooks = async () => {
      try {
        const query = `
          query {
            books {
              id
              title
              author
            }
          }
        `;

        const response = await axios.post(
          GRAPHQL_ENDPOINT,
          { query: query },
          {
            headers: {
              'Content-Type': 'application/json',
              // Authorization: `Bearer YOUR_JWT_TOKEN`, // Uncomment if using JWT
            }
          }
        );

        if (response.data.errors) {
          setError(response.data.errors[0].message);
        } else {
          setBooks(response.data.data.books);
        }
      } catch (err) {
        setError(err.message);
      }
    };

    fetchBooks();
  }, []);

  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h2>Book List</h2>
      <ul>
        {books.map((book) => (
          <li key={book.id}>
            <strong>{book.title}</strong> by {book.author}
          </li>
        ))}
      </ul>
    </div>
  );
};

export default BookList;

```

## ✅ Unit and Integration Testing
Create a test project:
```
dotnet new xunit -n GraphQLDemo.Tests
dotnet add GraphQLDemo.Tests reference GraphQLDemo
```
Example test for the query resolver:
```
using GraphQLDemo.GraphQL;
using GraphQLDemo.Models;
using Moq;
using Xunit;
using Microsoft.EntityFrameworkCore;
using System.Linq;

public class QueryTests
{
    [Fact]
    public void GetBooks_ReturnsBooks()
    {
        var options = new DbContextOptionsBuilder<GraphQLDemo.Data.AppDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDb")
            .Options;

        using var context = new GraphQLDemo.Data.AppDbContext(options);
        context.Books.Add(new Book { Id = 1, Title = "Test Book", Author = "Author" });
        context.SaveChanges();

        var query = new Query();
        var result = query.GetBooks(context).ToList();

        Assert.Single(result);
        Assert.Equal("Test Book", result[0].Title);
    }
}

```
✅ Integration Testing Approach

✔ Use WebApplicationFactory<T> to bootstrap the API in tests

✔ Seed the in-memory database before each test

✔ Test authentication flows with JWT tokens

✔ Validate GraphQL queries using HTTP clients

## 📚 References

Hot Chocolate – GraphQL for .NET

GraphQL Official Documentation

Entity Framework Core

.NET Documentation

## ✅ Conclusion

GraphQL provides a modern, efficient way to interact with APIs, and integrating it with .NET Core using libraries like Hot Chocolate makes development faster and easier. With GraphQL, you can build flexible, scalable, and performant APIs that cater to client needs without unnecessary overhead. This step-by-step guide equips you to start implementing GraphQL in your next .NET Core project.
