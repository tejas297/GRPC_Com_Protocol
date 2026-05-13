# gRPC Unary Stock Trading Project

This project demonstrates a **unary gRPC** implementation using **Spring Boot** with two separate applications:

- **`stock-trading-service`** — the gRPC server that fetches stock data from a MySQL database
- **`stock-trading-client`** — the gRPC client that calls the server and prints the response

---

## 1. Project Overview

The application exposes one unary RPC:

- **`getStockPrice(StockRequest) returns (StockResponse)`**

The flow is:

1. The client sends a stock symbol such as `AAPL`
2. The server receives the request
3. The server looks up the stock in the database
4. The server returns the stock symbol, current price, and last updated timestamp
5. The client prints the response to the console

---

## 2. Modules

### `stock-trading-service`
This is the gRPC server module.

Key responsibilities:
- Hosts the gRPC service implementation
- Connects to MySQL using Spring Data JPA
- Reads stock data from the `stocks` table
- Returns a `StockResponse` for each request

### `stock-trading-client`
This is the gRPC client module.

Key responsibilities:
- Creates a blocking gRPC stub
- Sends a `StockRequest`
- Receives and prints the `StockResponse`

---

## 3. Technology Stack

Based on the current source files:

- **Java 21**
- **Spring Boot 4.0.3**
- **gRPC**
- **Protocol Buffers**
- **Spring gRPC**
- **Spring Data JPA**
- **MySQL**

---

## 4. gRPC Contract

The protocol definition is located in:

- `stock-trading-service/src/main/proto/stock_trading.proto`
- `stock-trading-client/src/main/proto/stock_trading.proto`

### Proto file
```proto
syntax = "proto3";

package stocktrading;
option java_multiple_files = true;
option java_package = "com.javatechie.grpc";
option java_outer_classname = "StockTradingProto";

service StockTradingService {
  // UNARY - RPC -> get current stock price
  rpc getStockPrice(StockRequest) returns (StockResponse);
}

message StockRequest {
  string stock_symbol = 1;
}

message StockResponse {
  string stock_symbol = 1;
  double price = 2;
  string timestamp = 3;
}
```

---

## 5. Internal Method Call Flow

This section traces the internal method calls at both client and server levels during a unary RPC.

### 5.1 Client-Side Method Call Chain

When the client application starts, the following method calls occur in sequence:

#### 1. Application Startup
```
StockTradingClientApplication.main(String[] args)
  └─> SpringApplication.run(StockTradingClientApplication.class, args)
      └─> StockTradingClientApplication bean created
          └─> StockClientService injected via constructor
```

#### 2. CommandLineRunner Execution
```
StockTradingClientApplication.run(String... args)
  └─> System.out.println("Grpc client response : " + ...)
      └─> StockClientService.getStockPrice("AAPL")
```

#### 3. StockClientService.getStockPrice() Internal Steps
```
StockClientService.getStockPrice(String stockSymbol)
  ├─> StockRequest.newBuilder()
  │   └─> returns RequestBuilder instance
  │
  ├─> StockRequest.Builder.setStockSymbol(stockSymbol)
  │   └─> sets stock_symbol field to "AAPL"
  │
  ├─> StockRequest.Builder.build()
  │   └─> returns StockRequest object (ready to send)
  │
  └─> StockTradingServiceGrpc.StockTradingServiceBlockingStub
      .getStockPrice(StockRequest request)
          └─> (gRPC transport layer)
              └─> sends serialized request to server:9090
              └─> blocks waiting for response
              └─> receives serialized response
              └─> deserializes to StockResponse
              └─> returns StockResponse to caller
```

#### 4. Return and Print
```
StockResponse received
  └─> Printed via System.out.println() in main method
      └─> Output example: stock_symbol: "AAPL"
                        price: 189.45
                        timestamp: "2026-05-08T10:30:00"
```

---

### 5.2 Server-Side Method Call Chain

When the server receives a gRPC request, the following method calls occur in sequence:

#### 1. Application Startup
```
StockTradingServiceApplication.main(String[] args)
  └─> SpringApplication.run(StockTradingServiceApplication.class, args)
      └─> gRPC server initialized on port 9090
      └─> StockTradingServiceImpl bean created
          └─> StockRepository injected via constructor
              └─> JPA data source configured (MySQL connection)
```

#### 2. gRPC Request Reception
```
grpc.server:9090
  └─> receives incoming request (serialized StockRequest)
      └─> deserializes to StockRequest object
      └─> routes to StockTradingService implementation
          └─> Calls StockTradingServiceImpl.getStockPrice()
```

#### 3. StockTradingServiceImpl.getStockPrice() Internal Steps
```
StockTradingServiceImpl.getStockPrice(
    StockRequest request,
    StreamObserver<StockResponse> responseObserver)

  ├─> String stockSymbol = request.getStockSymbol()
  │   └─> extracts "AAPL" from request
  │
  ├─> Stock stockEntity = stockRepository.findByStockSymbol(stockSymbol)
  │   │   (Database query via Spring Data JPA)
  │   │
  │   ├─> JpaRepository.findByStockSymbol(String)
  │   │   └─> generates SQL: SELECT * FROM stocks WHERE stock_symbol = ?
  │   │       └─> executes query with parameter "AAPL"
  │   │       └─> MySQL returns matching row(s)
  │   │       └─> Hibernate ORM maps row to Stock entity
  │   │           ├─> id = 1
  │   │           ├─> stock_symbol = "AAPL"
  │   │           ├─> price = 189.45
  │   │           └─> last_updated = 2026-05-08T10:30:00
  │   │
  │   └─> returns Stock entity object
  │
  ├─> StockResponse.newBuilder()
  │   └─> returns ResponseBuilder instance
  │
  ├─> StockResponse.Builder.setStockSymbol(stockEntity.getStockSymbol())
  │   └─> sets stock_symbol = "AAPL"
  │
  ├─> StockResponse.Builder.setPrice(stockEntity.getPrice())
  │   └─> sets price = 189.45
  │
  ├─> StockResponse.Builder.setTimestamp(
  │                    stockEntity.getLastUpdated().toString())
  │   └─> converts LocalDateTime to String
  │       └─> sets timestamp = "2026-05-08T10:30:00"
  │
  ├─> StockResponse.Builder.build()
  │   └─> returns StockResponse object (ready to send)
  │
  ├─> responseObserver.onNext(stockResponse)
  │   └─> sends StockResponse back to client
  │       └─> serializes to protobuf bytes
  │       └─> transmits over gRPC channel to client:xxxxx
  │
  └─> responseObserver.onCompleted()
      └─> signals end of RPC
      └─> closes response stream
      └─> method returns
```

---

### 5.3 Complete End-to-End Flow Diagram

```
┌──────────────────────────┐                         ┌──────────────────────────┐
│    STOCK TRADING CLIENT  │                         │  STOCK TRADING SERVICE   │
└──────────────────────────┘                         └──────────────────────────┘

1. main() starts
   │
2. run() executes
   │
3. getStockPrice("AAPL")
   ├─ StockRequest.newBuilder()
   ├─ setStockSymbol("AAPL")
   ├─ build() creates request
   │
4. stub.getStockPrice(request) ──────────────────────> 5. gRPC server receives
   (blocks waiting)                                       │
                                                      6. Deserialize request
                                                         │
                                                      7. getStockPrice(request, observer)
                                                         ├─ getStockSymbol()
                                                         ├─ findByStockSymbol("AAPL")
                                                         │  ├─ SQL query to MySQL
                                                         │  └─ Hibernate ORM maps row
                                                         │     to Stock entity
                                                         ├─ StockResponse.newBuilder()
                                                         ├─ setStockSymbol("AAPL")
                                                         ├─ setPrice(189.45)
                                                         ├─ setTimestamp(...)
                                                         ├─ build() creates response
                                                         │
8. Receives StockResponse <───────── onNext() sends  9. responseObserver.onNext()
   │                          response back            │
   │                                               10. responseObserver.onCompleted()
9. System.out.println()
   │
10. Client shuts down

```

---

### 5.4 Key Method Details

#### Client-Side Key Methods

| Class | Method | Purpose |
|-------|--------|---------|
| `StockTradingClientApplication` | `main(String[])` | Entry point; starts Spring context |
| `StockTradingClientApplication` | `run(String...)` | Implements CommandLineRunner; triggers RPC call |
| `StockClientService` | `getStockPrice(String)` | Builds request, calls stub, returns response |
| `StockRequest.Builder` | `newBuilder()` | Creates a builder for StockRequest |
| `StockRequest.Builder` | `setStockSymbol(String)` | Sets the stock_symbol field |
| `StockRequest.Builder` | `build()` | Constructs immutable StockRequest |
| `StockTradingServiceBlockingStub` | `getStockPrice(StockRequest)` | Sends RPC and blocks for response |

#### Server-Side Key Methods

| Class | Method | Purpose |
|-------|--------|---------|
| `StockTradingServiceApplication` | `main(String[])` | Entry point; starts Spring context & gRPC server |
| `StockTradingServiceImpl` | `getStockPrice(StockRequest, StreamObserver)` | Implements the RPC handler |
| `StockRequest` | `getStockSymbol()` | Extracts the stock symbol from request |
| `StockRepository` | `findByStockSymbol(String)` | JPA method; queries database |
| `Stock` | `getStockSymbol()` | Returns symbol from entity |
| `Stock` | `getPrice()` | Returns price from entity |
| `Stock` | `getLastUpdated()` | Returns timestamp from entity |
| `StockResponse.Builder` | `newBuilder()` | Creates a builder for StockResponse |
| `StockResponse.Builder` | `setStockSymbol(String)` | Sets the stock_symbol field |
| `StockResponse.Builder` | `setPrice(double)` | Sets the price field |
| `StockResponse.Builder` | `setTimestamp(String)` | Sets the timestamp field |
| `StockResponse.Builder` | `build()` | Constructs immutable StockResponse |
| `StreamObserver<StockResponse>` | `onNext(StockResponse)` | Sends response to client |
| `StreamObserver<StockResponse>` | `onCompleted()` | Signals RPC completion |

---

### 5.5 gRPC Parameters and Callbacks Explained

This section explains the critical gRPC parameters and callback mechanisms used in the unary RPC handler.

#### StockRequest request

**Type:** `com.javatechie.grpc.StockRequest`

**Description:**
The incoming gRPC request object containing the stock symbol sent by the client.

**Characteristics:**
- Automatically deserialized from protobuf bytes by the gRPC framework
- Contains fields defined in the `.proto` file:
  - `stock_symbol` (String) — the symbol to look up

**Usage in Handler:**
```java
String stockSymbol = request.getStockSymbol();  // Extract the symbol
```

**Example Value:**
- When client calls `getStockPrice("AAPL")`, the request object contains:
  - `stock_symbol: "AAPL"`

---

#### StreamObserver<StockResponse>

**Type:** `io.grpc.stub.StreamObserver<com.javatechie.grpc.StockResponse>`

**Description:**
A gRPC callback object used to send the response back to the client. This is the asynchronous interface that allows the server to communicate with the client.

**Characteristics:**
- Provided by gRPC framework automatically
- Implements callback pattern for sending messages
- Required even for unary calls (which only send one response)
- Used to signal both data and completion/errors

**Methods:**
1. `onNext(StockResponse)` — delivers the response to the client
2. `onCompleted()` — signals successful completion of the RPC
3. `onError(Throwable)` — signals an error occurred

**Usage in Handler:**
```java
responseObserver.onNext(stockResponse);    // Send the response
responseObserver.onCompleted();            // Signal completion
```

---

#### onNext() Method

**Signature:**
```java
void onNext(StockResponse response)
```

**Description:**
Delivers the response to the client.

**What It Does:**
1. Takes the `StockResponse` object
2. Serializes it to protobuf bytes
3. Transmits the bytes over the gRPC channel to the client
4. The client deserializes and receives the `StockResponse`

**In Unary Calls:**
- Called exactly **once** to send the single response
- If called multiple times, only the first call's result is used

**Blocking Behavior:**
- For the blocking client stub (`StockTradingServiceBlockingStub`), this method:
  - Unblocks the client from waiting
  - Returns the `StockResponse` to the client code

**Example Flow:**
```
Server: responseObserver.onNext(stockResponse)
         │
         ├─ serializes StockResponse to protobuf bytes
         ├─ sends bytes over gRPC channel (localhost:9090)
         │
Client: (unblocks from stub.getStockPrice() call)
         └─ receives and deserializes response
            └─ returns StockResponse object
```

---

#### onCompleted() Method

**Signature:**
```java
void onCompleted()
```

**Description:**
Tells gRPC that the call is finished. This is **required** even for unary calls.

**What It Does:**
1. Signals to gRPC that the RPC stream is complete
2. No more messages will be sent (`onNext()` calls are no longer valid)
3. Indicates successful completion (no errors)
4. Closes the response stream

**Why It's Required:**
- gRPC uses this to know when the response is fully delivered
- Without it, the client may wait indefinitely
- Prevents resource leaks by properly closing the stream

**Order of Calls:**
```
Must be called AFTER onNext():
  1. responseObserver.onNext(response)     ← Send the response
  2. responseObserver.onCompleted()        ← Signal completion
```

**Never Mix With onError():**
- Call either `onCompleted()` for success
- Or `onError(throwable)` for failure
- Never call both

**Example:**
```java
@Override
public void getStockPrice(StockRequest request,
                          StreamObserver<StockResponse> responseObserver) {
    // ... process request, build response ...
    
    StockResponse response = StockResponse.newBuilder()
        .setStockSymbol("AAPL")
        .setPrice(189.45)
        .setTimestamp("2026-05-08T10:30:00")
        .build();
    
    responseObserver.onNext(response);      // Send response
    responseObserver.onCompleted();         // Signal done
}
```

---

#### Complete Callback Sequence

```
Server Handler Execution:
├─ getStockPrice(request, responseObserver) called by gRPC
├─ Process request (extract stock symbol, query database)
├─ Build StockResponse object
├─ responseObserver.onNext(response)
│  └─> Response sent to client
├─ responseObserver.onCompleted()
│  └─> Stream closed successfully
└─ Handler method returns

Client Receives:
├─ stub.getStockPrice(request) unblocks
├─ Returns StockResponse object
└─ Application continues
```

---
