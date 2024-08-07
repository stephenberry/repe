# REPE (Remote Efficient Protocol Extension)

> [!CAUTION]
>
> This specification is under ACTIVE DEVELOPMENT and should be considered UNSTABLE.

REPE is a fast and simple RPC protocol that can package any data format. It defines a simple binary header that pairs with a payload that can be JSON, BEVE, or any other binary or text format.

- High performance
- Easy to use
- Easy to parse

# Version 1

> [!IMPORTANT]
>
> This Version 1 of the specification is a complete rework in order to support any data format instead of just JSON and BEVE. It also embeds the message size and uses a packed binary header for more efficiency, smaller messages, and reserves fields to grow the specification.

# Request/Response (Message)

Requests and responses use the exact same data layout. There is no distinction between them. The format contains a header and then a body.

Layout: `[Header, Body]`

> If you require additional header information, such as a checksum, simply provide it as the first element of an array for your message body. This way you can inspect this additional information prior to parsing the rest of the body.

## Strings

All strings referred to in this specification must be UTF-8 compliant.

# Header

The header is sent as a packed 32 byte struct. Only the `path` name is optional if the `path_size` is zero.

```c++
enum struct Action : uint32_t
{
  notify = 1 << 0, // if this message does not require a response
  get = 1 << 1, // read a value
  set = 1 << 2, // write a value
  call = 1 << 3 // call a function with the body as input
};

// C++ pseudocode representing layout
struct header {
  uint8_t version = 1; // the REPE version
  bool error{}; // whether an error has occurred
  uint16_t reserved1{};
  Action action{};
  // ----
  uint64_t id{}; // identifier
  uint64_t body_length{-1}; // the total length of the body (-1 denotes no size given)
  uint32_t reserved2{};
  uint16_t reserved3{};
  uint16_t path_length{}; // the length of the path
  char path[256]{}; // the JSON Pointer path
};
```

If the `path_length` is zero length, then the header will be 32 bytes. The header size can always be calculated as `32 + path_length`.

> The header is designed so that it can be directly streamed over a network without any additional encoding. A pointer to the header can be provided along with the size as `32 + path_length`. This is if the method memory is allocated in the struct as in this example.

### Version

The `version` must be a `uint8_t`.

> The latest version is `1`

### Error Boolean

`error` is a single byte `bool` to denote that an error occurred. The body will contain the error information. `false` means no error.

### Action

`action` is an enumeration that may have multiple bits set to denote how to handle the request.

### ID

`id` must be a `uint64_t`.

### Body Length

`body_length` indicates the length in bytes of the body. A value of `-1` (wraps to largest uint64_t) denotes that no length is provided and the message termination will be determined some other way, such as the end of a valid parse.

The `body_length` must be a `uint64_t`.

## Path Length

The number of bytes used in the path string.

### Path

`path` must be a valid JSON Pointer.

# Body

For a request call, the body contains the input parameters. For a response, the body is the result of the call.

`VALUE` or `ERROR` or `EMPTY`

The body may be any text or binary value. If `error` is true, then the body must follow the specification for errors below.

## Error

An error requires an error code (`int32_t`) and a string message.

```c++
// C++ pseudocode representing layout
struct error {
  int32_t code = 0;
  uint32_t message_length{};
  char message[256]{};
};
```

Reserved error codes match [JSON RPC 2.0](https://www.jsonrpc.org/specification) and are nearly the same as those suggested for [XML-RPC](http://xmlrpc-epi.sourceforge.net/specs/rfc.fault_codes.php).

| code             | message          | meaning                                                      |
| :--------------- | :--------------- | :----------------------------------------------------------- |
| -32700           | Parse error      | Invalid data was received by the server. An error occurred on the server while parsing the data. |
| -32600           | Invalid Request  | The data sent is not a valid Request object.                 |
| -32601           | Method not found | The method does not exist / is not available.                |
| -32602           | Invalid params   | Invalid method parameter(s).                                 |
| -32603           | Internal error   | Internal REPE error.                                         |
| -32000 to -32099 | Server error     | Reserved for implementation-defined server-errors.           |

## Response

It is up to the discretion of implementors whether the response returns the `path` of the original request. Data can be saved by not returning the requested path, but it may be useful for debugging errors.
