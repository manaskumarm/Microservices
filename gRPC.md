# gRPC for .NET — Online Food Ordering System

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)]() [![License](https://img.shields.io/badge/license-MIT-blue)]()

> A practical README that explains gRPC concepts, advantages, and a step‑by‑step .NET implementation for an **online food ordering system** (menu, orders, tracking, chat).

---

## Table of Contents

1. [Overview](#overview)
2. [Core gRPC Concepts](#core-grpc-concepts)
3. [Why gRPC? Advantages](#why-grpc-advantages)
4. [When to use gRPC (and when not to)](#when-to-use-grpc-and-when-not-to)
5. [Food Ordering System — Architecture & RPC choices](#food-ordering-system--architecture--rpc-choices)
6. [Example `.proto` (food.proto)](#example-proto-foodproto)
7. [Implementing with .NET — Step by step](#implementing-with-net---step-by-step)
   - Prerequisites
   - Create projects & solution
   - Add proto and package references
   - Implement server
   - Implement client
   - Local testing
   - Docker & Kubernetes notes
8. [Best practices & production concerns](#best-practices--production-concerns)
9. [Repository layout (suggested)](#repository-layout-suggested)
10. [License & Contacts](#license--contacts)

---

## Overview

gRPC is a modern high-performance Remote Procedure Call (RPC) framework that uses **HTTP/2** for transport and **Protocol Buffers (protobuf)** for the interface definition and binary serialization. It is language-agnostic and ideal for microservices.

This README uses an **online food ordering system** to demonstrate typical gRPC usage patterns: fetching menus, placing orders, real-time order tracking, and live chat between customer and delivery partner.

---

## Core gRPC Concepts

- **.proto files**: IDL (interface definition) where you define services and message types. The compiler generates server & client stubs.
- **Services & Methods**: RPC methods defined inside a `service` block. Methods are one of:
  - **Unary**: single request → single response.
  - **Server streaming**: single request → stream of responses.
  - **Client streaming**: stream of requests → single response.
  - **Bidirectional streaming**: both sides stream messages independently.
- **Channels**: client-side connection abstraction to a gRPC server.
- **Stubs / Generated code**: strongly typed client and server classes generated from `.proto`.
- **Metadata**: key-value headers for auth or tracing.
- **Interceptors**: middleware for logging, auth, metrics applied at client or server.
- **Deadlines / Cancellation**: time limits on RPC calls; pass `CancellationToken`.

---

## Why gRPC? Advantages

- **Performance**: HTTP/2 + protobuf binary encoding → faster and smaller payloads than JSON/REST.
- **Streaming support**: built-in first-class streaming (server/client/bidi).
- **Strong typing & code generation**: client & server stubs from `.proto` reduce bugs.
- **Polyglot**: works across many languages (.NET, Java, Go, Python, Node, etc.).
- **HTTP/2 benefits**: multiplexed requests over single connection, lower latency.
- **Good for internal microservice-to-microservice comms** and real-time features.

---

## When to use gRPC (and when not to)

**Use gRPC when:**
- Microservices require low‑latency, high‑throughput RPCs.
- You need streaming (live tracking, chat, telemetry).
- You want auto-generated clients for many languages.

**Avoid gRPC when:**
- You must support many third‑party browsers directly (use gRPC‑Web or REST as a gateway).
- Human-facing public APIs that require wide accessibility (offer REST/JSON alongside gRPC or use transcoding).
- Simple, occasional requests where JSON is sufficient and simpler.

---

## Food Ordering System — Architecture & RPC choices

Microservices:
- **UserService** — auth, profile (unary)
- **RestaurantService** — menus, availability (unary)
- **OrderService** — place order, update order (unary + server streaming for tracking)
- **DeliveryService** — assign rider, stream rider location (server streaming)
- **PaymentService** — process payments (unary)
- **ChatService** — live chat between customer & delivery (bidirectional streaming)

Typical mapping:
- Fetch menu, place order, payments -> **Unary**
- Live order / rider location updates -> **Server streaming**
- Customer uploads many images/feedback in batch -> **Client streaming**
- Live chat/status negotiation -> **Bidirectional streaming**

---

## Example `.proto` (food.proto)

```proto
syntax = "proto3";
package food;
option csharp_namespace = "FoodOrder.Proto";

import "google/protobuf/timestamp.proto";

service RestaurantService {
  rpc GetMenu(GetMenuRequest) returns (GetMenuResponse);
}

service OrderService {
  rpc PlaceOrder(PlaceOrderRequest) returns (PlaceOrderResponse);
  rpc TrackOrder(TrackOrderRequest) returns (stream OrderStatusUpdate);
}

service FeedbackService {
  // client streaming: client sends many feedbacks, server replies summary
  rpc SubmitBulkFeedback(stream Feedback) returns (FeedbackSummary);
}

service ChatService {
  // bidirectional: live chat
  rpc LiveChat(stream ChatMessage) returns (stream ChatMessage);
}

message GetMenuRequest { string restaurant_id = 1; }
message MenuItem { string id = 1; string name = 2; double price = 3; }
message GetMenuResponse { repeated MenuItem items = 1; }

message PlaceOrderRequest { string user_id = 1; repeated OrderItem items = 2; }
message OrderItem { string menu_item_id = 1; int32 quantity = 2; }
message PlaceOrderResponse { string order_id = 1; }

message TrackOrderRequest { string order_id = 1; }
message OrderStatusUpdate {
  string order_id = 1;
  string status = 2;          // e.g., "Preparing", "Out for Delivery"
  google.protobuf.Timestamp timestamp = 3;
}

message Feedback { string user_id = 1; string restaurant_id = 2; int32 rating = 3; string comments = 4; }
message FeedbackSummary { int32 total_submitted = 1; double average_rating = 2; }

message ChatMessage { string sender_id = 1; string receiver_id = 2; string message = 3; google.protobuf.Timestamp timestamp = 4; }
```

> Place this file under `Protos/food.proto` in both server and client projects (or share a proto project).

---

## Implementing with .NET — Step by step

### Prerequisites

- Install **.NET SDK** (recommended: recent LTS — .NET 7/8).
- `git`, Docker (optional), and an editor (VS Code / Visual Studio).

### 1) Create solution & projects

```bash
# Create solution
dotnet new sln -n FoodOrderGrpc

# Create a gRPC server project (a template exists):
dotnet new grpc -n FoodOrder.Server

# Create a console client (or another service) that will act as a gRPC client
dotnet new console -n FoodOrder.Client

# Add to solution
dotnet sln FoodOrderGrpc.sln add FoodOrder.Server/FoodOrder.Server.csproj
dotnet sln FoodOrderGrpc.sln add FoodOrder.Client/FoodOrder.Client.csproj
```

> You can also create separate class library project to contain `.proto` files and share that.

### 2) Add proto and package references

Edit `FoodOrder.Server.csproj` to include the proto and enable server codegen:

```xml
<ItemGroup>
  <Protobuf Include="Protos\food.proto" GrpcServices="Server" />
</ItemGroup>

<ItemGroup>
  <PackageReference Include="Grpc.AspNetCore" />
  <PackageReference Include="Grpc.Tools" PrivateAssets="All" />
</ItemGroup>
```

For the client project, include the proto with `GrpcServices="Client"` and add `Grpc.Net.Client`:

```xml
<ItemGroup>
  <Protobuf Include="Protos\food.proto" GrpcServices="Client" />
</ItemGroup>

<ItemGroup>
  <PackageReference Include="Grpc.Net.Client" />
  <PackageReference Include="Grpc.Tools" PrivateAssets="All" />
</ItemGroup>
```

### 3) Implement the gRPC server (Program.cs + service classes)

`Program.cs` (minimal example):

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddGrpc();
var app = builder.Build();

app.MapGrpcService<RestaurantServiceImpl>();
app.MapGrpcService<OrderServiceImpl>();
app.MapGet("/", () => "gRPC server is running.");

app.Run();
```

`RestaurantServiceImpl.cs` (example unary):

```csharp
public class RestaurantServiceImpl : RestaurantService.RestaurantServiceBase
{
    public override Task<GetMenuResponse> GetMenu(GetMenuRequest request, ServerCallContext context)
    {
        var res = new GetMenuResponse();
        res.Items.Add(new MenuItem { Id = "m1", Name = "Paneer Butter Masala", Price = 9.99 });
        return Task.FromResult(res);
    }
}
```

`OrderServiceImpl.cs` (example server streaming):

```csharp
public class OrderServiceImpl : OrderService.OrderServiceBase
{
    public override async Task TrackOrder(TrackOrderRequest request, IServerStreamWriter<OrderStatusUpdate> responseStream, ServerCallContext context)
    {
        var statuses = new[] { "Placed", "Preparing", "Out for Delivery", "Delivered" };
        foreach (var s in statuses)
        {
            await responseStream.WriteAsync(new OrderStatusUpdate { OrderId = request.OrderId, Status = s, Timestamp = Google.Protobuf.WellKnownTypes.Timestamp.FromDateTime(DateTime.UtcNow) });
            await Task.Delay(2000, context.CancellationToken); // simulate progress
        }
    }
}
```

**Notes:**
- Use `ServerCallContext.CancellationToken` to stop long-running streams if the client cancels.

### 4) Implement the client

Client code (console app) — unary & server streaming examples:

```csharp
using var channel = GrpcChannel.ForAddress("https://localhost:5001");
var restaurantClient = new RestaurantService.RestaurantServiceClient(channel);
var menu = await restaurantClient.GetMenuAsync(new GetMenuRequest { RestaurantId = "R1" });

var orderClient = new OrderService.OrderServiceClient(channel);
using var call = orderClient.TrackOrder(new TrackOrderRequest { OrderId = "order123" });

while (await call.ResponseStream.MoveNext())
{
    var update = call.ResponseStream.Current;
    Console.WriteLine($"Order {update.OrderId}: {update.Status}");
}
```

Client streaming example (submit multiple feedback items):

```csharp
var feedbackClient = new FeedbackService.FeedbackServiceClient(channel);
using var call = feedbackClient.SubmitBulkFeedback();
await call.RequestStream.WriteAsync(new Feedback { UserId = "u1", RestaurantId = "r1", Rating = 5 });
await call.RequestStream.WriteAsync(new Feedback { UserId = "u2", RestaurantId = "r1", Rating = 4 });
await call.RequestStream.CompleteAsync();
var summary = await call.ResponseAsync;
Console.WriteLine($"Submitted: {summary.TotalSubmitted} avg: {summary.AverageRating}");
```

Bidirectional example (chat) — client reads & writes concurrently:

```csharp
using var chat = new ChatService.ChatServiceClient(channel);
using var call = chat.LiveChat();

// reader
var readTask = Task.Run(async () => {
    while (await call.ResponseStream.MoveNext()) Console.WriteLine("Received: " + call.ResponseStream.Current.Message);
});

// writer
await call.RequestStream.WriteAsync(new ChatMessage { SenderId = "customer1", Message = "Hi! Where is my order?" });
await call.RequestStream.CompleteAsync();
await readTask;
```

### 5) Local testing

- Run server: `dotnet run --project FoodOrder.Server`
- Run client(s) and observe logs.
- For browser clients, consider **gRPC-Web** (add `Grpc.AspNetCore.Web` + `app.UseGrpcWeb()` and `.EnableGrpcWeb()` on mapped services) or a gateway (Envoy) to translate.
- `grpcurl` is helpful for manual testing (CLI tool).

### 6) Docker & Kubernetes (notes)

- Dockerize the server with multi-stage build; expose port (usually 5001 for TLS).
- In Kubernetes, use a `Deployment` and `Service` (ClusterIP/LoadBalancer). Use Envoy/Ingress that supports HTTP/2/gRPC for external traffic or gRPC‑Web conversion for browsers.
- Add liveness/readiness probes and resource limits.

---

## Best practices & production concerns

- **Use TLS** in production (mTLS if needed) and authenticate with JWT/OAuth for end users.
- **Set deadlines/timeout** on client calls to avoid resource leaks.
- **Respect message size limits** and set appropriate gRPC max message sizes.
- **Use interceptors** for centralized logging, auth, and metrics.
- **Tracing & metrics**: integrate OpenTelemetry / Prometheus to trace RPCs, latencies and error rates.
- **Retry & idempotency**: use idempotent operation design when enabling retries. Use client-side retries carefully (avoid compounded requests for non-idempotent calls like Payments).
- **Versioning**: avoid removing or renumbering fields in proto. Add new fields and keep backward compatible changes.
- **Connection & channel management**: reuse channels, rely on HTTP/2 multiplexing rather than many short-lived connections.
- **Back-pressure and flow control**: be mindful of streaming producers that can overwhelm consumers.

---

## Repository layout (suggested)

```
/ (repo root)
├─ FoodOrder.Server/         # gRPC server project
├─ FoodOrder.Client/         # example client (console or web client)
├─ Protos/                   # canonical proto files (shareable)
├─ docker/                   # Dockerfile and k8s manifests
└─ README.md
```

---

## License & Contacts

This repository is provided as an example. Add your preferred license (e.g., MIT) and contributor notes here.

---

> Want me to:
> - generate a ready-to-commit `README.md` file (I already added it here),
> - create `food.proto` and example C# files in the repo, or
> - produce Dockerfile / k8s manifests next?

