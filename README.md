# REPE (Remote Efficient Protocol Extension)

> This specification is under ACTIVE DEVELOPMENT and should be considered UNSTABLE.

REPE is a fast and simple RPC protocol for the binary [BEVE](https://github.com/stephenberry/beve) specification or for JSON.

- High performance
- Easy to use
- Easy to parse

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
