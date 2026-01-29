Create a branch called `episode-1` based on the `master` branch.

Welcome to Episode 1 of this gRPC series. In this episode, we’re laying the foundation. By the end of this video, you’ll understand why gRPC exists in the first place, when it actually makes sense to use it, how proto-first contracts work, and how all of this fits naturally into an ASP.NET Core application. We’ll also set the stage for building a real unary gRPC service and calling it from a .NET client, but first, we need to get the “why” straight.

To keep things realistic, we’re going to work in an airline domain and build a simple Flight Directory service. This is intentional, because airline systems are a perfect example of where gRPC really shines.

Think about a real airline system for a second. You don’t have just one app. You have a booking system talking to flight operations, flight operations talking to check-in, check-in talking to gate services, and dashboards pulling data from multiple internal services all the time. These systems talk to each other constantly, and they do it behind the scenes. This is service-to-service communication, not public APIs for browsers.

In that kind of environment, latency matters. If a gate changes or a flight is delayed, you don’t want slow, chatty calls piling up. You also need strong contracts, because many teams are working on different services, often in different languages, and they all need to agree on what data is exchanged and how.

That brings us to the idea of a contract. In this context, a contract is a formal agreement between a client and a server. It clearly defines what operations are available, what data the client sends, what data the server returns, how that data is structured and typed, and even what kinds of errors can happen. In other words, it’s the rulebook for how two services talk to each other.

With gRPC, this contract is not something vague or informal. It’s defined explicitly in a `.proto` file. That file becomes the single source of truth. It’s language-neutral, meaning the same contract works for C#, Java, Go, Python, or whatever else you’re using. It’s strongly typed, versionable, and enforced at compile time. If you break the contract, you find out early, not at runtime in production.

This is very different from what we’re used to with REST. In many REST systems, contracts are implicit. They live in documentation, maybe in an OpenAPI spec if you’re lucky, and often they’re only enforced when something blows up at runtime. Clients can drift out of sync, and type mismatches don’t show up until it’s too late.

gRPC also gives you some serious technical advantages under the hood. First, it runs on HTTP/2. HTTP/2 allows multiple requests to be sent over a single connection, it compresses headers, and it uses the network much more efficiently. Compare that to classic REST, which usually runs on HTTP/1.1, where each request may require its own connection and comes with more overhead. In airline systems where services are constantly talking to each other, HTTP/2 alone can significantly reduce latency.

On top of that, gRPC uses Protocol Buffers, or protobuf, as its data format. Protobuf is binary, compact, and fast. The payloads are much smaller than JSON, and serialization and deserialization are significantly faster. JSON is great because it’s human-readable, but it’s verbose and slower to parse. For internal systems, where humans don’t read the payloads and performance matters more than readability, protobuf is a big win.

Another major benefit is strong typing. The `.proto` file defines exact data types, optional versus required fields, nested objects, and collections. There’s no guessing involved. With REST, you often rely on conventions and documentation, and mismatches show up only at runtime. With gRPC, many breaking changes are caught at compile time, which is exactly where you want them.

Finally, gRPC has first-class support in ASP.NET Core. You get native hosting, automatic code generation for both servers and clients, and full integration with dependency injection, logging, and middleware. It doesn’t feel like a bolt-on library. It feels like a natural part of the .NET ecosystem.

So if you want a simple mental model, think of gRPC as REST for internal systems that really care about performance, consistency, and strong contracts. In the next part of this episode, we’ll start turning these ideas into something concrete by designing our first proto-first contract for the Flight Directory service and building it step by step.

---

Alright, let’s keep rolling and zoom in on the actual domain we’re going to build.

At the heart of this episode, we’re working with a very focused domain slice called the Flight Directory. The idea here is simple and very intentional. This service answers questions like “What are the details of flight AC101?” or “Which flights go from YUL to YYZ on February 10th, 2026?” Nothing more, nothing less. We’re not booking tickets, we’re not assigning seats, and we’re not charging credit cards. We’re just exposing reliable flight information in a clean, well-defined way.

To support that, we only need a few core domain entities. Conceptually, we’re dealing with airports, aircraft, and flights. An airport has a code, a name, and a city. An aircraft has a model and a seat capacity. A flight ties everything together: it has a flight number, a route defined by an origin and a destination airport, a schedule with departure and arrival times, and the aircraft that operates the flight. Keeping this small and focused helps us learn gRPC without getting lost in business complexity.

Now let’s look at how the solution itself is structured. Even though this is a demo, we’re still going to respect clean architecture ideas.

```txt
Airline.FlightDirectory
│
├── Airline.FlightDirectory.Api        (ASP.NET Core gRPC Server)
├── Airline.FlightDirectory.Contracts  (.proto files)
├── Airline.FlightDirectory.Domain     (Pure domain models)
```

What you’re seeing here is a very deliberate separation of concerns. The API project hosts the actual gRPC server. The Contracts project owns the `.proto` files, which define the external contract of the service. The Domain project contains pure domain models with no gRPC, no ASP.NET, and no infrastructure concerns. Even in a small tutorial, this structure pays off because it forces us to think clearly about boundaries.

Now we’re ready for the first real gRPC step: defining the proto contract. This is what people mean when they say “proto-first.” We design the contract first, and the code comes later.

Before we even write a `.proto` file, we need to make sure the Contracts project is set up correctly. In the Contracts project `.csproj` file, we add the required gRPC and protobuf packages.

```
<PackageReference Include="Google.Protobuf" Version="3.25.3" />
<PackageReference Include="Grpc.Tools" Version="2.76.0" PrivateAssets="All" />
<PackageReference Include="Grpc.Core.Api" Version="2.76.0" />
```

What this does is prepare the project to understand protobuf definitions and to generate C# code from them. `Google.Protobuf` gives us the core protobuf types. `Grpc.Tools` is the code generator that runs at build time, and marking it as `PrivateAssets="All"` means it won’t leak into consumers of this project. `Grpc.Core.Api` provides the shared gRPC abstractions used by both clients and servers. Together, these packages are the foundation that lets the `.proto` file drive everything else.

Now let’s look at the actual proto file. This lives at `Airline.FlightDirectory.Contracts/flight_directory.proto`.

```proto
syntax = "proto3";

option csharp_namespace = "Airline.FlightDirectory.Contracts";

package flightdirectory;

// ===== Messages =====

message AirportDto {
  string code = 1;      // YUL
  string name = 2;      // Montréal–Trudeau
  string city = 3;      // Montreal
}

message AircraftDto {
  string model = 1;     // Dash 8-400
  int32 seatCapacity = 2;
}

message FlightDto {
  string flightNumber = 1;   // AC101
  AirportDto origin = 2;
  AirportDto destination = 3;
  string departureUtc = 4;
  string arrivalUtc = 5;
  AircraftDto aircraft = 6;
}

// ===== Requests =====

message GetFlightRequest {
  string flightNumber = 1;
}

message SearchFlightsRequest {
  string originCode = 1;
  string destinationCode = 2;
  string departureDate = 3; // YYYY-MM-DD
}

message SearchFlightsResponse {
  repeated FlightDto flights = 1;
}

// ===== Service =====

service FlightDirectoryService {
  rpc GetFlight (GetFlightRequest) returns (FlightDto);
  rpc SearchFlights (SearchFlightsRequest) returns (SearchFlightsResponse);
}
```

Let’s walk through this slowly, because this file is the heart of the entire service.

At the very top, `syntax = "proto3"` declares the protobuf version we’re using. Proto3 is the modern standard and what you’ll use in almost all new projects. The `option csharp_namespace` tells the code generator which .NET namespace to use for the generated classes, so everything lines up cleanly in C#. The `package` statement is a logical grouping inside protobuf itself and helps avoid naming collisions across services.

Next, we define messages. A `message` is basically the protobuf equivalent of a C# class or record. `AirportDto`, `AircraftDto`, and `FlightDto` are the data transfer shapes that will go over the wire. Notice how they mirror our domain concepts, but they’re not our domain models. They’re DTOs designed for communication.

Each field has a number, like `= 1`, `= 2`, and so on. These numbers are extremely important. They’re how protobuf identifies fields in binary form, and once a field number is published, it must never change. You can add new fields later, but you don’t reuse or renumber existing ones. This is how protobuf maintains backward compatibility.

You’ll also notice nesting. `FlightDto` contains `AirportDto` and `AircraftDto`. This shows how protobuf naturally supports rich object graphs. When we later map domain models to these DTOs, the structure lines up very naturally.

Then we define request and response messages. `GetFlightRequest` contains just a flight number. `SearchFlightsRequest` includes origin and destination airport codes and a departure date in a simple string format. `SearchFlightsResponse` uses the `repeated` keyword, which simply means “this is a collection.” In C#, this will map to something like a list of `FlightDto`.

Finally, we define the service itself. The `service FlightDirectoryService` block declares the gRPC service and its available remote methods. Each `rpc` line defines a method name, an input message, and an output message. `GetFlight` takes a `GetFlightRequest` and returns a single `FlightDto`. `SearchFlights` takes a `SearchFlightsRequest` and returns a `SearchFlightsResponse`. This is the contract. Nothing more and nothing less.

To make sure the tooling actually picks up this proto file, we also need to configure the Contracts project `.csproj` file like this:

```
<ItemGroup>
  <Protobuf Include="flight_directory.proto" GrpcServices="Both" />
</ItemGroup>
```

This tells the build system to process this proto file and generate both server-side and client-side code. The important thing to understand is that this is not optional. Without this line, the `.proto` file just sits there and nothing happens.

Now here’s the part that often trips people up, so let’s make it crystal clear. When you build the solution, a `FlightDirectoryService` class is generated automatically. You do not write this class by hand. Ever. The gRPC toolchain reads your `.proto` file, generates C# classes and base service classes, and injects them into the compilation process automatically. You won’t see them as physical `.cs` files in your project, but they’re there, and you can inherit from them and use them like any other C# type.

This is a core gRPC concept. The `.proto` file is the source of truth. The code is derived from it. Once that clicks, gRPC starts to feel a lot less mysterious and a lot more powerful. In the next step, we’ll take this generated service base class and actually implement the server-side logic in ASP.NET Core.

Cool, now we’re at the point where things start to feel like a real application, because we’re going to separate the pure business world from the gRPC world, and then glue them together in the API project.

First up is Step 2, the domain models. This is where we model the airline concepts using plain C# only, with no protobuf and no gRPC references anywhere. That separation is not just “nice to have.” It’s one of those things that keeps your codebase clean months later when the demo turns into a real service.

In the `Airline.FlightDirectory.Domain` project, we define the core entities as C# records.

```csharp
public record Airport(string Code, string Name, string City);

public record Aircraft(string Model, int SeatCapacity);

public record Flight(
    string FlightNumber,
    Airport Origin,
    Airport Destination,
    DateTime DepartureUtc,
    DateTime ArrivalUtc,
    Aircraft Aircraft);
```

What’s happening here is straightforward, but super important. `Airport` captures the idea of an airport with a code like YUL, plus a human-friendly name and a city. `Aircraft` captures the aircraft model and how many seats it has. And `Flight` ties it all together by describing one specific flight: its number, where it starts, where it lands, its schedule, and which aircraft is operating it.

The key point is this: these domain models don’t know anything about protobuf, they don’t know anything about gRPC, and they definitely don’t know anything about ASP.NET Core. They’re “pure C#.” That’s critical for maintainability, because it means your business logic stays stable even if you change the transport later. Today it’s gRPC, tomorrow you might expose the same domain through REST, GraphQL, messaging, or whatever else. The domain shouldn’t care.

Now we move to Step 3, building the ASP.NET Core gRPC server. This happens in the API project, because the API project is the transport layer. It’s the part of the system that speaks gRPC over the network.

To host gRPC in ASP.NET Core, we install the gRPC server package in the API project.

```bash
<PackageReference Include="Grpc.AspNetCore" Version="2.76.0" />
```

Even though it’s wrapped in a “bash” code block in the note, what this really represents is a package reference that belongs in the API project’s `.csproj`. This package is what allows ASP.NET Core to actually host gRPC endpoints. Without it, you can still write service classes, but nothing will be listening for gRPC calls.

And since the API project needs to both implement the service contract and work with domain models, it must reference both the Contracts and Domain projects.

```
<ItemGroup>
	<ProjectReference Include="..\Airline.FlightDirectory.Contracts\Airline.FlightDirectory.Contracts.csproj" />
	<ProjectReference Include="..\Airline.FlightDirectory.Domain\Airline.FlightDirectory.Domain.csproj" />
</ItemGroup>
```

This is one of those small lines that quietly makes everything possible. The Contracts project gives the API access to the generated protobuf types like `FlightDto`, `GetFlightRequest`, and the generated base class for the service. The Domain project gives the API access to the “real” models like `Flight`, `Airport`, and `Aircraft`. The API sits in the middle and translates between them.

Now we hit Step 4, where we actually implement the gRPC service. This is where the generated base class becomes real behavior.

In the API project, we create a file called `FlightDirectoryGrpcService.cs`. The idea is that this class is responsible for exposing gRPC endpoints, so it belongs in the transport layer. It’s not domain logic; it’s how the outside world talks to the domain.

Here’s the service implementation.

```csharp
using Airline.FlightDirectory.Api.Mappings;
using Airline.FlightDirectory.Contracts;
using Airline.FlightDirectory.Domain;
using Grpc.Core;
namespace Airline.FlightDirectory.Api.Services;

public class FlightDirectoryGrpcService
  : FlightDirectoryService.FlightDirectoryServiceBase
{
    private static readonly List<Flight> Flights = SeedFlights();

    public override Task<FlightDto> GetFlight(
        GetFlightRequest request,
        ServerCallContext context)
    {
        var flight = Flights
            .FirstOrDefault(f => f.FlightNumber == request.FlightNumber);

        if (flight is null)
        {
            throw new RpcException(
                new Status(StatusCode.NotFound,
                $"Flight {request.FlightNumber} not found"));
        }

        return Task.FromResult(flight.ToDto());
    }

    public override Task<SearchFlightsResponse> SearchFlights(
        SearchFlightsRequest request,
        ServerCallContext context)
    {
        var results = Flights
            .Where(f =>
                f.Origin.Code == request.OriginCode &&
                f.Destination.Code == request.DestinationCode &&
                f.DepartureUtc.Date ==
                    DateTime.Parse(request.DepartureDate).Date)
            .Select(f => f.ToDto());

        return Task.FromResult(new SearchFlightsResponse
        {
            Flights = { results }
        });
    }

    private static List<Flight> SeedFlights()
    {
        var yul = new Airport("YUL", "Montréal–Trudeau", "Montreal");
        var yyz = new Airport("YYZ", "Toronto Pearson", "Toronto");

        var q400 = new Aircraft("Dash 8-400", 78);

        return new List<Flight>
    {
        new Flight(
            "AC101",
            yul,
            yyz,
            new DateTime(2026, 2, 10, 14, 00, 00, DateTimeKind.Utc),
            new DateTime(2026, 2, 10, 15, 20, 00, DateTimeKind.Utc),
            q400),

        new Flight(
            "AC203",
            yul,
            yyz,
            new DateTime(2026, 2, 10, 18, 00, 00, DateTimeKind.Utc),
            new DateTime(2026, 2, 10, 19, 20, 00, DateTimeKind.Utc),
            q400)
    };
    }
}
```

Now let’s unpack what’s going on, in a way that makes gRPC feel familiar.

The first thing to notice is inheritance. `FlightDirectoryGrpcService` inherits from `FlightDirectoryService.FlightDirectoryServiceBase`. That base class is not something we wrote. It’s generated automatically from the `.proto` file. This is proto-first in action: you define the service contract in `.proto`, the tooling generates a C# base class, and you implement it by inheriting and overriding methods.

Inside the class, we have a static list of `Flight` domain objects called `Flights`, and it’s populated by a `SeedFlights()` method. This is just a quick in-memory data store so we can focus on gRPC behavior without bringing in a database yet. The important part is that the list contains domain models, not protobuf DTOs, because our “truth” is the domain.

Then we implement the first RPC method, `GetFlight`. In the proto file, `GetFlight` is defined as taking a `GetFlightRequest` and returning a `FlightDto`. That contract forces the shape of this method. So here, the method takes the request, reads `request.FlightNumber`, and searches the domain list for a matching flight.

If it can’t find the flight, we throw a `RpcException` with `StatusCode.NotFound`. This is a very big mindset shift if you’re used to controllers. In gRPC, you’re not returning `NotFound()` or setting HTTP status codes directly. You throw an `RpcException`, and the gRPC runtime converts that into a gRPC status code that the client understands. So it still communicates “404-ish” semantics, but in gRPC terms.

If the flight exists, we return it as a `FlightDto`. Notice we don’t manually build a `FlightDto` here. We call `flight.ToDto()`. That method comes from `Airline.FlightDirectory.Api.Mappings`, which is basically where we’ll keep translation logic between domain models and protobuf messages. This is what keeps the service clean. The service focuses on handling the RPC call, and the mapping layer focuses on converting shapes.

Next is `SearchFlights`. This method takes a `SearchFlightsRequest` and returns a `SearchFlightsResponse`. Again, the signature is driven by the contract. We filter the `Flights` list based on origin code, destination code, and departure date. One detail to point out is that the request uses strings for the date, so we parse it using `DateTime.Parse(request.DepartureDate)` and compare it to the domain flight’s `DepartureUtc.Date`. This is simple and good enough for a demo, and later we can tighten it up with stricter parsing or well-defined timestamp types.

Then we map the matching domain flights to DTOs using `.Select(f => f.ToDto())`. Finally, we build and return a `SearchFlightsResponse`, and we populate its `Flights` collection using `Flights = { results }`. That syntax is protobuf’s generated collection initializer pattern, and you’ll see it a lot. It’s basically saying “add all results into the repeated field.”

So stepping back, here’s the big picture. The class inherits from a generated base class, each RPC method becomes an override, `RpcException` is how you communicate errors instead of HTTP responses, and these methods behave like normal method calls rather than MVC controller actions. That’s why gRPC feels so clean for internal service-to-service communication. It’s not about endpoints and controllers. It’s about strongly typed method calls across the network.

---

Alright, now we’re getting to the part that makes the whole thing feel clean and “grown-up,” even though it’s a small demo. We’ve got domain models that are pure C#, we’ve got protobuf DTOs generated from the `.proto` file, and now we need a clear, explicit bridge between the two. That’s Step 5: mapping domain to proto.

The goal here is to keep responsibilities in the right place. The domain stays totally unaware of gRPC and protobuf, and the generated protobuf types stay isolated to the transport layer. The only thing that knows both worlds is the API project, because that’s the layer that has to translate between “internal model” and “network contract.”

So in the API project, we create this file: `Airline.FlightDirectory.Api/Mappings/FlightMappings.cs`, and we put the mapping code inside it.

```csharp
using Airline.FlightDirectory.Contracts;
using Airline.FlightDirectory.Domain;

namespace Airline.FlightDirectory.Api.Mappings
{
    public static class FlightMappings
    {
        public static FlightDto ToDto(this Flight flight)
        {
            return new FlightDto
            {
                FlightNumber = flight.FlightNumber,
                DepartureUtc = flight.DepartureUtc.ToString("O"),
                ArrivalUtc = flight.ArrivalUtc.ToString("O"),
                Origin = new AirportDto
                {
                    Code = flight.Origin.Code,
                    Name = flight.Origin.Name,
                    City = flight.Origin.City
                },
                Destination = new AirportDto
                {
                    Code = flight.Destination.Code,
                    Name = flight.Destination.Name,
                    City = flight.Destination.City
                },
                Aircraft = new AircraftDto
                {
                    Model = flight.Aircraft.Model,
                    SeatCapacity = flight.Aircraft.SeatCapacity
                }
            };
        }
    }
}
```

What we’re doing here is creating an extension method called `ToDto` on the `Flight` domain model. That’s why the method signature looks like `this Flight flight`. It means anywhere in the API project, if we have a `Flight`, we can just call `flight.ToDto()` like it’s part of the type, which makes the gRPC service implementation feel super natural and uncluttered.

Inside the mapping, we construct a new `FlightDto`, which is one of the generated protobuf types from the Contracts project. We copy over the obvious scalar fields like `FlightNumber`, and then for the timestamps we convert `DateTime` to string using `ToString("O")`. That `"O"` format is the round-trip ISO 8601 format, which is a nice choice for a demo because it preserves the exact value and is easy to read if you print it out on the client side.

Then we map nested objects explicitly. `Origin` and `Destination` become `AirportDto` objects, each filled with `Code`, `Name`, and `City`. And the `Aircraft` becomes an `AircraftDto` with `Model` and `SeatCapacity`. Nothing magical is happening, and that’s a good thing. It’s explicit, easy to debug, and easy to unit test later if you want.

And this is exactly why this mapping step matters. It keeps the domain clean, it keeps protobuf isolated to the transport layer, and it keeps the mapping logic explicit and testable instead of being scattered across the service methods.

Now that mapping exists, our gRPC service methods can stay focused on “handle request, find domain data, return response,” and they don’t need to worry about the low-level shape-building for DTOs.

Next, we wire up the gRPC server in the API project’s `Program.cs`. This is where ASP.NET Core actually starts hosting the service and listening for calls.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddGrpc();

var app = builder.Build();

app.MapGrpcService<FlightDirectoryGrpcService>();
app.MapGet("/", () => "gRPC Flight Directory is running");

app.Run();
```

Here’s what this does in plain English. `builder.Services.AddGrpc()` registers the gRPC services with the ASP.NET Core dependency injection container and sets up the gRPC pipeline. Without this line, your gRPC services won’t run, even if the class exists.

Then `app.MapGrpcService<FlightDirectoryGrpcService>()` is the line that actually exposes your `FlightDirectoryGrpcService` over gRPC. This is basically the equivalent of “hook this service up to incoming gRPC calls.” And `app.MapGet("/", ...)` is just a friendly little HTTP endpoint so that if you open the server URL in a browser, you get a simple message instead of confusion. It’s not your gRPC API, it’s just a quick “yes, the server is alive” check.

At this point, the server side is wired up. We have a contract in `.proto`, generated DTOs and base classes, a real implementation that returns domain data mapped to DTOs, and a host that exposes it.

Now we move to Step 6, the console client, and this is where a lot of beginners have that “wait… why can’t I just test it like REST?” moment.

So let’s address it head-on. You can’t test gRPC like a REST API using `.http` files, Postman, or curl in the usual way. If you try to do something like:

POST [https://localhost:5225/flights](https://localhost:5225/flights)
Content-Type: application/json

it just won’t work for gRPC. And it’s not because you’re doing something wrong. It’s because gRPC is fundamentally not a URL-based, JSON-over-HTTP/1.1 style API. gRPC payloads are binary, not JSON. The endpoints aren’t designed around resource URLs. And classic HTTP/1.1 clients don’t understand gRPC’s framing rules. gRPC relies on HTTP/2 and uses headers like `application/grpc`. So if you try to poke it the REST way, it’s like trying to play a Blu-ray disc in a cassette player. The formats just don’t match.

That’s why the simplest, most correct approach for this tutorial is to create a .NET console app and use an actual gRPC client.

In the console client project, you add the minimal required packages.

```
<PackageReference Include="Grpc.Net.Client" Version="2.76.0" />
<PackageReference Include="Google.Protobuf" Version="3.25.3" />
<PackageReference Include="Grpc.Tools" Version="2.76.0" PrivateAssets="All" />
```

Here’s the role each one plays. `Grpc.Net.Client` is the actual gRPC client library for .NET. It handles HTTP/2, creates the channel, and lets you call remote methods like they’re local. `Google.Protobuf` is needed for serialization and deserialization of messages, because everything is protobuf under the hood. And `Grpc.Tools` is the build-time code generator that reads the `.proto` file and generates the C# client types. Just like on the server side, we only need it during build, which is why `PrivateAssets="All"` is there so it doesn’t ship as a runtime dependency.

Then we add a reference to the Contracts project, because the console app needs the generated client type and the request and response message classes.

```
<ItemGroup>
  <ProjectReference Include="..\Airline.FlightDirectory.Contracts\Airline.FlightDirectory.Contracts.csproj" />
</ItemGroup>
```

Now we can write the client code in the console app’s `Program.cs`. This is the part that usually makes people smile, because it looks like calling a normal C# service, even though it’s happening over the network.

```
using Airline.FlightDirectory.Contracts;
using Grpc.Net.Client;

var channel = GrpcChannel.ForAddress("http://localhost:5225");
var client = new FlightDirectoryService.FlightDirectoryServiceClient(channel);

// Test: Get a single flight
var flight = await client.GetFlightAsync(new GetFlightRequest
{
    FlightNumber = "AC101"
});

Console.WriteLine($"{flight.FlightNumber} from {flight.Origin.Code} to {flight.Destination.Code}");

// Test: Search flights
var response = await client.SearchFlightsAsync(new SearchFlightsRequest
{
    OriginCode = "YUL",
    DestinationCode = "YYZ",
    DepartureDate = "2026-02-10"
});

foreach (var f in response.Flights)
{
    Console.WriteLine($"{f.FlightNumber} departs at {f.DepartureUtc}");
}
```

Let’s break this down in the simplest way. First, we create a `GrpcChannel` pointing at the server address. That channel is basically your connection to the gRPC server, and it uses HTTP/2 under the hood.

Then we create the client: `new FlightDirectoryService.FlightDirectoryServiceClient(channel)`. This client type is generated from your `.proto` file. You didn’t write it. The tooling created it for you, and it contains strongly-typed methods that match the service definition in protobuf.

Next, we test the unary call `GetFlightAsync`. We create a `GetFlightRequest` and set `FlightNumber` to `"AC101"`. When we call `await client.GetFlightAsync(...)`, we’re making a remote call that feels like a local method call, and we get back a `FlightDto` that contains nested DTOs for origin and destination. Then we print a friendly line like “AC101 from YUL to YYZ.” That’s our first proof that the pipeline works end to end: client → server → domain lookup → mapping → response.

After that, we test `SearchFlightsAsync`. We send origin code, destination code, and departure date. The server filters the in-memory list, maps each matching domain flight to a `FlightDto`, and returns a `SearchFlightsResponse` with a repeated list of flights. On the client side, we loop through `response.Flights` and print each flight number and its departure time, which is exactly why we formatted times using `"O"` earlier, because now the client can print them directly without losing precision.

So at this stage, everything is connected. The `.proto` contract defines the service and messages, the server implements the generated base class and maps domain models to protobuf DTOs, and the client uses the generated client class to call the server like it’s just another C# service. That’s gRPC’s sweet spot right there: strong contracts, clean code, and fast internal communication without the usual REST overhead.

Alright, let’s wrap this episode up with something practical and a little bit fun. At this point, we have a gRPC server, we have a console client, and they work. But running them manually in two terminals every time gets old fast, especially when you’re demoing or recording videos. So let’s automate the whole thing with a small PowerShell script that runs the API and the client side by side.

The idea is simple. We assume the solution root contains two projects: `Airline.FlightDirectory.Api` for the gRPC server and `Airline.FlightDirectory.Client` for the console client. The script will start the API in a new PowerShell window, wait until it’s actually up and responding, and then run the client in the current window.

We’ll create a file called `run-grpc-demo.ps1` directly in the solution root. This way, you can just open PowerShell in the solution folder and run one command to see the whole flow end to end.

Here’s the full script.

```powershell
param(
  [string]$ApiProject    = ".\Airline.FlightDirectory.Api\Airline.FlightDirectory.Api.csproj",
  [string]$ClientProject = ".\Airline.FlightDirectory.Client\Airline.FlightDirectory.Client.csproj",
  [string]$ApiUrl        = "https://localhost:7005",
  [int]$TimeoutSeconds   = 30
)

$ErrorActionPreference = "Stop"

function Wait-ForHttp {
  param(
    [Parameter(Mandatory=$true)][string]$Url,
    [int]$TimeoutSeconds = 30
  )

  $deadline = (Get-Date).AddSeconds($TimeoutSeconds)

  while ((Get-Date) -lt $deadline) {
    try {
      # gRPC itself isn't "HTTP GET friendly", but your API mapped "/" => message
      $resp = Invoke-WebRequest -Uri $Url -UseBasicParsing -Method GET -TimeoutSec 2
      if ($resp.StatusCode -ge 200 -and $resp.StatusCode -lt 500) {
        return $true
      }
    } catch {
      Start-Sleep -Milliseconds 500
    }
  }

  return $false
}

Write-Host "Starting API in a new PowerShell window..." -ForegroundColor Cyan

# Start API in a separate window so it keeps running
$apiCmd = "dotnet run --project `"$ApiProject`" --launch-profile https"
Start-Process powershell -ArgumentList "-NoExit", "-Command", $apiCmd | Out-Null

Write-Host "Waiting for API to respond at $ApiUrl ..." -ForegroundColor Yellow
if (-not (Wait-ForHttp -Url $ApiUrl -TimeoutSeconds $TimeoutSeconds)) {
  throw "API did not become ready at $ApiUrl within $TimeoutSeconds seconds. Check API logs window."
}

Write-Host "API is up. Running client..." -ForegroundColor Green
dotnet run --project "$ClientProject"

Write-Host "`nDone. API is still running in the other window. Close that window when finished." -ForegroundColor Cyan
```

Let’s walk through this in plain language so it doesn’t feel like magic.

At the top, we define a few parameters. `ApiProject` and `ClientProject` point to the `.csproj` files for the API and the client. `ApiUrl` is the HTTPS address where the API will be listening, and `TimeoutSeconds` controls how long we’re willing to wait for the API to start before giving up. The nice thing about doing it this way is that you can override these values later without changing the script itself.

Next, we set `$ErrorActionPreference = "Stop"`. This tells PowerShell to stop immediately if something goes wrong instead of silently continuing. For demos and automation scripts, this is exactly what you want, because failures should be loud and obvious.

Then we define a helper function called `Wait-ForHttp`. Even though gRPC itself isn’t friendly to normal HTTP GET requests, remember that in our API we mapped the root path `/` to a simple message. That’s our little health check. This function keeps trying to send an HTTP GET request to the API URL until it gets a response or until the timeout expires. It sleeps briefly between attempts, so it doesn’t hammer the server while it’s starting up.

After that, we print a message saying we’re starting the API in a new PowerShell window. This is important, because we want the API to keep running independently while the client executes.

The line that does the heavy lifting is the `Start-Process` call. It opens a new PowerShell window, runs `dotnet run` on the API project using the HTTPS launch profile, and keeps that window open. This way, you can see the API logs live while the client runs in parallel.

Once the API process is started, we don’t immediately run the client. Instead, we wait. The script calls `Wait-ForHttp` with the API URL and the timeout. If the API doesn’t become ready in time, the script throws an error with a clear message telling you to check the API logs. This is much better than just running the client and watching it fail mysteriously.

When the API is confirmed to be up, we print a green message saying it’s ready, and then we run the client using `dotnet run` on the client project. At this point, you’ll see the console output from the client showing the results of `GetFlight` and `SearchFlights`, just like we did earlier manually.

Finally, the script prints a friendly message reminding you that the API is still running in the other PowerShell window, and that you can close that window when you’re done. That’s it. One script, one command, full demo.

Before you run this script for the first time, there’s one important one-time setup step. Because we’re using HTTPS, you need to trust the ASP.NET Core development certificate. In PowerShell, run this command once:

```powershell
dotnet dev-certs https --trust
```

After that, you’re good to go. If your API uses a different HTTPS port than `7005`, just update the `ApiUrl` parameter in the script to match your setup.

To run the demo, navigate to the solution folder in PowerShell and execute:

```powershell
.\run-grpc-demo.ps1
```

You’ll see a new PowerShell window pop up for the API, you’ll see the client output in the original window, and you can watch the logs on both sides to understand exactly what’s happening.

And that brings us to the end of this episode. If you enjoyed this walkthrough and it helped gRPC finally click for you, go ahead and like the video, it really helps the channel. Don’t forget to subscribe to *A Coder’s Journey* so you don’t miss the next episodes in this gRPC tutorial series. In the upcoming videos, we’ll build on this foundation and go deeper into real-world patterns. And if you want to play with the code yourself, grab the full solution from the link in the video description.



