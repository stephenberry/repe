# REPE (Remote Efficient Protocol Extension)

> This specification is under ACTIVE DEVELOPMENT and should be considered UNSTABLE.

REPE is a fast and simple RPC protocol for the binary [BEVE](https://github.com/stephenberry/beve) specification or for JSON.

- High performance
- Easy to use
- Easy to parse

## Design

- The header and body are separate to allow parsers to verify a header and run logic prior to parsing any of the body.

# Request/Response (Message)

Requests and responses use the exact same data layout. There is no distinction between them.

BEVE Layout: `Header | Delimiter | Optional User Data | Delimiter | Body | Delimiter`

JSON Layout: `Header array | Optional User Data | Body value`

- Where JSON elements are delimited by new lines `\n`

# Header

The header is sent as an array of `[version, error, notification, user_data, method, id]`. No elements are optional, but the `id`  may be `null`.

```c++
// C++ pseudocode
struct header {
  uint8_t version = 0; // the REPE version
  uint8_t error = 0; // 0 denotes no error
  uint8_t notification = 0; // whether this RPC is a notification (no response returned)
  uint8_t user_data = 0; // whether this RPC contains user data
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

### User Data

`user_data` is a single byte `uint8_t`. A value of 1 represents user data being present.

### Method

`method` must be a string type of UTF-8 characters.

### ID

`id` is either null, a `uint64_t`, or a `string`.

### Example JSON Header

```json
[0,0,0,"method","id"]
```

# Delimiter

The BEVE data delimiter is used to separate the header, parameters, and the body.

```c++
0b00000'110 // delimiter in binary
```

# Optional User Data

Optional user data must conform to a BEVE or JSON `VALUE`.

# Body

For a request call, the body contains the input parameters. For a response, the body is the result of the call.

`VALUE` or `ERROR`

The body may be any BEVE value. For example, it could be an object, array, or single number. A body must exist, but it could simply be a null value.

> Allowing any BEVE/JSON value makes interfaces more efficient by avoiding additional wrapping inside structures like arrays.

## Error

An error requires an error code (`int32_t`) and a string message. Additional BEVE data may be returned. The serialization format must be an array.

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
  T& data;
};
// or
template <>
struct error<void>
{
  int32_t code = 0;
  std::string message = "";
};
```
