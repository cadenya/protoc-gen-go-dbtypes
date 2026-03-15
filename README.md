# protoc-gen-go-dbtypes

A protoc plugin that generates Go wrapper types for protobuf messages, enabling seamless database scanning and valuing via the `sql.Scanner` and `driver.Valuer` interfaces.

## Overview

When storing protobuf messages in databases as binary blobs, you typically need to manually marshal/unmarshal the data. This plugin generates wrapper types that handle this automatically, allowing you to use protobuf messages directly with Go's `database/sql` package.

For each protobuf message, the plugin generates a `*Value` wrapper type (e.g., `ToolSetSpec` becomes `ToolSetSpecValue`) that implements the database interfaces.

## Installation

### Prerequisites

- Go 1.21 or later
- `buf` CLI (recommended) or `protoc`
- `protoc-gen-go` for standard protobuf generation

### Install the Plugin

```bash
go install github.com/cadenya/protoc-gen-go-dbtypes/cmd/protoc-gen-go-dbtypes@latest
```

Or build from source:

```bash
git clone https://github.com/cadenya/protoc-gen-go-dbtypes.git
cd protoc-gen-go-dbtypes
go install ./cmd/protoc-gen-go-dbtypes
```

## Configuration

### With Buf (Recommended)

Add the plugin to your `buf.gen.yaml`:

```yaml
version: v2
plugins:
  # Standard Go protobuf generation
  - local: protoc-gen-go
    out: gen/go
    opt:
      - paths=source_relative

  # DBTypes wrapper generation
  - local: protoc-gen-go-dbtypes
    out: gen/go
    opt:
      - paths=source_relative
```

Then run:

```bash
buf generate
```

### With protoc

```bash
protoc \
  --go_out=gen/go --go_opt=paths=source_relative \
  --go-dbtypes_out=gen/go --go-dbtypes_opt=paths=source_relative \
  proto/your/package/v1/messages.proto
```

### Plugin Options

| Option | Description |
|--------|-------------|
| `paths=source_relative` | Generate files relative to the source proto file location |
| `exclude=Name1,Name2` | Comma-separated list of message names to exclude from generation |
| `package=example.v1` | Only generate for the specified proto package |

The `exclude` option accepts both Go type names (e.g., `UserPreferences`) and full proto names (e.g., `example.v1.UserPreferences`).

Example with exclusions:

```yaml
# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-go-dbtypes
    out: gen/go
    opt:
      - paths=source_relative
      - exclude=InternalMessage,DebugInfo
```

Example filtering to a specific package:

```yaml
# buf.gen.yaml
version: v2
plugins:
  - local: protoc-gen-go-dbtypes
    out: gen/go
    opt:
      - paths=source_relative
      - package=myapp.api.v1
```

## Generated Code

Given a protobuf message:

```protobuf
syntax = "proto3";
package example.v1;

option go_package = "github.com/example/gen/go/example/v1;examplev1";

message ToolSetSpec {
  repeated string tool_ids = 1;
  string name = 2;
  bool enabled = 3;
}
```

The plugin generates `*_dbtypes.pb.go` containing:

```go
// ProtoValue wraps a protobuf message for database scanning/valuing.
type ProtoValue[T proto.Message] struct {
    Message T
}

func (p *ProtoValue[T]) Scan(src any) error { ... }
func (p *ProtoValue[T]) Value() (driver.Value, error) { ... }

// ToolSetSpecValue wraps *ToolSetSpec for database operations.
type ToolSetSpecValue struct {
    *ProtoValue[*ToolSetSpec]
}

// NewToolSetSpecValue creates a new ToolSetSpecValue wrapper.
func NewToolSetSpecValue(msg *ToolSetSpec) *ToolSetSpecValue { ... }

// Scan implements sql.Scanner.
func (x *ToolSetSpecValue) Scan(src any) error { ... }

// Value implements driver.Valuer.
func (x *ToolSetSpecValue) Value() (driver.Value, error) { ... }

// Unwrap returns the underlying protobuf message.
func (x *ToolSetSpecValue) Unwrap() *ToolSetSpec { ... }
```

## Usage

### Basic Usage

```go
import (
    examplev1 "github.com/example/gen/go/example/v1"
)

// Create a wrapper from an existing message
spec := &examplev1.ToolSetSpec{
    ToolIds: []string{"tool-1", "tool-2"},
    Name:    "my-toolset",
    Enabled: true,
}
wrapper := examplev1.NewToolSetSpecValue(spec)

// Access the underlying message
msg := wrapper.Unwrap()
fmt.Println(msg.Name) // "my-toolset"
```

### Database Models

Define your database model with the wrapper type:

```go
type Tool struct {
    ID        string
    Name      string
    Spec      *examplev1.ToolSetSpecValue
    CreatedAt time.Time
}
```

### Inserting Records

```go
func (r *ToolRepo) Create(ctx context.Context, tool *Tool) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO tools (id, name, spec, created_at) VALUES ($1, $2, $3, $4)",
        tool.ID,
        tool.Name,
        tool.Spec,      // Automatically marshals to binary
        tool.CreatedAt,
    )
    return err
}
```

### Querying Records

```go
func (r *ToolRepo) GetByID(ctx context.Context, id string) (*Tool, error) {
    tool := &Tool{
        Spec: &examplev1.ToolSetSpecValue{}, // Initialize the wrapper
    }

    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, spec, created_at FROM tools WHERE id = $1",
        id,
    ).Scan(&tool.ID, &tool.Name, &tool.Spec, &tool.CreatedAt)

    if err != nil {
        return nil, err
    }
    return tool, nil
}
```

### Modifying and Saving

```go
func (r *ToolRepo) AddTool(ctx context.Context, id string, toolID string) error {
    tool, err := r.GetByID(ctx, id)
    if err != nil {
        return err
    }

    // Access and modify the underlying proto
    spec := tool.Spec.Unwrap()
    spec.ToolIds = append(spec.ToolIds, toolID)

    // Save back to database
    _, err = r.db.ExecContext(ctx,
        "UPDATE tools SET spec = $1 WHERE id = $2",
        tool.Spec,
        id,
    )
    return err
}
```

### Handling NULL Values

The wrapper handles NULL database values gracefully:

```go
func (r *ToolRepo) GetByID(ctx context.Context, id string) (*Tool, error) {
    tool := &Tool{
        Spec: &examplev1.ToolSetSpecValue{},
    }

    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, spec, created_at FROM tools WHERE id = $1",
        id,
    ).Scan(&tool.ID, &tool.Name, &tool.Spec, &tool.CreatedAt)

    if err != nil {
        return nil, err
    }

    // Check if the spec was NULL in the database
    if tool.Spec.Unwrap() == nil {
        // Handle NULL case
    }

    return tool, nil
}
```

### Creating Empty Wrappers

```go
// Pass nil to create a wrapper with an empty message
wrapper := examplev1.NewToolSetSpecValue(nil)

// The wrapper is initialized with an empty ToolSetSpec
spec := wrapper.Unwrap()
spec.Name = "new-toolset"
```

## Database Schema

Store protobuf messages as binary columns:

### PostgreSQL

```sql
CREATE TABLE tools (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    spec BYTEA,  -- Binary data
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### MySQL

```sql
CREATE TABLE tools (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    spec BLOB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### SQLite

```sql
CREATE TABLE tools (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    spec BLOB,
    created_at TEXT NOT NULL
);
```

## Supported Data Types

The `Scan` method accepts:

- `[]byte` - Binary data (most common)
- `string` - String data (some databases return this)
- `nil` - NULL values

The `Value` method returns:

- `[]byte` - Marshaled protobuf binary
- `nil` - If the wrapper or message is nil

## Nested Messages

The plugin generates wrappers for all top-level messages, including those with nested messages:

```protobuf
message Container {
  string id = 1;
  ToolSetSpec spec = 2;
  repeated Item items = 3;

  message Item {
    string key = 1;
    string value = 2;
  }
}
```

This generates `ContainerValue` which serializes the entire message including nested `spec` and `items` fields.

## Skipped Types

The plugin automatically skips:

- Map entry messages (internal protobuf types)

## Error Handling

Scan errors occur when:

- The source data is not `[]byte`, `string`, or `nil`
- The binary data cannot be unmarshaled into the protobuf message

```go
err := wrapper.Scan(someValue)
if err != nil {
    // Handle unmarshal error or unsupported type
    log.Printf("scan error: %v", err)
}
```

## Comparison with Alternatives

### Manual Marshaling

Without this plugin:

```go
// Insert
data, err := proto.Marshal(spec)
if err != nil {
    return err
}
_, err = db.Exec("INSERT INTO tools (spec) VALUES ($1)", data)

// Query
var data []byte
err := db.QueryRow("SELECT spec FROM tools WHERE id = $1", id).Scan(&data)
if err != nil {
    return err
}
spec := &examplev1.ToolSetSpec{}
if err := proto.Unmarshal(data, spec); err != nil {
    return err
}
```

With this plugin:

```go
// Insert
_, err = db.Exec("INSERT INTO tools (spec) VALUES ($1)", wrapper)

// Query
wrapper := &examplev1.ToolSetSpecValue{}
err := db.QueryRow("SELECT spec FROM tools WHERE id = $1", id).Scan(&wrapper)
spec := wrapper.Unwrap()
```

### JSON Storage

If you prefer JSON storage over binary protobuf, consider using `protojson` with a custom type. Binary protobuf offers:

- Smaller storage size
- Faster serialization/deserialization
- Schema evolution support via protobuf's compatibility guarantees

## License

MIT License
