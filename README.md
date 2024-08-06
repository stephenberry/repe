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

The header is sent as a packed 32 byte struct. Only the `method` name is optional if the `method_size` is zero.

```c++
// C++ pseudocode representing layout
struct header {
  uint8_t version = 1; // the REPE version
  bool error{}; // whether an error has occurred
  bool notify{}; // whether this message does not require a response
  bool has_body{}; // whether a body is provided
  uint32_t reserved1{};
  // ----
  uint64_t id{}; // identifier
  int64_t body_size = -1; // the total size of the body (-1 denotes no size given)
  uint32_t reserved2{};
  uint16_t reserved3{};
  uint16_t method_size{}; // the size of the method string
  char method[256]{}; // the method name
};
```

If the `method_size` is zero length, then the header will be 32 bytes. The header size can always be calculated as `32 + method_size`.

> The header is designed so that it can be directly streamed over a network without any additional encoding. A pointer to the header can simply be provided along with the size as `32 + method_size`. This is if the method memory is allocated in the struct as in this example.

### Version

The `version` must be a `uint8_t`.

> The latest version is `1`

### Error Boolean

`error` is a single byte `bool` to denote that an error occurred. The body will contain the error information. `false` means no error.

### Notify

`notify` is a single byte `bool` to denote that this message does not require a response.

### No Body

`no_body` is a single byte `bool` that indicates when no body is provided.

### Size

`size` indicates the size in bytes of the entire message, including both the header and the body. A value of `-1` denotes that no size is provided and the message termination will be determined some other way, such as the end of a valid parse.

The `size` must be an `int64_t`.

### Action

`action` is a single byte `uint8_t`. The individual bits on this byte are used to define separate actions. The least significant bit is the right-most bit and indexed with 0.

| Bit Index | Action                                                |
| --------- | ----------------------------------------------------- |
| 0         | notify (no response returned)                         |
| 1         | empty (if this bit is set the body should be ignored) |

In code, defining a notification that also should ignore the body might look like:

```c++
action = repe::notify | repe::empty;
```

> The `empty` action is useful to distinguish between a `null` body as a value and a `null` body that should be ignored.
>
> This mechanism allows variables to be registered with the RPC system, where `get` calls use `empty` to express that nothing is to be assigned and `set` calls leave this bit unset to indicate that the body should be used to assign to the variable.

### Method

`method` must be a string.

### ID

`id` is either `null`, a `uint64_t`, or a `string`.

# Body

For a request call, the body contains the input parameters. For a response, the body is the result of the call.

`VALUE` or `ERROR`

The body may be any JSON/BEVE value. For example, it could be an object, array, or single number. The body may not exist if `no_body` is set to `true`.

## Error

An error requires an error code (`int32_t`), a string category, and a string message. The serialization format must be an array.

The category for REPE errors is `"repe"`.

Reserved error codes match [JSON RPC 2.0](https://www.jsonrpc.org/specification) and are nearly the same as those suggested for [XML-RPC](http://xmlrpc-epi.sourceforge.net/specs/rfc.fault_codes.php).

| code             | message          | meaning                                                      |
| :--------------- | :--------------- | :----------------------------------------------------------- |
| -32700           | Parse error      | Invalid data was received by the server. An error occurred on the server while parsing the data. |
| -32600           | Invalid Request  | The data sent is not a valid Request object.                 |
| -32601           | Method not found | The method does not exist / is not available.                |
| -32602           | Invalid params   | Invalid method parameter(s).                                 |
| -32603           | Internal error   | Internal REPE error.                                         |
| -32000 to -32099 | Server error     | Reserved for implementation-defined server-errors.           |

```c++
struct error
{
  int32_t code = 0;
  std::string category = "repe";
  std::string message = "";
};
```

## Responses

It is up to the discretion of implementors whether the response returns the `method` of the original request. If you have an opinion about whether this should be required, please reach out as we formalize the specification.
