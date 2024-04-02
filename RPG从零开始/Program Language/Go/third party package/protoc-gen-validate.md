# protoc-gen-validate

PGV是一个protoc的插件，虽然protobuf可以有效保证数据的结构类型，但是它没有数据的语义校验规则，这个插件就是通过生成代码来完成这种校验规则限制

## 用法

### Dependencies

- `go` toolchain (≥ v1.7)
- `protoc` compiler in `$PATH`
- `protoc-gen-validate` in `$PATH`
- official language-specific plugin for target language(s)
- **Only `proto3` syntax is currently supported.** `proto2` syntax support is planned.

### Parameters

- **`lang`**: specify the target language to generate. Currently, the only supported options are:
    - `go`
    - `cc` for c++ (partially implemented)
    - `java`
- Note: Python works via runtime code generation. There's no compile-time generation. See the Python section for details.

### Go Example

Go generation should occur into the same output path as the official plugin. For a proto file `example.proto`, the corresponding validation code is generated into `../generated/example.pb.validate.go`:

```jsx
protoc \
  -I . \
  -I path/to/validate/ \
  --go_out=":../generated" \
  --validate_out="lang=go:../generated" \
  example.proto
```

All messages generated include the following methods:

- `Validate() error` which returns the first error encountered during validation.
- `ValidateAll() error` which returns all errors encountered during validation.

## 校验规则

- numerics
- bools
- strings
- bytes
- enums
- messages
- repeated
- maps

### Well-Known Types(WKTs)

A set of [WKTs](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf) are packaged with protoc and common message patterns useful in many domains.

- Anys
- Durations
- Timestamps
- Message-Global
- OneOfs

reference:

[https://github.com/envoyproxy/protoc-gen-validate](https://github.com/envoyproxy/protoc-gen-validate)