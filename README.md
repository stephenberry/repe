# REPE

REPE is a fast and simple RPC protocol for the binary [BEVE](https://github.com/stephenberry/beve) specification.

- High performance
- Easy to use
- Easy to parse

## Design

- The header and body are separated with a unique delimiter to allow parsers to verify a header and run logic prior to parsing any of the body.

# Request/Response

Requests and responses use the exact same data layout. There is no distinction between them.

Layout: `Header | Delimiter | Parameters | Delimiter | Body | Delimiter`

# Header

The header is sent as an array of `[version, error, notification, method, id]`. None of the fields are optional in the header, but the `id`  may be `null`.

```c++
// C++ pseudocode (serialized as a BEVE array)
struct header {
  uint8_t version = 0; // the RPCZ version
  bool error = false;
  bool notification = false; // whether this RPC is a notification (no response returned)
  std::string method = ""; // the RPC method to call
  std::variant<null_t, uint64_t, std::string> id{}; // an identifier
};
```

### Version

The `version` must be a `uint8_t`.

> The current version is `0`

### Error Flag

`error` is a single byte `bool` to denote that an error occurred. The body will contain the error information

### Notification

`notification` is a single byte `bool`. A notification does not expect a response.

### Method

`method` must be a string type of UTF-8 characters.

### ID

`id` is either null, a `uint64_t`, or a `string`.

# Delimiter

The BEVE data delimiter is used to separate the header, parameters, and the body.

```c++
0b00000'110 // delimiter in binary
```

# Parameters

The parameters may be any BEVE value. For example, it could be an object, array, a single number, or null. Response parameters will typically be null.

# Body

`VALUE` or `ERROR`

The body may be any BEVE value. For example, it could be an object, array, or single number. A body must exist, but it could simply be a null value.

> Allowing any BEVE value makes interfaces more efficient by avoiding additional wrapping inside structures like arrays.

## Error

An error requires an error code (`uint32_t`) and a string message. Additional BEVE data may be returned.

```c++
template <class T>
struct error_t
{
  uint32_t code = 0;
  std::string message = "";
  T data{};
};
// or
template <>
struct error_t<void>
{
  uint32_t code = 0;
  std::string message = "";
};
```

