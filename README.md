# REPE (Remote Efficient Protocol Extension)

> This specification is under ACTIVE DEVELOPMENT and should be considered UNSTABLE.

REPE is a fast and simple RPC protocol for JSON or the binary [BEVE](https://github.com/stephenberry/beve) specification.

- High performance
- Easy to use
- Easy to parse

Why not use JSON RPC 2.0? Besides needing binary support with BEVE, there are a number of [issues with JSON RPC 2.0](#json-rpc-2.0-issues) that REPE seeks to solve.

## Design

- The header and body are in an array, to allow parsers to verify a header and run logic prior to parsing any of the body.

# Request/Response (Message)

Requests and responses use the exact same data layout. There is no distinction between them. The format is an array of two elements.

BEVE Layout: `[Header, Body]`

JSON Layout: `[Header, Body]`

> If you require additional header information, such as a checksum, simply provide it as the first element of an array for your message body. This way you can inspect this additional information prior to parsing the rest of the body.

# Header

The header is sent as an array of `[version, error, notification, method, id]`. No elements are optional, but the `id`  may be `null`.

```c++
// C++ pseudocode
struct header {
  uint8_t version = 0; // the REPE version
  uint8_t error = 0; // 0 denotes no error
  uint8_t notification = 0; // whether this RPC is a notification (no response returned)
  std::string method = ""; // the RPC method to call
  std::variant<null_t, uint64_t, std::string> id{}; // an identifier
};
```

### Version

The `version` must be a `uint8_t`.

> The latest version is `0`

### Error Flag

`error` is a single byte `uint8_t` to denote that an error occurred. The body will contain the error information.

### Notification

`notification` is a single byte `uint8_t`. A value of 1 represents a notification. A notification does not expect a response.

### Method

`method` must be a string type of UTF-8 characters.

### ID

`id` is either null, a `uint64_t`, or a `string`.

### Example JSON Header

```json
[0,0,0,"method","id"]
```

# Body

For a request call, the body contains the input parameters. For a response, the body is the result of the call.

`VALUE` or `ERROR`

The body may be any BEVE value. For example, it could be an object, array, or single number. A body must exist, but it could simply be a null value.

> Allowing any BEVE/JSON value makes interfaces more efficient by avoiding additional wrapping inside structures like arrays.

## Error

An error requires an error code (`int32_t`) and a string message. The serialization format must be an array.

Reserved error codes match [JSON RPC 2.0](https://www.jsonrpc.org/specification) and are nearly the same as those suggested for [XML-RPC](http://xmlrpc-epi.sourceforge.net/specs/rfc.fault_codes.php)

| code             | message          | meaning                                                      |
| :--------------- | :--------------- | :----------------------------------------------------------- |
| -32700           | Parse error      | Invalid data was received by the server. An error occurred on the server while parsing the data. |
| -32600           | Invalid Request  | The data sent is not a valid Request object.                 |
| -32601           | Method not found | The method does not exist / is not available.                |
| -32602           | Invalid params   | Invalid method parameter(s).                                 |
| -32603           | Internal error   | Internal REPE error.                                         |
| -32000 to -32099 | Server error     | Reserved for implementation-defined server-errors.           |

```c++
template <class T = void>
struct error
{
  int32_t code = 0;
  std::string message = "";
};
```

## Deducing between BEVE and JSON

The specification does not include a format indicator, as we don't want to require multiple formats to be handled by the server. Additionally, we don't want to require servers to build handling both BEVE and JSON. However, the message type can be determine from the first byte, since JSON messages must begin with `[` and the BEVE header must begin with a uint8_t tag ` 0b000'10'001`, which is the ascii value of `device control 1`.

## JSON RPC 2.0 Issues

The issues with JSON RPC 2.0 that REPE seeks to address:
- JSON RPC 2.0 sends the `method` and `id` in the same object as the parameters or result. Because JSON is unordered this means the entire object needs to be parsed before logical decisions about handling the params or result can be made. One could argue that the sequence could be required, but this would be breaking the JSON specification.
- JSON RPC 2.0 sends the same keys over and over for "header" information, which can be easily deduced from an array. This wastes space and reduces performance for small messages.
- JSON RPC 2.0 has different structures for the request and the response. However, it is often useful to use a response as a request, and this difference in format creates performance issues. It also makes the specification and implementations more complex.
- JSON RPC 2.0 does not define the specification at the binary level. This makes it less efficient for implementing in C++ and other compiled languages. In JSON RPC 2.0 the id is only discouraged from being a floating point number. REPE is more restrictive, defining the integer `id` as a `uint64_t`, which makes implementations simpler and faster.
- Notifications in JSON RPC 2.0 are not explicit. Omitting the `id` makes the request a notification. However, it can be useful to make notification calls with a specified `id`. Sometimes the response will be generated from another network path, and we want to stop results from being returned from the primary RPC path. REPE allows notifications to have an `id`, which allows more flexible use of notifications and can also help with debugging.
