# REPE (Remote Efficient Protocol Extension)

> [!CAUTION]
>
> This specification is under ACTIVE DEVELOPMENT and should be considered UNSTABLE.

REPE is a fast and simple RPC/Database protocol that can package any data format and any query specification. It defines a binary header that provides a simple action protocol, which can be invoked with any query specification. The payload (body) can be any format, JSON, BEVE, raw binary, text, etc.

- High performance
- Easy to use
- Easy to parse

> The motivation is to provide a standard way to build RPC interfaces and databases with flexible formats and query specifications.

# Version 1

> [!IMPORTANT]
>
> This Version 1 of the specification is a complete rework in order to support any data format. It also embeds the message size and uses a packed binary header for better efficiency and smaller messages.

## Endianness

The endianness must be `little endian`, selected for optimal performance on modern systems.

## Strings

All strings referred to in this specification must be UTF-8 compliant. Implementers must validate string encodings upon receiving messages.

# Request/Response (Message)

Requests and responses use the exact same data layout. There is no distinction between them. The format contains a header, query, and body.

REPE Layout: `[Header, Query, Body]`

```c++
// C++ pseudocode representing a complete REPE message
struct message {
  repe::header header{}; // fixed size header
  std::string query{}; // dynamic size query
  std::string body{}; // dynamic size body
};
```

# Header

The header information that describes how to handle the request and payload (query & body).

```c++
enum struct Action : uint32_t
{
  notify = 1 << 0, // If this message does not require a response
  get = 1 << 1, // Read a value
  set = 1 << 2, // Write a value
  call = 1 << 3 // Call a function with the body as input
};

constexpr auto no_length_provided = std::numeric_limits<uint64_t>::max();

// C++ pseudocode representing layout
struct header {
  uint64_t length{no_length_provided}; // Total length of [header, query, body]
  //
  uint16_t spec{0x1507}; // (5383) Magic two bytes to denote the REPE specification
  uint8_t version = 1; // REPE version
  bool error{}; // Whether an error has occurred
  Action action{}; // Action to take, multiple actions may be bit-packed together
  //
  uint64_t id{}; // Identifier
  //
  uint64_t query_length{no_length_provided}; // The total length of the query
  //
  uint64_t body_length{no_length_provided}; // The total length of the body
  //
  uint16_t query_format{};
  uint16_t body_format{};
  uint32_t reserved{}; // Must be set to zero
};
```

The header must be exactly 48 bytes allocated in the layout shown above. All fields in the header must be aligned to their natural boundaries (e.g., `uint64_t` on 8-byte boundaries). No padding bytes are included whatsoever.

### Length

The `length` field represents the total length of `[header, query, body]`. A value of `-1` (wrapped to the largest `uint64_t`) indicates that the length is unspecified. In such cases, the message termination must be determined by alternative means.

### Spec

The REPE `spec` value of `0x1507` is used to disambiguate REPE from other specifications that may use a similar layout. In this way manner branching may be performed on the first 8 bytes (length + spec).

### Version

The `version` must be a `uint8_t`.

> The latest version is `1`

This version of REPE does not require backwards compatibility, but errors must be returned for mismatching versions.

### Error Boolean

`error` is a single byte `bool` to denote that an error occurred. The body will contain the error information.

- `0x00` denotes `false` (no error)

- `0x01` denotes `true` (error occurred)

All other values are considered invalid.

### Action

The `Action` enumeration defines the operation to perform. Multiple actions can be combined using bitwise OR, but certain combinations may have specific interpretations.

- `notify` - Denotes that no response should be sent. May be combined with `set` or `call` actions.
- `get` - Reads a value denoted by the query. The `body_length` must be zero.
- `set` - Writes the body to the value denoted by the query.
- `call` - Calls a function at the query location with the body as input.

`notify | set`: Indicates a notification to set a value without expecting a response.

 `notify | call`: Indicates a notification to call a function without expecting a response.

Invalid combinations (e.g., `get | call`) should result in an error.

### ID

`id` must be a `uint64_t`.

The `id` field may be used as a unique `uint64_t` identifier for each request. Responses must use the same `id` to allow clients to match responses to their corresponding requests, for asynchronous communication.

### Query Length

`query_length` indicates the length in bytes of the query. A value of `-1` (wraps to largest uint64_t) denotes that no length is provided and the message termination will be determined some other way, such as the end of a valid parse.

### Body Length

`body_length` indicates the length in bytes of the body. A value of `-1` (wraps to largest uint64_t) denotes that no length is provided and the message termination will be determined some other way, such as the end of a valid parse.

If `body_length` is set to zero, the body section is omitted. Implementers must ensure that actions requiring a body (e.g., `set`, `call`) are not used with `body_length` zero, and respond with an appropriate error if such a mismatch occurs.

### String Length Constraints

While the protocol supports strings of extremely long length, implementers should enforce reasonable maximum lengths based on application requirements to prevent resource exhaustion and ensure efficient processing.

### Query Format

`query_format` is a two byte unsigned integer that may denote the format for the query.

**Reserved Query Formats**

```
1 - JSON Pointer Syntax
```

### Body Format

`query_format` is a two byte unsigned integer that may denote the format for the body.

**Reserved Body Formats**

```
1 - BEVE
2 - JSON
```

### Reserved Space

The reserved 4s bytes in the header are for potential future versions of REPE.

# Query

The query may use any specification chosen by implementors.

# Body

The body can contain data in any format, including JSON, BEVE, raw binary, or text. Implementers should document the expected data formats for their specific use cases to ensure consistent parsing and handling.

The body must contain a REPE error if the `error` boolean in the header is set to `true`. If the `body_length` is set to zero, then no body is provided.

For a `call` action, the body contains the input parameters.

## Error

An error requires a `uint32_t` error code and a string message.

```c++
// C++ pseudocode representing layout
struct error {
  uint32_t code = 0; // 0 is OK (no error)
  uint32_t message_length{};
  const char* message{};
};
```

Below is a table of defined error codes. Values from `0` to `4095` are reserved for REPE high-level error codes. Codes `4096` and above are available for application-specific errors.

| code | message          | meaning                                                      |
| :--- | :--------------- | :----------------------------------------------------------- |
| 0    | OK               | No error                                                     |
| 1    | Version mismatch | Mismatching REPE version                                     |
| 2    | Invalid header   | The REPE header as invalid                                   |
| 3    | Invalid query    | The query was invalid                                        |
| 4    | Invalid body     | The body was invalid.                                        |
| 5    | Parse error      | Invalid data was received. An error occurred while parsing the data. |
| 6    | Method not found | The method does not exist / is not available.                |
| 7    | Timeout          | Timeout                                                      |

### Application-Specific Errors

 Applications may define their own error codes starting from `4096` to avoid conflicts with REPE's high-level errors.

# Response

RPC responses from the server should typically return an action with a pure `notify`. This indicates that the server is not expecting anything back from the client.

It is up to the discretion of implementors whether the response returns the original query. Data can be saved by not returning the requested query, but it may be useful for debugging errors.
