Create a branch called `episode-2` based on the `episode-1` branch.

Alright, let’s shift gears a bit and zoom in on Episode 2 from a different angle. This time, we’re not talking about flight status or real-time updates. We’re talking about bookings, and more importantly, we’re talking about contracts and versioning in gRPC, which is one of those topics that looks boring at first… right up until it saves you from breaking production clients.

So the title of this episode is “Contracts & Versioning — Booking API as a Stable Contract,” and the goal is very clear. We want to design a BookingService contract, version one, in a way that we can evolve safely over time without breaking existing clients.

Before we write any proto, we need the right mental model.

The most important mindset shift with gRPC is this: your `.proto` file is your public API surface. Not your controller. Not your C# service class. The `.proto` file.

A gRPC service is basically two things combined. First, a public contract defined in `.proto`. Second, a compiled client SDK that’s generated from that contract, like C# classes, methods, and DTOs. When someone consumes your gRPC service, they’re not just calling an endpoint. They’re compiling against your contract.

That’s why the `.proto` file behaves more like a shared interface than a controller. Once you publish it and clients generate code from it, changes have real consequences.

This leads directly to the core rules.

The first rule is that field numbers are forever. When you assign a field number in protobuf, you’re not just labeling a property. You’re defining how that data is encoded on the wire. If you change that number later, old clients will interpret new data incorrectly, and new clients will misread old data. Best case, fields silently break. Worst case, data gets mixed up in dangerous ways.

The second rule is to be very conservative about removing or renaming things. Removing a field from the proto doesn’t magically remove it from older clients. Those clients still think it exists. Renaming a field might look harmless in code, but on the wire, the field number still matters. If you’re not careful, you end up with two sides that think they’re talking about the same thing but actually aren’t.

The third rule is that you should add new fields without changing the meaning of old ones. That means if a field used to represent “total price in cents,” you don’t later decide it means “price in dollars.” Old clients will keep sending cents, new clients will expect dollars, and now you’ve got a silent data corruption bug. Those are the worst.

Now that the mental model is clear, let’s talk about the domain slice: Booking.

In this episode, the booking domain includes a few core entities. We have a Passenger, which represents who is traveling. We have a Booking, which is the reservation itself. We have ItinerarySegment, which represents each flight leg, because a booking can include multiple flights. And we have Fare, which captures pricing, currency, and breakdown.

From that domain, we get three core use cases. We want to create a booking. We want to retrieve an existing booking. And we want to do a price check, which is essentially a quote without committing anything. This is a very realistic airline API shape.

Now let’s get into the proto design guidelines. These are not abstract “best practices.” These are the rules you actually use if you don’t want to regret your API six months from now.

First, field numbers.

Once a field number is published, you never change it. Ever. You never reuse it either, even if you delete the field from the proto file. The reason is that protobuf encoding relies on those numbers. Old messages might still contain that field number, and if you reuse it for something else, the decoder will happily map old data into a completely different field.

That’s why protobuf gives you `reserved`.

```proto
reserved 7, 8;
reserved "oldFieldName";
```

What this is saying is: “These field numbers and names are permanently off-limits.” You’re telling future you, and future teammates, that these slots are intentionally burned. This prevents accidental reuse and protects backward compatibility.

A simple example helps here. Imagine you had `string promoCode = 7;` in V1. Later you decide to remove promo codes. If you just delete it and later add `bool isCorporate = 7;`, old messages that still contain field 7 will suddenly be interpreted as `isCorporate`. That’s not a compile error. That’s a silent logic bug. Reserving field numbers avoids this entirely.

Next, collections.

Whenever you have a list of things, you use `repeated`.

```proto
repeated ItinerarySegmentDto segments = 3;
```

This might look obvious, but it’s important to understand the contract implication. A repeated field is explicitly saying, “There can be zero, one, or many of these.” If you instead model a single field and later realize you need multiple, you’re stuck. Changing a singular field to a collection is a breaking change for clients. Starting with `repeated` gives you room to grow.

In the booking world, itinerary segments are the perfect example. Even if today you only support one flight per booking, tomorrow you’ll support connections. If you start with `repeated`, you don’t need a V2 just to support multi-leg trips.

Now let’s talk about timestamps and money, because this is where a lot of APIs quietly get messy.

For timestamps, protobuf gives you `google.protobuf.Timestamp`. This is almost always better than using strings like `"2026-02-10T14:00:00Z"`. With timestamps, you get a well-defined structure, timezone clarity, and generated code that maps cleanly to native time types. Strings are flexible, but they’re also easy to misuse, parse incorrectly, or format inconsistently.

For money, you don’t need to over-engineer on day one, but you do need to be deliberate. A simple and stable approach is `string currencyCode` plus `int64 amountCents`. This avoids floating-point issues, makes rounding behavior explicit, and works well across languages. If you later need something more sophisticated, you can evolve it, but this baseline is solid and predictable.

If you ignore these rules and use floating-point numbers for prices or free-form strings for dates, you’ll eventually run into rounding bugs, parsing issues, or subtle differences between clients written in different languages.

Finally, let’s talk about optional fields and the idea of “missing versus default.”

In proto3, every field has a default value. For strings, that’s an empty string. For numbers, that’s zero. The tricky part is that you can’t tell the difference between “the client didn’t set this” and “the client explicitly set it to the default” unless you use `optional` or wrapper types.

Modern proto3 supports `optional`, so you can write something like `optional string middleName = 3;`. This allows the server to know whether the field was actually provided. Wrapper types like `google.protobuf.StringValue` do the same thing, just in a more explicit way.

The practical rule here is simple: use `optional` when “missing” is meaningful. A middle name is a perfect example. An empty string and “not provided” are not the same thing. Another example is an optional loyalty number or optional promo code. If you don’t model that distinction correctly, you’ll eventually write weird conditional logic to guess intent.

If you ignore this and rely on defaults, you’ll end up in situations where you can’t tell whether a client meant “no value” or “default value,” and that ambiguity leaks into business logic in very unpleasant ways.

So the big takeaway is that proto design is not about syntax. It’s about commitment. Once you publish a `.proto`, you’re making a promise to clients. Field numbers, meanings, and shapes matter. If you design with evolution in mind from day one, you can move fast without breaking people. If you don’t, you’ll end up versioning everything in panic mode later.

And that’s exactly why contracts and versioning deserve their own episode.

---

Alright, now we’re going to make Episode 2 very concrete by actually introducing a brand-new contracts project for Booking, defining a stable V1 `.proto`, and then backing it with pure C# domain models. After that, we’ll talk about validation, because this is where a lot of people accidentally misuse proto and end up baking business rules into the contract.

So let’s start with the Contracts project.

You already have `Airline.FlightDirectory.Contracts`, which is great. But for Episode 2, we’re intentionally treating Booking as its own contract surface, so we create a new class library called `Airline.Booking.Contracts`.

In that project’s `.csproj`, we add the protobuf and gRPC tooling plus the `Protobuf` item so `booking.proto` gets compiled and generates code.

```xml
	<ItemGroup>
		<PackageReference Include="Google.Protobuf" Version="3.25.3" />
		<PackageReference Include="Grpc.Tools" Version="2.76.0" PrivateAssets="All" />
		<PackageReference Include="Grpc.Core.Api" Version="2.76.0" />
	</ItemGroup>
	<ItemGroup>
		<Protobuf Include="booking.proto" GrpcServices="Both" />
	</ItemGroup>
```

The important “tiny detail that matters” here is the proto filename. We’re explicitly calling it `booking.proto`, and we’re telling MSBuild to generate both server and client code from it with `GrpcServices="Both"`. This is what makes the `.proto` become real C# classes and a real generated client later.

Now we write the actual V1 contract file.

Create `Airline.Booking.Contracts/booking.proto` with the following content.

```proto
syntax = "proto3";

option csharp_namespace = "Airline.Booking.Contracts";

package booking.v1;

import "google/protobuf/timestamp.proto";

// ===== Common DTOs =====

message PassengerDto {
  string firstName = 1;
  string lastName = 2;
  string email = 3;
}

message ItinerarySegmentDto {
  string flightNumber = 1;
  string originCode = 2;
  string destinationCode = 3;
  google.protobuf.Timestamp departureUtc = 4;
  google.protobuf.Timestamp arrivalUtc = 5;
}

message FareDto {
  string currencyCode = 1;     // "CAD"
  int64 baseFareCents = 2;     // 19900
  int64 taxesCents = 3;        // 4500
  int64 totalCents = 4;        // 24400
}

// ===== Requests / Responses =====

message PriceCheckRequest {
  PassengerDto passenger = 1;
  repeated ItinerarySegmentDto segments = 2;
}

message PriceCheckResponse {
  FareDto fare = 1;
}

message CreateBookingRequest {
  PassengerDto passenger = 1;
  repeated ItinerarySegmentDto segments = 2;
}

message CreateBookingResponse {
  string bookingId = 1;
  FareDto fare = 2;
}

message GetBookingRequest {
  string bookingId = 1;
}

message BookingDto {
  string bookingId = 1;
  PassengerDto passenger = 2;
  repeated ItinerarySegmentDto segments = 3;
  FareDto fare = 4;
  google.protobuf.Timestamp createdUtc = 5;
}

service BookingService {
  rpc PriceCheck (PriceCheckRequest) returns (PriceCheckResponse);
  rpc CreateBooking (CreateBookingRequest) returns (CreateBookingResponse);
  rpc GetBooking (GetBookingRequest) returns (BookingDto);
}
```

Let’s unpack what this file is saying, in human terms, because this is your public API surface for Booking.

At the top, `syntax = "proto3"` declares we’re using proto3. Then `option csharp_namespace = "Airline.Booking.Contracts"` controls where the generated C# types land, so they don’t end up in some awkward namespace. Then `package booking.v1;` is a big deal: it’s your versioning boundary. You’re effectively saying, “This is Booking API version one.” Later, if you truly need breaking changes, you can introduce `booking.v2` cleanly.

Then we import `google/protobuf/timestamp.proto`. That gives us `google.protobuf.Timestamp`, which is protobuf’s standard way to represent time in a strongly typed, language-neutral way.

Next we define the common DTOs. `PassengerDto` is the passenger’s core identity for a booking: first name, last name, email. `ItinerarySegmentDto` represents one leg of the itinerary, like YYZ to YVR, with a flight number and departure and arrival timestamps. It’s marked as a message because it’s a reusable structure that shows up in multiple requests and responses. `FareDto` represents pricing in a safe way: a currency code like CAD, and amounts expressed in cents as integers, which avoids floating-point rounding issues.

Then we define request and response shapes. A `PriceCheckRequest` contains a passenger and a list of itinerary segments, and the response gives you a `FareDto`. This is the “quote without committing” use case. A `CreateBookingRequest` looks very similar, because creating a booking needs the same inputs, and its response gives you a booking ID plus the final fare used. Finally, `GetBookingRequest` takes just a booking ID and returns a `BookingDto`, which is the full read model: passenger, segments, fare, and when it was created.

And at the bottom, the service ties it together. `BookingService` exposes three unary RPCs: `PriceCheck`, `CreateBooking`, and `GetBooking`. Each one is strongly typed, and the contract is very explicit about the input and output message types.

After this, the next step is to build `Airline.Booking.Contracts`. When you build it, the tooling reads `booking.proto` and generates all the C# message types and the base service/client types. Then we add a reference to this contracts project from `Airline.FlightDirectory.Api` so that the API layer can reuse generated DTOs or call the Booking service later depending on how you structure the demo.

Now, why do we call this V1 “stable”?

One reason is that all request and response models are explicit. That means clients don’t have to guess what to send or what they’ll receive. The shapes are clear, and the compiler enforces them. If you later want to add something like a loyalty number, you can add a new field, and older clients won’t break because they simply ignore fields they don’t know.

Another reason is that we use `Timestamp`, which is strongly typed and locale-safe. That means we’re not sending dates as strings that can be formatted differently depending on culture or parsing assumptions. A timestamp is a timestamp, whether your client is in .NET, Java, Go, or Python. It prevents that classic mess where one client sends `"01/02/2026"` and someone argues whether it’s January 2 or February 1.

Another reason is money in cents. Using integers for money avoids floating errors. With floating-point, you eventually get rounding surprises like `19.9 + 0.1` not being exactly `20.0` in binary representation. Money should be boring and predictable, and integers give you that. You can still format it nicely on the UI, but the contract stays stable and safe.

And finally, we return `BookingDto` for `GetBooking`, which is a good read model. Instead of forcing the client to call multiple endpoints to reconstruct a booking, we give them one “this is the booking” DTO that includes everything needed to display or process it. That makes the contract pleasant to use, and it keeps the client logic simpler.

Now we move to the domain models for Booking, and we keep them pure C# just like we did in Flight Directory. The domain doesn’t know protobuf exists. The domain doesn’t care about gRPC. It just models the business concepts.

In `Airline.Booking.Domain`, add the following.

```csharp
namespace Airline.Booking.Domain;

public record Passenger(string FirstName, string LastName, string Email);

public record ItinerarySegment(
    string FlightNumber,
    string OriginCode,
    string DestinationCode,
    DateTime DepartureUtc,
    DateTime ArrivalUtc);

public record Fare(string CurrencyCode, long BaseFareCents, long TaxesCents, long TotalCents);

public record Booking(
    string BookingId,
    Passenger Passenger,
    IReadOnlyList<ItinerarySegment> Segments,
    Fare Fare,
    DateTime CreatedUtc);
```

This mirrors the contract conceptually, but it uses domain-friendly types. It uses `DateTime` internally. It uses `IReadOnlyList<ItinerarySegment>` to represent a stable list of segments. And it models money as integer cents just like the contract does, which makes mapping straightforward. The key point is that these are domain models. They can evolve for internal needs, and you can map them to contract DTOs in the transport layer, just like we did with `FlightMappings`.

Now let’s talk about validation strategies, because this is where people sometimes get confused and either do too little or too much.

There are two different layers of responsibility: the proto contract and the server.

The proto contract is responsible for the shape of the data. It defines types, nested messages, collections, and what fields exist. It’s like saying, “A booking has a passenger and a list of segments.” It can’t really enforce “required” rules the same way C# can, because proto3 doesn’t enforce required fields like proto2 used to. You can document intent with comments, but you don’t rely on the contract alone to enforce business validity.

The server is responsible for rules. And rules are things that can change over time. Email format rules might evolve. Allowed airport codes might be tied to your airline’s network and change seasonally. Pricing rules absolutely change. Min and max segments can change. Seat availability constraints depend on live inventory and can change minute by minute. You don’t want to bake those into the contract because contracts are hard to change safely.

So the practical pattern is: validate on the server, and use gRPC status codes to communicate the type of failure.

If the client sends bad input, like missing required data, you return `InvalidArgument`. For example, a passenger email is required, but the client didn’t provide it.

```csharp
throw new RpcException(new Status(StatusCode.InvalidArgument, "Passenger email is required"));
```

If the client asks for a booking ID that doesn’t exist, you return `NotFound`. That’s a clear “this resource doesn’t exist” signal, and clients can handle it cleanly.

If the input shape is fine but the business rule fails, like “you can’t book this itinerary because the fare expired” or “no seats available,” then `FailedPrecondition` is a good fit. It basically means “your request is valid, but the system is not in a state where it can be performed.”

So in plain terms, the contract defines what data looks like, and the server decides whether that data is acceptable and feasible. That separation keeps your proto stable and your business logic flexible, which is exactly what you want if you’re designing a BookingService V1 that can evolve safely.

---

# 8) Server: BookingGrpcService (Unary)

Add the following service implementation to `Airline.Booking.Api/Services/BookingGrpcService.cs`

```csharp
using Airline.Booking.Contracts;
using Airline.FlightDirectory.Api.Mappings;
using Airline.FlightDirectory.Domain;
using Grpc.Core;
namespace Airline.FlightDirectory.Api.Services;

public class BookingGrpcService : BookingService.BookingServiceBase
{
    private static readonly Dictionary<string, Domain.Booking> Store = new();

    public override Task<PriceCheckResponse> PriceCheck(
        PriceCheckRequest request,
        ServerCallContext context)
    {
        ValidatePassenger(request.Passenger);
        ValidateSegments(request.Segments);

        var fare = CalculateFare(request.Segments.Count);

        return Task.FromResult(new PriceCheckResponse
        {
            Fare = fare.ToDto()
        });
    }

    public override Task<CreateBookingResponse> CreateBooking(
        CreateBookingRequest request,
        ServerCallContext context)
    {
        ValidatePassenger(request.Passenger);
        ValidateSegments(request.Segments);

        var fare = CalculateFare(request.Segments.Count);

        var bookingId = $"BK-{Guid.NewGuid():N}".ToUpperInvariant();
        var createdUtc = DateTime.UtcNow;

        var booking = new Domain.Booking(
            bookingId,
            request.Passenger.ToDomain(),
            request.Segments.Select(s => s.ToDomain()).ToList(),
            fare,
            createdUtc);

        Store[bookingId] = booking;

        return Task.FromResult(new CreateBookingResponse
        {
            BookingId = bookingId,
            Fare = fare.ToDto()
        });
    }

    public override Task<BookingDto> GetBooking(
        GetBookingRequest request,
        ServerCallContext context)
    {
        if (string.IsNullOrWhiteSpace(request.BookingId))
            throw new RpcException(new Status(StatusCode.InvalidArgument, "bookingId is required"));

        if (!Store.TryGetValue(request.BookingId, out var booking))
            throw new RpcException(new Status(StatusCode.NotFound, $"Booking {request.BookingId} not found"));

        return Task.FromResult(booking.ToDto());
    }

    // ----- Validation -----

    private static void ValidatePassenger(PassengerDto passenger)
    {
        if (passenger is null)
            throw new RpcException(new Status(StatusCode.InvalidArgument, "passenger is required"));

        if (string.IsNullOrWhiteSpace(passenger.FirstName))
            throw new RpcException(new Status(StatusCode.InvalidArgument, "passenger.firstName is required"));

        if (string.IsNullOrWhiteSpace(passenger.LastName))
            throw new RpcException(new Status(StatusCode.InvalidArgument, "passenger.lastName is required"));

        if (string.IsNullOrWhiteSpace(passenger.Email))
            throw new RpcException(new Status(StatusCode.InvalidArgument, "passenger.email is required"));
    }

    private static void ValidateSegments(
        Google.Protobuf.Collections.RepeatedField<ItinerarySegmentDto> segments)
    {
        if (segments is null || segments.Count == 0)
            throw new RpcException(new Status(StatusCode.InvalidArgument, "At least one segment is required"));

        if (segments.Count > 6)
            throw new RpcException(new Status(StatusCode.InvalidArgument, "Too many segments (max 6)"));

        foreach (var s in segments)
        {
            if (string.IsNullOrWhiteSpace(s.FlightNumber))
                throw new RpcException(new Status(StatusCode.InvalidArgument, "segment.flightNumber is required"));
            if (string.IsNullOrWhiteSpace(s.OriginCode) || string.IsNullOrWhiteSpace(s.DestinationCode))
                throw new RpcException(new Status(StatusCode.InvalidArgument, "segment origin/destination required"));
        }
    }

    // ----- Pricing (dummy) -----
    private static Fare CalculateFare(int segmentCount)
    {
        var baseFare = 15000L * segmentCount; // $150/segment
        var taxes = (long)(baseFare * 0.20);  // 20%
        var total = baseFare + taxes;
        return new Fare("CAD", baseFare, taxes, total);
    }
}
```

---

# 9) Mapping: Domain ↔ Proto (clean and explicit)

Add the following mappings to `Airline.Booking.Api/Mappings/BookingMappings.cs`

```csharp
using Airline.Booking.Contracts;
using Airline.FlightDirectory.Domain;
using Google.Protobuf.WellKnownTypes;

namespace Airline.FlightDirectory.Api.Mappings;

public static class BookingMappings
{
    public static Passenger ToDomain(this PassengerDto dto)
        => new(dto.FirstName, dto.LastName, dto.Email);

    public static ItinerarySegment ToDomain(this ItinerarySegmentDto dto)
        => new(dto.FlightNumber, dto.OriginCode, dto.DestinationCode,
               dto.DepartureUtc.ToDateTime(), dto.ArrivalUtc.ToDateTime());

    public static FareDto ToDto(this Fare fare)
        => new()
        {
            CurrencyCode = fare.CurrencyCode,
            BaseFareCents = fare.BaseFareCents,
            TaxesCents = fare.TaxesCents,
            TotalCents = fare.TotalCents
        };

    public static BookingDto ToDto(this Domain.Booking booking)
        => new()
        {
            BookingId = booking.BookingId,
            Passenger = new PassengerDto
            {
                FirstName = booking.Passenger.FirstName,
                LastName = booking.Passenger.LastName,
                Email = booking.Passenger.Email
            },
            Fare = booking.Fare.ToDto(),
            CreatedUtc = Timestamp.FromDateTime(DateTime.SpecifyKind(booking.CreatedUtc, DateTimeKind.Utc)),
            Segments =
            {
                booking.Segments.Select(s => new ItinerarySegmentDto
                {
                    FlightNumber = s.FlightNumber,
                    OriginCode = s.OriginCode,
                    DestinationCode = s.DestinationCode,
                    DepartureUtc = Timestamp.FromDateTime(DateTime.SpecifyKind(s.DepartureUtc, DateTimeKind.Utc)),
                    ArrivalUtc = Timestamp.FromDateTime(DateTime.SpecifyKind(s.ArrivalUtc, DateTimeKind.Utc))
                })
            }
        };
}
```

---

Alright, now we’re at the part where the whole contract-first idea becomes super tangible: we’re going to write a real client that calls the BookingService end-to-end, in the exact flow a booking system typically follows.

And the flow is intentionally realistic. We do a price check first so the user gets a quote. Then we create the booking to actually commit it. And finally we fetch the booking back as a read model to prove it’s stored and retrievable.

So let’s start by creating a new console app called `Airline.BookingService.Client`. This will be the dedicated client for the Booking gRPC service.

In the client project `.csproj`, we add the gRPC client packages and reference the Booking Contracts project. This is important because the generated client and DTOs come from the `.proto` in `Airline.Booking.Contracts`, so the console app needs that project referenced.

Here’s what you add.

```xml
<ItemGroup>
	<PackageReference Include="Grpc.Net.Client" Version="2.76.0" />
	<PackageReference Include="Google.Protobuf" Version="3.25.3" />
	<PackageReference Include="Grpc.Tools" Version="2.76.0" PrivateAssets="All" />
</ItemGroup>
<ItemGroup>
  <ProjectReference Include="..\Airline.Booking.Contracts\Airline.Booking.Contracts.csproj" />
</ItemGroup>
```

What this setup gives you is the ability to create a `GrpcChannel`, use the generated `BookingServiceClient`, and work with real strongly typed protobuf DTOs like `PassengerDto`, `ItinerarySegmentDto`, and `PriceCheckRequest`. `Grpc.Tools` is there at build time because it can generate types when needed, and `PrivateAssets="All"` keeps it from becoming a runtime dependency.

Now we write `Program.cs`, and this is where it starts to feel almost suspiciously easy. You’re going to see remote calls that look like normal C# method calls, because gRPC generates a strongly typed client SDK for you.

Here’s the full code.

```csharp
using Airline.Booking.Contracts;
using Google.Protobuf.WellKnownTypes;
using Grpc.Net.Client;

using var channel = GrpcChannel.ForAddress("https://localhost:7005"); // or your Booking API URL
var client = new BookingService.BookingServiceClient(channel);

var passenger = new PassengerDto
{
    FirstName = "Morteza",
    LastName = "Giti",
    Email = "morteza@example.com"
};

var segment = new ItinerarySegmentDto
{
    FlightNumber = "AC101",
    OriginCode = "YUL",
    DestinationCode = "YYZ",
    DepartureUtc = Timestamp.FromDateTime(DateTime.SpecifyKind(new DateTime(2026, 2, 10, 14, 0, 0), DateTimeKind.Utc)),
    ArrivalUtc = Timestamp.FromDateTime(DateTime.SpecifyKind(new DateTime(2026, 2, 10, 15, 20, 0), DateTimeKind.Utc))
};

// 1) Price check
var quote = await client.PriceCheckAsync(new PriceCheckRequest
{
    Passenger = passenger,
    Segments = { segment }
});

Console.WriteLine($"Quote total (cents): {quote.Fare.TotalCents} {quote.Fare.CurrencyCode}");

// 2) Create booking
var created = await client.CreateBookingAsync(new CreateBookingRequest
{
    Passenger = passenger,
    Segments = { segment }
});

Console.WriteLine($"Created booking: {created.BookingId} total={created.Fare.TotalCents}");

// 3) Get booking
var booking = await client.GetBookingAsync(new GetBookingRequest
{
    BookingId = created.BookingId
});

Console.WriteLine($"Fetched booking {booking.BookingId}, segments={booking.Segments.Count}");
```

Let’s read this the way you’d explain it on camera.

We start by creating a gRPC channel with `GrpcChannel.ForAddress("https://localhost:7005")`. That URL is your Booking API base address, and because gRPC uses HTTP/2 under the hood, the channel is the thing that manages that connection properly.

Then we create a client: `new BookingService.BookingServiceClient(channel)`. That client is generated from `booking.proto`. You didn’t write it. The tooling did. And it contains real methods like `PriceCheckAsync`, `CreateBookingAsync`, and `GetBookingAsync` that match the service definition in your contract.

Next we build the input DTOs. We create a `PassengerDto` with first name, last name, and email. This is a protobuf DTO type generated from the proto message `PassengerDto`.

Then we create one itinerary segment using `ItinerarySegmentDto`. Here’s the important thing: notice we’re not using strings for times. We’re using protobuf timestamps. That’s why we import `Google.Protobuf.WellKnownTypes`, and we convert `DateTime` to `Timestamp` using `Timestamp.FromDateTime(...)`.

And there’s another subtle but crucial thing: we call `DateTime.SpecifyKind(..., DateTimeKind.Utc)` before converting. That’s because protobuf timestamps are UTC-based, and if you pass a `DateTime` without a UTC kind, you can end up with unexpected conversions or runtime complaints. By specifying UTC explicitly, you’re keeping the contract and the data consistent.

Now the flow begins.

First, we do a price check. We call `client.PriceCheckAsync` with a `PriceCheckRequest` containing the passenger and the segments list. Notice how protobuf repeated fields work nicely in C#: `Segments = { segment }` is the generated collection initializer syntax for repeated fields. The result is a `PriceCheckResponse` with a `Fare` inside it, and we print the total cents and currency code.

Second, we create the booking. We call `client.CreateBookingAsync` with a `CreateBookingRequest`, again sending passenger and segments. The server returns `CreateBookingResponse`, which contains the new `BookingId` plus the fare that was committed. We print that out, because this is the “booking was actually created” proof.

Third, we fetch the booking back. We call `client.GetBookingAsync` with the booking ID we just got. The server returns a `BookingDto`, which contains the passenger, segments, fare, and created timestamp. And we print the booking ID and the number of segments to confirm it came back as expected.

So with this tiny console app, you’ve proven that the contract is real and usable as a compiled SDK, not just documentation.

Now let’s talk about the V1 → V2 roadmap, because this is the entire point of Episode 2. We want to evolve without breaking clients.

The first safe evolution is adding a new optional field. For example, we decide to add `phoneNumber` to the passenger.

```proto
message PassengerDto {
  string firstName = 1;
  string lastName = 2;
  string email = 3;
  string phoneNumber = 4; // NEW (safe)
}
```

This is safe because old clients don’t know about field 4, so they ignore it when they receive it. They keep working. New clients can start sending it and receiving it immediately. Nothing breaks, because protobuf is designed to tolerate unknown fields.

The second safe evolution is adding a new RPC. For example, `CancelBooking`.

```proto
rpc CancelBooking (CancelBookingRequest) returns (CancelBookingResponse);
```

This is safe because adding a new method doesn’t change the behavior of existing methods. Old clients simply never call it. They’re unaffected. New clients can use it when ready.

The third safe evolution pattern is reserving removed fields if you must delete something. Maybe you used to have a discount code field in `FareDto` and you’re removing it.

```proto
message FareDto {
  reserved 5;
  reserved "oldDiscountCode";
  // ...
}
```

This is you protecting future you. You’re saying, “We used field number 5 and this name in the past. Do not reuse them.” That prevents accidental wire-level collisions with older messages.

Now the breaking changes, the ones you really want to avoid unless you’re doing a new package/version.

If you rename a field in a way that changes meaning, you’re basically making the contract lie. If you reuse a field number for a different purpose, you’re inviting silent data corruption. If you change a field type, like `int32` to `string`, you will break deserialization or force weird edge-case behavior. And if you move fields across messages without a migration plan, clients that expect the old shape will simply stop understanding what they receive.

So the mindset is: V1 is a promise. We evolve by adding, not by mutating. And if we truly must make a breaking change, we introduce a V2 package or a V2 message contract and support both during migration.

That’s how you move fast without breaking your consumers.

---

Alright, let’s wrap Episode 2 by talking about something that sounds abstract but is very practical once you’ve lived through it: how versioning actually works in the real world with gRPC.

Up to now, we’ve been careful, conservative, and additive. That’s not an accident. That’s exactly how most teams run gRPC services in production.

There are two main ways people do versioning in practice, and understanding when to use each one saves you a lot of pain later.

The first option, and by far the most common, is to keep the same service and evolve the contract carefully. In this model, your `booking.v1` package stays in place, and you only make safe, additive changes. You add new fields, you add new RPCs, you reserve removed fields, but you never change the meaning of existing fields. From the client’s point of view, nothing ever breaks. Old clients keep working. New clients slowly start using the new capabilities. This is boring, and boring is good when you’re running APIs that other teams depend on.

The second option is what you reach for when you truly must break things. Maybe the data model was wrong. Maybe the semantics changed so much that “adding fields” would just be lying. In that case, you introduce a parallel V2 package and service.

That looks like this.

```proto
package booking.v2;
service BookingServiceV2 { ... }
```

You don’t delete V1. You run V1 and V2 side by side during migration. Old clients keep calling V1. New clients move to V2 when they’re ready. Over time, V1 traffic drops to zero, and only then do you consider retiring it. This approach costs more upfront, but it’s the only safe way to handle real breaking changes in a distributed system.

Now that we’ve talked about versioning strategy, let’s actually wire things together so you can see both services running at the same time.

In your API’s `Program.cs`, we add the new Booking gRPC service alongside the existing Flight Directory service. We also update the root endpoint text so it reflects the fact that multiple gRPC services are now hosted in the same app.

Update the relevant part of `Program.cs` like this.

```csharp
app.MapGrpcService<FlightDirectoryGrpcService>();
app.MapGrpcService<BookingGrpcService>();
app.MapGet("/", () => "gRPC services are running");
```

What this means is that your ASP.NET Core app is now hosting two separate gRPC services over the same HTTP/2 endpoint. Clients don’t care. They just point at the same base address and call different generated clients. This is very common in internal systems where a single service host exposes multiple bounded contexts.

Now we update the PowerShell script so we can actually run everything and see it in action without juggling terminals manually.

The updated script does a few things in a very intentional order. It starts the API in its own PowerShell window using the HTTPS launch profile. It waits until the API responds to a simple GET request on the root path, which tells us the host is up and listening. Then it starts two clients in their own PowerShell windows: the Flight Directory client and the Booking Service client. And finally, it leaves all windows open so you can see logs and output side by side.

You can drop this script into your solution root and run it as-is.

```powershell
param(
  [string]$ApiProject     = ".\Airline.FlightDirectory.Api\Airline.FlightDirectory.Api.csproj",
  [string]$Client1Project = ".\Airline.FlightDirectory.Client\Airline.FlightDirectory.Client.csproj",
  [string]$Client2Project = ".\Airline.BookingService.Client\Airline.BookingService.Client.csproj",
  [string]$ApiUrl         = "https://localhost:7005",
  [int]$TimeoutSeconds    = 30
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

function Start-InNewWindow {
  param(
    [Parameter(Mandatory=$true)][string]$Title,
    [Parameter(Mandatory=$true)][string]$Command
  )

  # Set window title then run the command
  $wrapped = "`$Host.UI.RawUI.WindowTitle = '$Title'; $Command"
  Start-Process powershell -ArgumentList "-NoExit", "-Command", $wrapped | Out-Null
}

Write-Host "Starting API in a new PowerShell window (HTTPS profile)..." -ForegroundColor Cyan
Start-InNewWindow -Title "API - FlightDirectory (gRPC)" -Command "dotnet run --project `"$ApiProject`" --launch-profile https"

Write-Host "Waiting for API to respond at $ApiUrl ..." -ForegroundColor Yellow
if (-not (Wait-ForHttp -Url $ApiUrl -TimeoutSeconds $TimeoutSeconds)) {
  throw "API did not become ready at $ApiUrl within $TimeoutSeconds seconds. Check the API window logs."
}

Write-Host "API is up. Starting clients in separate windows..." -ForegroundColor Green

Start-InNewWindow -Title "Client 1 - FlightDirectory.Client" -Command "dotnet run --project `"$Client1Project`""
Start-InNewWindow -Title "Client 2 - BookingService.Client"   -Command "dotnet run --project `"$Client2Project`""

Write-Host "`nAll processes started in separate windows." -ForegroundColor Cyan
Write-Host "Close each window when you're done." -ForegroundColor Cyan
```

When you run this script, you’ll see three windows open. One for the API, one for the Flight Directory client, and one for the Booking Service client. You can watch the API logs while both clients make calls, which really drives home the idea that gRPC services are just strongly typed endpoints living side by side on the same host.

And that brings Episode 2 to a clean close. If this episode helped you understand why contracts and versioning matter in gRPC, go ahead and like the video—it really helps the channel. Don’t forget to subscribe to *A Coder’s Journey* so you don’t miss the next episodes, where we’ll keep building on this foundation. And if you want to explore the code yourself, grab the full solution from the link in the video description.



