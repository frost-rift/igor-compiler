# Igor Language - C# and UE4 Target Reference

**Version:** Latest
**Last Updated:** February 2026

---

## Table of Contents

1. [Introduction](#introduction)
2. [Type System](#type-system)
   - [Primitive Types](#primitive-types)
   - [Composite Types](#composite-types)
   - [User-Defined Types](#user-defined-types)
3. [Attribute System](#attribute-system)
   - [Core Attributes](#core-attributes)
   - [C# Attributes](#c-attributes)
   - [UE4 Attributes](#ue4-attributes)
4. [Service & RPC System](#service--rpc-system)
5. [Serialization](#serialization)
6. [Web Services](#web-services)
7. [Complete Examples](#complete-examples)

---

## Introduction

Igor is a data definition language and code generator similar to Protocol Buffers/gRPC. It generates type-safe code for:
- **C#**: Classes, enums, services with async/await
- **UE4/C++**: Structs, enums with Unreal reflection (UENUM, USTRUCT, UPROPERTY)

Key features:
- Bidirectional RPC services (client-to-server and server-to-client)
- Multiple serialization formats (Binary, JSON, String)
- HTTP/REST API client generation
- CouchDB data modeling support
- Full attribute-driven code generation

---

## Type System

### Primitive Types

#### Integer Types

| Igor Type | C# Type | UE4 Type | Size | Range |
|-----------|---------|----------|------|-------|
| `sbyte`   | `sbyte` | `int8`   | 8-bit | -128 to 127 |
| `byte`    | `byte`  | `uint8`  | 8-bit | 0 to 255 |
| `short`   | `short` | `int16`  | 16-bit | -32,768 to 32,767 |
| `ushort`  | `ushort`| `uint16` | 16-bit | 0 to 65,535 |
| `int`     | `int`   | `int32`  | 32-bit | -2.1B to 2.1B (default) |
| `uint`    | `uint`  | `uint32` | 32-bit | 0 to 4.3B |
| `long`    | `long`  | `int64`  | 64-bit | Large signed |
| `ulong`   | `ulong` | `uint64` | 64-bit | Large unsigned |

**Example:**
```igor
record PlayerStats {
    byte level;        // 0-255
    short health;      // Can be negative
    uint score;        // Always positive
    long experience;   // Large numbers
}
```

#### Floating-Point Types

| Igor Type | C# Type  | UE4 Type | Precision |
|-----------|----------|----------|-----------|
| `float`   | `float`  | `float`  | Single precision (32-bit) |
| `double`  | `double` | `double` | Double precision (64-bit) |

**Example:**
```igor
record Position {
    float x;
    float y;
    float z;
    double precise_latitude;
}
```

#### String Types

| Igor Type | C# Type  | UE4 Type  | Description |
|-----------|----------|-----------|-------------|
| `string`  | `string` | `FString` | Dynamic UTF-16 string |
| `atom`    | `string` | `FName`   | Interned/pooled string (lightweight) |

**Usage:**
- Use `string` for general text, user input, messages
- Use `atom` for identifiers, keys, tags (memory efficient)

**Example:**
```igor
record Item {
    atom item_type;      // "weapon", "armor" (interned)
    string display_name;  // User-visible text
}
```

#### Other Primitive Types

| Igor Type | C# Type | UE4 Type | Description |
|-----------|---------|----------|-------------|
| `bool`    | `bool`  | `bool`   | Boolean true/false |
| `binary`  | `byte[]`| `FBufferArchive` | Binary data blob |
| `json`    | `Json.ImmutableJson` | `TSharedPtr<FJsonValue>` | JSON value |

**Example:**
```igor
record FileData {
    string filename;
    binary content;          // Raw file bytes
    json metadata;           // Flexible JSON metadata
}
```

---

### Composite Types

#### Lists

**Syntax:** `list<T>`

**C# Mapping:**
- `List<T>` (default, mutable)
- `IReadOnlyList<T>` (with `[csharp list_implementation=readonly]`)

**UE4 Mapping:**
- `TArray<T>` (Unreal dynamic array)

**Example:**
```igor
record Team {
    list<string> member_names;
    list<int> player_ids;
}
```

**Generated C#:**
```csharp
public class Team {
    public List<string> MemberNames { get; set; }
    public List<int> PlayerIds { get; set; }
}
```

**Generated UE4:**
```cpp
USTRUCT()
struct FTeam {
    UPROPERTY()
    TArray<FString> MemberNames;

    UPROPERTY()
    TArray<int32> PlayerIds;
};
```

#### Dictionaries

**Syntax:** `dict<K, V>`

**C# Mapping:**
- `Dictionary<K, V>` (default, mutable)
- `IReadOnlyDictionary<K, V>` (with `[csharp dict_implementation=readonly]`)

**UE4 Mapping:**
- `TMap<K, V>` (Unreal associative container)

**Example:**
```igor
record GameState {
    dict<string, int> player_scores;
    dict<int, string> id_to_name;
}
```

**Generated C#:**
```csharp
public class GameState {
    public Dictionary<string, int> PlayerScores { get; set; }
    public Dictionary<int, string> IdToName { get; set; }
}
```

**Generated UE4:**
```cpp
USTRUCT()
struct FGameState {
    UPROPERTY()
    TMap<FString, int32> PlayerScores;

    UPROPERTY()
    TMap<int32, FString> IdToName;
};
```

#### Optional Types

**Syntax:** `?T`

**C# Mapping:**
- Value types: `T?` (nullable struct)
- Reference types: `T?` (nullable reference type, if enabled)

**UE4 Mapping:**
- `TOptional<T>` (Unreal optional wrapper)

**Example:**
```igor
record UserProfile {
    string username;           // Required
    ?string email;             // Optional
    ?int age;                  // Optional
    ?list<string> interests;   // Optional list
}
```

**Generated C#:**
```csharp
public class UserProfile {
    public string Username { get; set; }
    public string? Email { get; set; }      // Nullable reference
    public int? Age { get; set; }           // Nullable value
    public List<string>? Interests { get; set; }
}
```

**Generated UE4:**
```cpp
USTRUCT()
struct FUserProfile {
    UPROPERTY()
    FString Username;

    UPROPERTY()
    TOptional<FString> Email;

    UPROPERTY()
    TOptional<int32> Age;

    UPROPERTY()
    TOptional<TArray<FString>> Interests;
};
```

---

### User-Defined Types

#### Records (Structs/Classes)

Records define data structures with named fields.

**Basic Syntax:**
```igor
record TypeName {
    type1 field1;
    type2 field2;
    ...
}
```

**With Default Values:**
```igor
record GameSettings {
    int max_players = 10;
    float gravity = 9.8;
    bool friendly_fire = false;
    list<string> maps = [];
}
```

**Generic Records:**
```igor
record Container<T> {
    T value;
    int count;
}

record Pair<K, V> {
    K key;
    V value;
}
```

**Nested Records:**
```igor
record Player {
    string name;
    Position position;
}

record Player.Position {
    float x;
    float y;
    float z;
}
```

**C# Generation Options:**

```igor
[csharp class]           // Generate as class (reference type)
[csharp struct]          // Generate as struct (value type)
[csharp sealed]          // Generate sealed class
[csharp partial]         // Generate partial class
[csharp immutable]       // Read-only properties, init-only
```

**Example:**
```igor
[csharp class immutable]
record Employee {
    int employee_id;
    string name;
    string department;
    ?string email;
}
```

**Generated C#:**
```csharp
public class Employee {
    public int EmployeeId { get; init; }
    public string Name { get; init; }
    public string Department { get; init; }
    public string? Email { get; init; }

    public Employee(int employeeId, string name, string department, string? email) {
        EmployeeId = employeeId;
        Name = name;
        Department = department;
        Email = email;
    }
}
```

**UE4 Generation Options:**

```igor
[ue4 ustruct]                  // Generate USTRUCT()
[ue4 blueprint_type]           // BlueprintType
[ue4 uproperty]                // UPROPERTY() on all fields
[ue4 ptr]                      // Use TSharedPtr<const T>
```

**Example:**
```igor
[ue4 ustruct blueprint_type uproperty]
record Enemy {
    string enemy_name;
    float health;
    int damage;
}
```

**Generated UE4:**
```cpp
USTRUCT(BlueprintType)
struct FEnemy {
    GENERATED_BODY()

    UPROPERTY(BlueprintReadWrite)
    FString EnemyName;

    UPROPERTY(BlueprintReadWrite)
    float Health;

    UPROPERTY(BlueprintReadWrite)
    int32 Damage;
};
```

#### Enums

Enums define named constants.

**Basic Syntax:**
```igor
enum EnumName {
    value1;
    value2;
    value3;
}
```

**With Explicit Values:**
```igor
enum Status {
    inactive = 0;
    active = 1;
    suspended = 2;
}
```

**Flags (Bitwise):**
```igor
[flags]
enum Permissions {
    read = 1;
    write = 2;
    execute = 4;
    admin = 8;
}
```

**Custom Integer Type:**
```igor
[int_type=byte]
enum Color {
    red = 1;
    green = 2;
    blue = 3;
}
```

**C# Generation:**

```igor
[csharp namespace="MyApp.Enums"]
enum Priority {
    low = 0;
    medium = 1;
    high = 2;
}
```

**Generated C#:**
```csharp
namespace MyApp.Enums {
    public enum Priority : int {
        Low = 0,
        Medium = 1,
        High = 2
    }
}
```

**UE4 Generation:**

```igor
[ue4 uenum]
enum GameMode {
    singleplayer;
    multiplayer;
    coop;
}
```

**Generated UE4:**
```cpp
UENUM(BlueprintType)
enum class EGameMode : uint8 {
    Singleplayer,
    Multiplayer,
    Coop
};
```

#### Variants (Discriminated Unions)

Variants represent one-of-many types with a tag field.

**Syntax:**
```igor
enum MessageType {
    text;
    image;
    video;
}

variant Message {
    tag MessageType type;
}

record Message.TextMessage[text] {
    string content;
}

record Message.ImageMessage[image] {
    binary image_data;
    string mime_type;
}

record Message.VideoMessage[video] {
    string video_url;
    int duration_seconds;
}
```

**C# Generation:**
- Base class with `Type` property
- Derived classes per variant
- Type-safe casting

**Generated C#:**
```csharp
public abstract class Message {
    public MessageType Type { get; set; }
}

public class TextMessage : Message {
    public string Content { get; set; }
}

public class ImageMessage : Message {
    public byte[] ImageData { get; set; }
    public string MimeType { get; set; }
}

public class VideoMessage : Message {
    public string VideoUrl { get; set; }
    public int DurationSeconds { get; set; }
}
```

**UE4 Generation:**
- Struct with tag and optional fields
- Helper functions for variant access

#### Interfaces (Multiple Inheritance)

```igor
interface Identifiable {
    int id;
}

interface Timestamped {
    long created_at;
    long updated_at;
}

record User : Identifiable, Timestamped {
    string username;
    string email;
}
```

**C# Generation:**
- Interfaces for contracts
- Classes implement interfaces

#### Defines (Type Aliases)

**Simple Alias:**
```igor
define UserId int;
define Email string;
define Timestamp long;
```

**Custom Target Mapping (C#):**
```igor
[csharp alias=System.DateTime]
define DateTime long;  // Stored as long, mapped to System.DateTime in C#
```

**Generated C#:**
```csharp
public class MyRecord {
    public System.DateTime CreatedAt { get; set; }  // Uses DateTime, not long
}
```

---

## Attribute System

### Core Attributes

These attributes work across all target languages.

#### General Attributes

##### `enabled`
Enable or disable code generation for a scope.

**Targets:** Any
**Inheritance:** Scope

```igor
[enabled=false]
module DeprecatedModule {
    // This module won't generate code
}

[enabled]
record MyType {
    // This will generate code
}
```

##### `annotation`
Documentation string attached to declarations.

**Targets:** Any

```igor
[annotation="User authentication data"]
record User {
    [annotation="Unique user identifier"]
    int id;
}
```

---

#### JSON Serialization Attributes

##### `json.enabled`
Enable JSON serialization for a scope.

**Targets:** Any
**Inheritance:** Scope

```igor
[json.enabled]
module MyModule {
    record MyType { ... }  // Will have JSON serializers
}
```

##### `json.ignore`
Exclude field from JSON serialization.

**Targets:** RecordField

```igor
record User {
    string username;
    string password;

    [json.ignore]
    string internal_token;  // Not serialized to JSON
}
```

##### `json.notation`
Set JSON naming convention for type fields.

**Targets:** Type
**Inheritance:** Scope
**Values:** `camel_case`, `pascal_case`, `snake_case`, `kebab_case`

```igor
[json.notation=snake_case]
record UserProfile {
    string firstName;   // Serialized as "first_name"
    string lastName;    // Serialized as "last_name"
}
```

##### `json.field_notation`
Set JSON naming convention for individual field.

**Targets:** RecordField, EnumField
**Inheritance:** Scope

```igor
record Config {
    [json.field_notation=camel_case]
    string api_key;  // Serialized as "apiKey"
}
```

##### `json.key`
Custom JSON key name.

**Targets:** Any

```igor
record User {
    [json.key="userId"]
    int id;  // Serialized as "userId" instead of "id"
}
```

##### `json.nulls`
Include null values in JSON output.

**Targets:** Any
**Inheritance:** Scope

```igor
[json.nulls]
record Data {
    ?string optional_field;  // null will be written to JSON
}
```

##### `json.number`
Serialize enums as numbers instead of strings.

**Targets:** Enum

```igor
[json.number]
enum Status {
    active = 1;
    inactive = 2;
}
// Serialized as: { "status": 1 } instead of { "status": "active" }
```

---

#### Binary Serialization Attributes

##### `binary.enabled`
Enable binary serialization.

**Targets:** Any
**Inheritance:** Scope

```igor
[binary.enabled]
module MyModule {
    record MyType { ... }  // Will have binary serializers
}
```

##### `binary.ignore`
Exclude field from binary serialization.

**Targets:** RecordField

```igor
record GameState {
    int level;

    [binary.ignore]
    string debug_info;  // Not serialized to binary
}
```

##### `binary.header`
Add header information to binary format.

**Targets:** Record

```igor
[binary.header]
record ProtocolMessage {
    int version;
    binary payload;
}
```

---

#### Value Constraints

##### `min` / `max`
Minimum and maximum value constraints.

**Targets:** Any
**Inheritance:** Type

```igor
record PlayerStats {
    [min=0, max=100]
    int health;  // Valid range: 0-100

    [min=1.0, max=10.0]
    float speed;  // Valid range: 1.0-10.0
}
```

##### `min_length` / `max_length`
String or collection length constraints.

**Targets:** Any
**Inheritance:** Type

```igor
record User {
    [min_length=3, max_length=20]
    string username;  // 3-20 characters

    [min_length=1, max_length=10]
    list<string> tags;  // 1-10 items
}
```

##### `not_empty`
Require non-empty strings or collections.

**Targets:** Type, RecordField
**Inheritance:** Type

```igor
record Message {
    [not_empty]
    string content;  // Cannot be empty string

    [not_empty]
    list<string> recipients;  // Cannot be empty list
}
```

---

#### Service Attributes

##### `client`
Generate client-side service code.

**Targets:** Service

```igor
[csharp client]
service MyService {
    c->s GetData() returns (string data);
}
```

##### `server`
Generate server-side service code.

**Targets:** Service

```igor
[csharp server]
service MyService {
    c->s ProcessRequest(string data);
}
```

---

#### HTTP Attributes

##### `http.client` / `http.server`
Enable HTTP client or server generation.

**Targets:** WebService
**Inheritance:** Scope

```igor
[http.client]
webservice MyApi {
    GetData => GET /data -> string;
}
```

##### `http.unfold`
Expand list parameters into multiple query parameters.

**Targets:** Any
**Inheritance:** Type

```igor
// With unfold=true:  /search?tags=a&tags=b&tags=c
// With unfold=false: /search?tags=a,b,c
[http.unfold]
list<string> tags;
```

##### `http.separator`
Custom separator for list expansion.

**Targets:** Any
**Inheritance:** Type

```igor
[http.separator="|"]
list<string> values;  // Serialized as: value1|value2|value3
```

---

### C# Attributes

#### Type Generation

##### `csharp namespace`
Set C# namespace for generated code.

**Targets:** Type, Service, WebService
**Inheritance:** Scope

```igor
[csharp namespace="MyCompany.GameLogic"]
module GameTypes {
    record Player { ... }
}
```

**Generated C#:**
```csharp
namespace MyCompany.GameLogic {
    public class Player { ... }
}
```

##### `csharp name`
Custom C# name for type or field.

**Targets:** Any

```igor
record User {
    [csharp name="EmailAddress"]
    string email;
}
```

**Generated C#:**
```csharp
public class User {
    public string EmailAddress { get; set; }
}
```

##### `csharp field_notation`
C# field naming convention.

**Targets:** Any
**Inheritance:** Scope
**Values:** `camel_case`, `pascal_case`, `snake_case`

```igor
[csharp field_notation=pascal_case]
record Config {
    string api_key;  // Generated as: public string ApiKey { get; set; }
}
```

##### `csharp class`
Generate as C# class (reference type).

**Targets:** Type

```igor
[csharp class]
record Employee {
    int id;
    string name;
}
```

##### `csharp struct`
Generate as C# struct (value type).

**Targets:** Type

```igor
[csharp struct]
record Vector3 {
    float x;
    float y;
    float z;
}
```

##### `csharp sealed`
Generate sealed class.

**Targets:** Type

```igor
[csharp sealed class]
record FinalType {
    int value;
}
```

##### `csharp partial`
Generate partial class.

**Targets:** Type

```igor
[csharp partial class]
record ExtendableType {
    int value;
}
```

**Generated C#:**
```csharp
public partial class ExtendableType {
    public int Value { get; set; }
}
```

---

#### Constructor Attributes

##### `csharp setup_ctor`
Generate constructor with all fields as parameters.

**Targets:** Type
**Inheritance:** Inherited

```igor
[csharp setup_ctor]
record User {
    int id;
    string name;
    string email;
}
```

**Generated C#:**
```csharp
public class User {
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }

    public User(int id, string name, string email) {
        Id = id;
        Name = name;
        Email = email;
    }
}
```

##### `csharp setup_ctor.ignore`
Exclude field from setup constructor.

**Targets:** RecordField

```igor
[csharp setup_ctor]
record User {
    int id;
    string name;

    [csharp setup_ctor.ignore]
    long created_at = 0;  // Not in constructor
}
```

##### `csharp default_ctor`
Generate default (parameterless) constructor.

**Targets:** Type
**Inheritance:** Inherited

```igor
[csharp default_ctor]
record Config {
    int timeout = 30;
}
```

---

#### Equality & Comparison

##### `csharp equals`
Generate Equals() and GetHashCode() overrides.

**Targets:** Type

```igor
[csharp equals]
record Point {
    int x;
    int y;
}
```

**Generated C#:**
```csharp
public class Point {
    public int X { get; set; }
    public int Y { get; set; }

    public override bool Equals(object obj) {
        if (obj is Point other) {
            return X == other.X && Y == other.Y;
        }
        return false;
    }

    public override int GetHashCode() {
        return HashCode.Combine(X, Y);
    }
}
```

##### `csharp equals.ignore`
Exclude field from equality comparison.

**Targets:** RecordField

```igor
[csharp equals]
record User {
    int id;
    string username;

    [csharp equals.ignore]
    long last_login;  // Not compared in Equals()
}
```

##### `csharp equality`
Generate `==` and `!=` operators.

**Targets:** Type

```igor
[csharp equality equals]
record Point {
    int x;
    int y;
}
```

**Generated C#:**
```csharp
public static bool operator ==(Point left, Point right) {
    return Equals(left, right);
}

public static bool operator !=(Point left, Point right) {
    return !Equals(left, right);
}
```

##### `csharp equality_comparer`
Generate IEqualityComparer implementation for enum.

**Targets:** Enum
**Inheritance:** Scope

```igor
[csharp equality_comparer]
enum Status {
    active;
    inactive;
}
```

---

#### Serialization Configuration

##### `csharp alias`
Map Igor type to existing C# type.

**Targets:** Type

```igor
[csharp alias=System.DateTime]
define DateTime long;
```

**Generated C#:**
```csharp
public class Event {
    public System.DateTime Timestamp { get; set; }  // Not long
}
```

##### `csharp binary.namespace`
Namespace for binary serializers.

**Targets:** Type, Service, WebService
**Inheritance:** Scope

```igor
[csharp binary.namespace="MyApp.Serialization"]
module MyTypes {
    record Data { ... }
}
```

##### `csharp binary.serializer`
Custom binary serializer type.

**Targets:** Type

```igor
[csharp binary.serializer=MyCustomSerializer]
record CustomType {
    string value;
}
```

##### `csharp json.namespace`
Namespace for JSON serializers.

**Targets:** Type, Service, WebService
**Inheritance:** Scope

```igor
[csharp json.namespace="MyApp.Json"]
module MyTypes {
    record Data { ... }
}
```

##### `csharp json.serializer`
Custom JSON serializer type.

**Targets:** Type

```igor
[csharp json.serializer=MyJsonSerializer]
record CustomType {
    string value;
}
```

##### `csharp json.serializable`
Generate JSON serialization interfaces.

**Targets:** Type
**Inheritance:** Scope

```igor
[csharp json.serializable]
record User {
    string name;
}
```

---

#### Collection Configuration

##### `csharp list_implementation`
Choose list implementation type.

**Targets:** Any
**Inheritance:** Scope
**Values:** `list` (List<T>), `readonly` (IReadOnlyList<T>)

```igor
[csharp list_implementation=readonly]
record Data {
    list<string> items;  // IReadOnlyList<string>
}
```

##### `csharp dict_implementation`
Choose dictionary implementation type.

**Targets:** Any
**Inheritance:** Scope
**Values:** `dictionary` (Dictionary<K,V>), `readonly` (IReadOnlyDictionary<K,V>)

```igor
[csharp dict_implementation=readonly]
record Data {
    dict<string, int> values;  // IReadOnlyDictionary<string, int>
}
```

##### `csharp immutable`
Generate immutable types (init-only properties).

**Targets:** Any
**Inheritance:** Inherited

```igor
[csharp immutable class]
record User {
    int id;
    string name;
}
```

**Generated C#:**
```csharp
public class User {
    public int Id { get; init; }
    public string Name { get; init; }
}
```

##### `csharp readonly`
Generate readonly fields.

**Targets:** RecordField
**Inheritance:** Scope

```igor
record Config {
    [csharp readonly]
    int version = 1;
}
```

##### `csharp nullable`
Enable nullable reference types.

**Targets:** Any
**Inheritance:** Scope

```igor
[csharp nullable]
module MyModule {
    record User {
        string name;    // string (non-nullable)
        ?string email;  // string? (nullable)
    }
}
```

---

#### Framework Configuration

##### `csharp target_framework`
Specify .NET framework version.

**Targets:** Any
**Inheritance:** Scope
**Values:** `net45`, `net46`, `netstandard2.0`, `net6.0`, etc.

```igor
[csharp target_framework=net6.0]
module MyModule { ... }
```

##### `csharp tpl`
Enable Task Parallel Library (async/await).

**Targets:** Any
**Inheritance:** Scope

```igor
[csharp tpl]
service MyService {
    c->s GetDataAsync() returns (string data);
}
```

---

### UE4 Attributes

#### Naming & Organization

##### `ue4 name`
Custom C++ name for type or field.

**Targets:** Any

```igor
record Player {
    [ue4 name="PlayerId"]
    int id;
}
```

**Generated UE4:**
```cpp
struct FPlayer {
    int32 PlayerId;
};
```

##### `ue4 namespace`
C++ namespace for generated code.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 namespace="MyGame::Data"]
module GameTypes {
    record Player { ... }
}
```

**Generated UE4:**
```cpp
namespace MyGame {
namespace Data {
    struct FPlayer { ... };
}}
```

##### `ue4 prefix`
Name prefix for generated types.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 prefix="Game"]
record Player {
    string name;
}
```

**Generated UE4:**
```cpp
struct FGamePlayer {  // "Game" prefix added
    FString Name;
};
```

##### `ue4 category`
Unreal Editor category for Blueprint types.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 category="MyGame|Characters"]
record Player {
    string name;
}
```

**Generated UE4:**
```cpp
USTRUCT(Category="MyGame|Characters")
struct FPlayer { ... };
```

##### `ue4 alias`
Type alias for defines.

**Targets:** Any

```igor
[ue4 alias=FVector]
define Position float;
```

##### `ue4 base_type`
Custom base class.

**Targets:** Any

```igor
[ue4 base_type=UObject]
record MyActor {
    string name;
}
```

---

#### File Organization

##### `ue4 h_file`
Custom header filename.

**Targets:** Any

```igor
[ue4 h_file="GameTypes.h"]
record Player { ... }
```

##### `ue4 cpp_file`
Custom cpp filename.

**Targets:** Any

```igor
[ue4 cpp_file="GameTypes.cpp"]
record Player { ... }
```

##### `ue4 h_path`
Include path for header files.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 h_path="Public/Types"]
module GameTypes { ... }
```

##### `ue4 cpp_path`
Include path for cpp files.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 cpp_path="Private/Types"]
module GameTypes { ... }
```

##### `ue4 h_include`
Additional header includes.

**Targets:** Module

```igor
[ue4 h_include="CoreMinimal.h"]
[ue4 h_include="GameFramework/Actor.h"]
module GameTypes { ... }
```

**Generated UE4:**
```cpp
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
```

##### `ue4 cpp_include`
Additional cpp includes.

**Targets:** Module

```igor
[ue4 cpp_include="IgorRuntime.h"]
module GameTypes { ... }
```

##### `ue4 igor_path`
Include path for Igor runtime headers.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 igor_path="IgorRuntime/Public"]
module GameTypes { ... }
```

##### `ue4 api_macro`
Export macro (e.g., MYMODULE_API).

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 api_macro="MYGAME_API"]
module GameTypes { ... }
```

**Generated UE4:**
```cpp
struct MYGAME_API FPlayer { ... };
```

---

#### Unreal Reflection System

##### `ue4 uenum`
Generate UENUM() macro for enum reflection.

**Targets:** Enum
**Inheritance:** Scope

```igor
[ue4 uenum]
enum GameMode {
    singleplayer;
    multiplayer;
}
```

**Generated UE4:**
```cpp
UENUM(BlueprintType)
enum class EGameMode : uint8 {
    Singleplayer,
    Multiplayer
};
```

##### `ue4 ustruct`
Generate USTRUCT() macro for struct reflection.

**Targets:** Type
**Inheritance:** Inherited

```igor
[ue4 ustruct]
record Player {
    string name;
    int health;
}
```

**Generated UE4:**
```cpp
USTRUCT(BlueprintType)
struct FPlayer {
    GENERATED_BODY()

    FString Name;
    int32 Health;
};
```

##### `ue4 uclass`
Generate UCLASS() macro for UObject-derived class.

**Targets:** Type
**Inheritance:** Inherited

```igor
[ue4 uclass]
record GameInstance {
    int player_count;
}
```

**Generated UE4:**
```cpp
UCLASS()
class UGameInstance : public UObject {
    GENERATED_BODY()

    int32 PlayerCount;
};
```

##### `ue4 uproperty`
Generate UPROPERTY() macro for all fields.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Item {
    string item_name;
    int quantity;
}
```

**Generated UE4:**
```cpp
USTRUCT()
struct FItem {
    GENERATED_BODY()

    UPROPERTY()
    FString ItemName;

    UPROPERTY()
    int32 Quantity;
};
```

---

#### Blueprint Integration

##### `ue4 blueprint_type`
Mark struct as BlueprintType.

**Targets:** Record

```igor
[ue4 ustruct blueprint_type]
record Weapon {
    string name;
    int damage;
}
```

**Generated UE4:**
```cpp
USTRUCT(BlueprintType)
struct FWeapon { ... };
```

##### `ue4 blueprint_read_write`
UPROPERTY with BlueprintReadWrite.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Character {
    [ue4 blueprint_read_write]
    string character_name;
}
```

**Generated UE4:**
```cpp
UPROPERTY(BlueprintReadWrite)
FString CharacterName;
```

##### `ue4 blueprint_read_only`
UPROPERTY with BlueprintReadOnly.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Character {
    [ue4 blueprint_read_only]
    int character_id;
}
```

**Generated UE4:**
```cpp
UPROPERTY(BlueprintReadOnly)
int32 CharacterId;
```

##### `ue4 edit_anywhere`
UPROPERTY with EditAnywhere.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Settings {
    [ue4 edit_anywhere]
    float volume;
}
```

**Generated UE4:**
```cpp
UPROPERTY(EditAnywhere)
float Volume;
```

##### `ue4 edit_defaults_only`
UPROPERTY with EditDefaultsOnly.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Config {
    [ue4 edit_defaults_only]
    int max_connections;
}
```

**Generated UE4:**
```cpp
UPROPERTY(EditDefaultsOnly)
int32 MaxConnections;
```

##### `ue4 visible_anywhere`
UPROPERTY with VisibleAnywhere.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Stats {
    [ue4 visible_anywhere]
    int total_kills;
}
```

**Generated UE4:**
```cpp
UPROPERTY(VisibleAnywhere)
int32 TotalKills;
```

##### `ue4 visible_defaults_only`
UPROPERTY with VisibleDefaultsOnly.

**Targets:** RecordField
**Inheritance:** Scope

```igor
[ue4 ustruct uproperty]
record Info {
    [ue4 visible_defaults_only]
    string build_version;
}
```

**Generated UE4:**
```cpp
UPROPERTY(VisibleDefaultsOnly)
FString BuildVersion;
```

---

#### Memory Management

##### `ue4 ptr`
Use TSharedPtr instead of direct value.

**Targets:** Any

```igor
[ue4 ptr]
record LargeData {
    binary content;
}
```

**Generated UE4:**
```cpp
// Field usage:
TSharedPtr<const FLargeData> Data;
```

##### `ue4 typedef`
Generate as typedef instead of struct.

**Targets:** Type

```igor
[ue4 typedef]
define PlayerId int;
```

**Generated UE4:**
```cpp
typedef int32 FPlayerId;
```

---

#### Serialization

##### `ue4 json.custom_serializer`
Use custom JSON serialization.

**Targets:** Any

```igor
[ue4 json.custom_serializer]
record CustomData {
    string value;
}
```

##### `ue4 ignore`
Skip field in code generation.

**Targets:** RecordField

```igor
record Data {
    string value;

    [ue4 ignore]
    int temp_field;  // Not generated in UE4
}
```

---

#### HTTP Client Configuration

##### `ue4 http.client.lazy`
Lazy-initialize HTTP client.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 http.client.lazy]
webservice MyApi { ... }
```

##### `ue4 http.client.path_setup`
Setup path parameters from Args or Request.

**Targets:** Any
**Inheritance:** Scope
**Values:** `args`, `request`

```igor
[ue4 http.client.path_setup=args]
webservice MyApi { ... }
```

##### `ue4 http.client.query_setup`
Setup query parameters from Args or Request.

**Targets:** Any
**Inheritance:** Scope
**Values:** `args`, `request`

```igor
[ue4 http.client.query_setup=request]
webservice MyApi { ... }
```

##### `ue4 http.client.header_setup`
Setup headers from Args or Request.

**Targets:** Any
**Inheritance:** Scope
**Values:** `args`, `request`

```igor
[ue4 http.client.header_setup=request]
webservice MyApi { ... }
```

##### `ue4 http.client.content_setup`
Setup request content from Args or Request.

**Targets:** Any
**Inheritance:** Scope
**Values:** `args`, `request`

```igor
[ue4 http.client.content_setup=args]
webservice MyApi { ... }
```

##### `ue4 http.base_url`
Default base URL for HTTP client.

```igor
[ue4 http.base_url="https://api.example.com"]
webservice MyApi { ... }
```

---

#### Other Features

##### `ue4 log_category`
Unreal logging category.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 log_category="LogGameTypes"]
module GameTypes { ... }
```

##### `ue4 interfaces`
Generate interface support.

**Targets:** Any
**Inheritance:** Scope

```igor
[ue4 interfaces]
interface Serializable {
    int version;
}
```

##### `ue4 meta`
JSON metadata attachment.

**Targets:** Any

```igor
[ue4 meta={"tooltip": "Player health value"}]
int health;
```

**Generated UE4:**
```cpp
UPROPERTY(meta=(tooltip="Player health value"))
int32 Health;
```

---

## Service & RPC System

### Service Declaration

Services define bidirectional RPC communication between client and server.

**Basic Syntax:**
```igor
service ServiceName {
    c->s FunctionName(arg1_type arg1, arg2_type arg2, ...);
    s->c FunctionName(arg1_type arg1, ...);
}
```

**Direction Syntax:**
- `c->s`: Client-to-Server (client calls server)
- `s->c`: Server-to-Client (server pushes to client)

---

### RPC Types

#### Fire-and-Forget

No return value, no response expected.

**Igor:**
```igor
service ChatService {
    c->s SendMessage(string message);
    s->c NotifyUserJoined(string username);
}
```

**Generated C#:**
```csharp
public interface IChatServiceClientToServer {
    Task SendMessage(string message);
}

public interface IChatServiceServerToClient {
    Task NotifyUserJoined(string username);
}
```

**Generated UE4:**
```cpp
class IChatServiceClientToServer {
    virtual void SendMessage(const FString& InMessage) = 0;
};

class IChatServiceServerToClient {
    virtual void NotifyUserJoined(const FString& InUsername) = 0;
};
```

---

#### Request-Response RPC

With return values, expects response.

**Igor:**
```igor
service DataService {
    c->s GetUser(int user_id) returns (string username, int level);
}
```

**Generated C# (with result class):**
```csharp
public interface IDataServiceClientToServer {
    Task<DataService.GetUserResult> GetUser(int userId);
}

public static class DataService {
    public class GetUserResult {
        public string Username { get; set; }
        public int Level { get; set; }
    }
}
```

**Generated UE4 (with result struct):**
```cpp
struct FGetUserResultStruct {
    FString Username;
    int32 Level;
};

class IDataServiceClientToServer {
    virtual TIgorRpcResponse<FGetUserResultStruct> GetUser(
        const int32& InUserId) = 0;
};
```

---

#### RPC with Exceptions

**Igor:**
```igor
exception NotFoundException {
    string message;
}

exception UnauthorizedException {
    int error_code;
}

service DataService {
    c->s GetUserData(int user_id)
        returns (string data)
        throws NotFoundException, UnauthorizedException;
}
```

**Generated C# (exception handling):**
```csharp
public interface IDataServiceClientToServer {
    Task<DataService.GetUserDataResult> GetUserData(int userId);
}

public class NotFoundException : Exception {
    public string Message { get; set; }
}

public class UnauthorizedException : Exception {
    public int ErrorCode { get; set; }
}
```

**Generated UE4 (exception in response type):**
```cpp
struct FNotFoundException {
    FString Message;
};

struct FUnauthorizedException {
    int32 ErrorCode;
};

using TGetUserDataResponse = TIgorRpcResponse<
    FGetUserDataResultStruct,
    FNotFoundException,
    FUnauthorizedException>;

class IDataServiceClientToServer {
    virtual TGetUserDataResponse GetUserData(const int32& InUserId) = 0;
};
```

---

### Service Configuration

**C# Client Service:**
```igor
[csharp client namespace="MyApp.Services"]
service MyService {
    c->s DoAction(string data);
}
```

**C# Server Service:**
```igor
[csharp server namespace="MyApp.Services"]
service MyService {
    c->s DoAction(string data);
}
```

**UE4 Client Service:**
```igor
[ue4 client namespace="MyGame::Services"]
service MyService {
    c->s DoAction(string data);
}
```

---

### Service Messages

Generate separate message classes for RPC with `[csharp messages]`:

**Igor:**
```igor
[csharp messages]
service MyService {
    c->s GetData(int id) returns (string data, int count);
}
```

**Generated C# (additional message classes):**
```csharp
public static class MyService {
    // Request message
    public class GetDataRequest {
        public int Id { get; set; }
    }

    // Response message
    public class GetDataResponse {
        public string Data { get; set; }
        public int Count { get; set; }
    }

    // Result (same as before)
    public class GetDataResult {
        public string Data { get; set; }
        public int Count { get; set; }
    }
}
```

---

## Web Services

Web services generate HTTP/REST API clients.

### WebService Syntax

**Basic Structure:**
```igor
webservice ServiceName {
    FunctionName => METHOD /path -> ResponseType;
    FunctionName => METHOD /path <- RequestType -> ResponseType;
    FunctionName => METHOD /path?query={type param} -> ResponseType;
}
```

**HTTP Methods:**
- `GET` - Read data
- `POST` - Create data
- `PUT` - Update data
- `PATCH` - Partial update
- `DELETE` - Delete data
- `HEAD` - Headers only
- `OPTIONS` - Allowed methods

---

### Path Parameters

**Syntax:**
```igor
webservice UserApi {
    GetUser => GET /users/{int id} -> User;
    GetPost => GET /users/{int user_id}/posts/{int post_id} -> Post;
}
```

**Generated C#:**
```csharp
public class UserApi {
    public async Task<User> GetUser(int id, CancellationToken cancellationToken) {
        var url = $"/users/{id}";
        // HTTP GET request...
    }

    public async Task<Post> GetPost(int userId, int postId,
        CancellationToken cancellationToken) {
        var url = $"/users/{userId}/posts/{postId}";
        // HTTP GET request...
    }
}
```

**Generated UE4:**
```cpp
class FUserApi : public FIgorHttpClient {
    TSharedRef<FGetUserRequest> GetUser(const int32& InId);
    TSharedRef<FGetPostRequest> GetPost(const int32& InUserId, const int32& InPostId);
};
```

---

### Query Parameters

**Syntax:**
```igor
webservice SearchApi {
    Search => GET /search?q={string query}&limit={int limit} -> SearchResults;
}
```

**Generated C#:**
```csharp
public class SearchApi {
    public async Task<SearchResults> Search(string query, int limit,
        CancellationToken cancellationToken) {
        var url = $"/search?q={Uri.EscapeDataString(query)}&limit={limit}";
        // HTTP GET request...
    }
}
```

**With Lists:**
```igor
webservice FilterApi {
    Filter => GET /items?tags={list<string> tags} -> list<Item>;
}
```

**Generated URL (with unfold):**
```
/items?tags=tag1&tags=tag2&tags=tag3
```

**Generated URL (without unfold):**
```
/items?tags=tag1,tag2,tag3
```

---

### Request Body

**Syntax:**
```igor
webservice UserApi {
    CreateUser => POST /users <- CreateUserRequest -> User;
    UpdateUser => PUT /users/{int id} <- UpdateUserRequest -> User;
}
```

**Generated C#:**
```csharp
public class UserApi {
    public async Task<User> CreateUser(CreateUserRequest request,
        CancellationToken cancellationToken) {
        var url = "/users";
        var json = JsonSerializer.Serialize(request);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        // HTTP POST request...
    }

    public async Task<User> UpdateUser(int id, UpdateUserRequest request,
        CancellationToken cancellationToken) {
        var url = $"/users/{id}";
        var json = JsonSerializer.Serialize(request);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        // HTTP PUT request...
    }
}
```

**Generated UE4:**
```cpp
class FUserApi : public FIgorHttpClient {
    TSharedRef<FCreateUserRequest> CreateUser(const FCreateUserRequest& InRequest);
    TSharedRef<FUpdateUserRequest> UpdateUser(const int32& InId,
        const FUpdateUserRequest& InRequest);
};
```

---

### Headers

**Custom Headers (via attributes):**
```igor
[http.header="Authorization"]
string auth_token;
```

---

### Complete WebService Example

**Igor:**
```igor
[csharp http.client namespace="MyApp.Api"]
[ue4 http.client namespace="MyGame::Api"]
webservice TodoApi {
    // GET all todos
    GetTodos => GET /todos -> list<Todo>;

    // GET single todo by ID
    GetTodo => GET /todos/{int id} -> Todo;

    // POST create new todo
    CreateTodo => POST /todos <- CreateTodoRequest -> Todo;

    // PUT update todo
    UpdateTodo => PUT /todos/{int id} <- UpdateTodoRequest -> Todo;

    // DELETE todo
    DeleteTodo => DELETE /todos/{int id} -> bool;

    // GET with query parameters
    SearchTodos => GET /todos/search?q={string query}&completed={bool completed}
        -> list<Todo>;
}

record Todo {
    int id;
    string title;
    bool completed;
}

record CreateTodoRequest {
    string title;
}

record UpdateTodoRequest {
    string title;
    bool completed;
}
```

**Generated C#:**
```csharp
namespace MyApp.Api {
    public class TodoApi : IDisposable {
        private readonly HttpClient _httpClient;

        public TodoApi(HttpClient httpClient) {
            _httpClient = httpClient;
        }

        public async Task<List<Todo>> GetTodos(CancellationToken cancellationToken) {
            var response = await _httpClient.GetAsync("/todos", cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<Todo>>(json);
        }

        public async Task<Todo> GetTodo(int id, CancellationToken cancellationToken) {
            var response = await _httpClient.GetAsync($"/todos/{id}", cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<Todo>(json);
        }

        public async Task<Todo> CreateTodo(CreateTodoRequest request,
            CancellationToken cancellationToken) {
            var json = JsonSerializer.Serialize(request);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            var response = await _httpClient.PostAsync("/todos", content, cancellationToken);
            response.EnsureSuccessStatusCode();
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<Todo>(responseJson);
        }

        public async Task<Todo> UpdateTodo(int id, UpdateTodoRequest request,
            CancellationToken cancellationToken) {
            var json = JsonSerializer.Serialize(request);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            var response = await _httpClient.PutAsync($"/todos/{id}", content,
                cancellationToken);
            response.EnsureSuccessStatusCode();
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<Todo>(responseJson);
        }

        public async Task<bool> DeleteTodo(int id, CancellationToken cancellationToken) {
            var response = await _httpClient.DeleteAsync($"/todos/{id}", cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<bool>(json);
        }

        public async Task<List<Todo>> SearchTodos(string query, bool completed,
            CancellationToken cancellationToken) {
            var url = $"/todos/search?q={Uri.EscapeDataString(query)}&completed={completed}";
            var response = await _httpClient.GetAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<Todo>>(json);
        }

        public void Dispose() {
            _httpClient?.Dispose();
        }
    }
}
```

**Generated UE4:**
```cpp
namespace MyGame {
namespace Api {

class FTodoApi : public FIgorHttpClient {
public:
    TSharedRef<FGetTodosRequest> GetTodos();
    TSharedRef<FGetTodoRequest> GetTodo(const int32& InId);
    TSharedRef<FCreateTodoRequest> CreateTodo(const FCreateTodoRequest& InRequest);
    TSharedRef<FUpdateTodoRequest> UpdateTodo(const int32& InId,
        const FUpdateTodoRequest& InRequest);
    TSharedRef<FDeleteTodoRequest> DeleteTodo(const int32& InId);
    TSharedRef<FSearchTodosRequest> SearchTodos(const FString& InQuery,
        const bool& InCompleted);
};

}}  // namespace MyGame::Api
```

---

## Serialization

### Binary Serialization

**Enable Binary:**
```igor
[binary.enabled]
module MyModule {
    record Data { ... }
}
```

**C# Binary Serialization:**
```csharp
// Serialize
var bytes = IgorSerializer.Serialize(myData);

// Deserialize
var data = IgorSerializer.Deserialize<Data>(bytes);
```

**UE4 Binary Serialization:**
```cpp
// Serialize
FIgorWriter Writer;
IgorWriteBinary(Writer, MyData);
TArray<uint8> Bytes = Writer.GetBytes();

// Deserialize
FIgorReader Reader(Bytes);
FData MyData;
IgorReadBinary(Reader, MyData);
```

---

### JSON Serialization

**Enable JSON:**
```igor
[json.enabled]
module MyModule {
    record Data { ... }
}
```

**C# JSON Serialization:**
```csharp
// Serialize
var json = JsonSerializer.Serialize(myData);

// Deserialize
var data = JsonSerializer.Deserialize<Data>(json);
```

**UE4 JSON Serialization:**
```cpp
// Serialize
FString JsonString;
IgorWriteJson(JsonString, MyData);

// Deserialize
FData MyData;
IgorReadJson(JsonString, MyData);
```

---

### String Serialization

Used for URI/query parameters.

**C# String Serialization:**
```csharp
var str = StringSerializer.Serialize(value);
var value = StringSerializer.Deserialize<T>(str);
```

**UE4 String Serialization:**
```cpp
FString Str = VarToString(Value);
```

---

### Custom Serializers

**C# Custom Serializer:**
```igor
[csharp json.serializer=MyCustomSerializer]
record CustomType {
    string value;
}
```

**UE4 Custom Serializer:**
```igor
[ue4 json.custom_serializer]
record CustomType {
    string value;
}
```

---

## Complete Examples

### Example 1: Game Player Data (CouchDB)

**Igor:**
```igor
using ProtocolShared;

[schema enabled]
[json.enabled json.notation=snake_case]
[csharp namespace="Game.Data"]
[ue4 ustruct uproperty namespace="GameData"]
module GamePlayer {

    define PlayerId string;
    define ItemId string;
    define Cash float;

    [ue4 uenum]
    enum AccountRole {
        user = 0;
        admin = 1;
        moderator = 2;
    }

    [csharp class setup_ctor]
    [ue4 ustruct]
    record WalletData {
        Cash rubles;
        Cash dollars;
        Cash euros;
    }

    [csharp class setup_ctor]
    [ue4 ustruct]
    record ItemData {
        ItemId item_id;
        string item_name;
        int quantity;
        [min=0.0]
        float durability;
    }

    [csharp class setup_ctor equals]
    [ue4 ustruct blueprint_type]
    record PlayerData {
        PlayerId player_id;
        string player_name;
        AccountRole role;
        long account_created_unix;
        long account_updated_unix;

        WalletData wallet;

        list<ItemData> items;

        [min_length=0, max_length=100]
        dict<string, int> stats;
    }
}
```

**Generated C#:**
```csharp
namespace Game.Data {
    public class WalletData {
        public float Rubles { get; init; }
        public float Dollars { get; init; }
        public float Euros { get; init; }

        public WalletData(float rubles, float dollars, float euros) {
            Rubles = rubles;
            Dollars = dollars;
            Euros = euros;
        }
    }

    public class ItemData {
        public string ItemId { get; init; }
        public string ItemName { get; init; }
        public int Quantity { get; init; }
        public float Durability { get; init; }

        public ItemData(string itemId, string itemName, int quantity, float durability) {
            ItemId = itemId;
            ItemName = itemName;
            Quantity = quantity;
            Durability = durability;
        }
    }

    public class PlayerData {
        public string PlayerId { get; init; }
        public string PlayerName { get; init; }
        public AccountRole Role { get; init; }
        public long AccountCreatedUnix { get; init; }
        public long AccountUpdatedUnix { get; init; }
        public WalletData Wallet { get; init; }
        public List<ItemData> Items { get; init; }
        public Dictionary<string, int> Stats { get; init; }

        public override bool Equals(object obj) {
            // Equality comparison implementation
        }

        public override int GetHashCode() {
            // Hash code implementation
        }
    }
}
```

**Generated UE4:**
```cpp
namespace GameData {

UENUM(BlueprintType)
enum class EAccountRole : uint8 {
    User = 0,
    Admin = 1,
    Moderator = 2
};

USTRUCT()
struct FWalletData {
    GENERATED_BODY()

    UPROPERTY()
    float Rubles;

    UPROPERTY()
    float Dollars;

    UPROPERTY()
    float Euros;
};

USTRUCT()
struct FItemData {
    GENERATED_BODY()

    UPROPERTY()
    FString ItemId;

    UPROPERTY()
    FString ItemName;

    UPROPERTY()
    int32 Quantity;

    UPROPERTY()
    float Durability;
};

USTRUCT(BlueprintType)
struct FPlayerData {
    GENERATED_BODY()

    UPROPERTY()
    FString PlayerId;

    UPROPERTY()
    FString PlayerName;

    UPROPERTY()
    EAccountRole Role;

    UPROPERTY()
    int64 AccountCreatedUnix;

    UPROPERTY()
    int64 AccountUpdatedUnix;

    UPROPERTY()
    FWalletData Wallet;

    UPROPERTY()
    TArray<FItemData> Items;

    UPROPERTY()
    TMap<FString, int32> Stats;
};

}  // namespace GameData
```

---

### Example 2: Client-Server Game Service

**Igor:**
```igor
[csharp namespace="Game.Services" client]
[ue4 namespace="GameServices" client]
module GameService {

    record Vector3 {
        float x;
        float y;
        float z;
    }

    record PlayerState {
        int player_id;
        Vector3 position;
        Vector3 velocity;
        float health;
    }

    exception GameException {
        int error_code;
        string message;
    }

    [csharp client]
    [ue4 client]
    service GameplayService {
        // Client -> Server RPCs
        c->s UpdatePosition(int player_id, Vector3 position, Vector3 velocity);
        c->s Attack(int target_id) returns (bool hit, float damage) throws GameException;
        c->s UseItem(int item_id) returns (bool success) throws GameException;

        // Server -> Client pushes
        s->c NotifyPlayerJoined(int player_id, string player_name);
        s->c NotifyPlayerLeft(int player_id);
        s->c UpdatePlayerState(PlayerState state);
        s->c NotifyDamage(int player_id, float damage);
    }
}
```

**Generated C# Service Interface:**
```csharp
namespace Game.Services {

    // Client-to-Server interface (client implements sender)
    public interface IGameplayServiceClientToServer {
        Task UpdatePosition(int playerId, Vector3 position, Vector3 velocity);
        Task<GameplayService.AttackResult> Attack(int targetId);
        Task<GameplayService.UseItemResult> UseItem(int itemId);
    }

    // Server-to-Client interface (client implements receiver)
    public interface IGameplayServiceServerToClient {
        Task NotifyPlayerJoined(int playerId, string playerName);
        Task NotifyPlayerLeft(int playerId);
        Task UpdatePlayerState(PlayerState state);
        Task NotifyDamage(int playerId, float damage);
    }

    // Result classes
    public static class GameplayService {
        public class AttackResult {
            public bool Hit { get; set; }
            public float Damage { get; set; }
        }

        public class UseItemResult {
            public bool Success { get; set; }
        }
    }

    // Exception
    public class GameException : Exception {
        public int ErrorCode { get; set; }
        public string Message { get; set; }
    }
}
```

**Generated UE4 Service Interface:**
```cpp
namespace GameServices {

struct FAttackResultStruct {
    bool Hit;
    float Damage;
};

struct FUseItemResultStruct {
    bool Success;
};

struct FGameException {
    int32 ErrorCode;
    FString Message;
};

// Client-to-Server interface
class IGameplayServiceClientToServer {
public:
    virtual void UpdatePosition(const int32& InPlayerId, const FVector3& InPosition,
        const FVector3& InVelocity) = 0;

    virtual TIgorRpcResponse<FAttackResultStruct, FGameException> Attack(
        const int32& InTargetId) = 0;

    virtual TIgorRpcResponse<FUseItemResultStruct, FGameException> UseItem(
        const int32& InItemId) = 0;
};

// Server-to-Client interface
class IGameplayServiceServerToClient {
public:
    virtual void NotifyPlayerJoined(const int32& InPlayerId,
        const FString& InPlayerName) = 0;
    virtual void NotifyPlayerLeft(const int32& InPlayerId) = 0;
    virtual void UpdatePlayerState(const FPlayerState& InState) = 0;
    virtual void NotifyDamage(const int32& InPlayerId, const float& InDamage) = 0;
};

}  // namespace GameServices
```

---

### Example 3: REST API Client

**Igor:**
```igor
[csharp http.client namespace="Api.GitHub"]
[ue4 http.client namespace="Api::GitHub"]
module GitHubApi {

    record User {
        int id;
        string login;
        string name;
        ?string email;
        int public_repos;
    }

    record Repository {
        int id;
        string name;
        string full_name;
        bool private;
        User owner;
        string description;
    }

    record CreateRepoRequest {
        string name;
        ?string description;
        bool private = false;
    }

    [csharp http.client]
    [ue4 http.client]
    webservice GitHub {
        // Get user by username
        GetUser => GET /users/{string username} -> User;

        // Get authenticated user
        GetAuthUser => GET /user -> User;

        // List user repositories
        ListRepos => GET /users/{string username}/repos -> list<Repository>;

        // Create repository
        CreateRepo => POST /user/repos <- CreateRepoRequest -> Repository;

        // Update repository
        UpdateRepo => PATCH /repos/{string owner}/{string repo}
            <- CreateRepoRequest -> Repository;

        // Delete repository
        DeleteRepo => DELETE /repos/{string owner}/{string repo} -> bool;

        // Search repositories
        SearchRepos => GET /search/repositories?q={string query}&sort={string sort}
            -> list<Repository>;
    }
}
```

**Generated C# Client:**
```csharp
namespace Api.GitHub {

    public class GitHub : IDisposable {
        private readonly HttpClient _httpClient;

        public GitHub(HttpClient httpClient) {
            _httpClient = httpClient;
        }

        public async Task<User> GetUser(string username,
            CancellationToken cancellationToken) {
            var url = $"/users/{Uri.EscapeDataString(username)}";
            var response = await _httpClient.GetAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<User>(json);
        }

        public async Task<User> GetAuthUser(CancellationToken cancellationToken) {
            var response = await _httpClient.GetAsync("/user", cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<User>(json);
        }

        public async Task<List<Repository>> ListRepos(string username,
            CancellationToken cancellationToken) {
            var url = $"/users/{Uri.EscapeDataString(username)}/repos";
            var response = await _httpClient.GetAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<Repository>>(json);
        }

        public async Task<Repository> CreateRepo(CreateRepoRequest request,
            CancellationToken cancellationToken) {
            var json = JsonSerializer.Serialize(request);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            var response = await _httpClient.PostAsync("/user/repos", content,
                cancellationToken);
            response.EnsureSuccessStatusCode();
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<Repository>(responseJson);
        }

        public async Task<Repository> UpdateRepo(string owner, string repo,
            CreateRepoRequest request, CancellationToken cancellationToken) {
            var url = $"/repos/{Uri.EscapeDataString(owner)}/{Uri.EscapeDataString(repo)}";
            var json = JsonSerializer.Serialize(request);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            var response = await _httpClient.PatchAsync(url, content, cancellationToken);
            response.EnsureSuccessStatusCode();
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<Repository>(responseJson);
        }

        public async Task<bool> DeleteRepo(string owner, string repo,
            CancellationToken cancellationToken) {
            var url = $"/repos/{Uri.EscapeDataString(owner)}/{Uri.EscapeDataString(repo)}";
            var response = await _httpClient.DeleteAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            return response.StatusCode == System.Net.HttpStatusCode.NoContent;
        }

        public async Task<List<Repository>> SearchRepos(string query, string sort,
            CancellationToken cancellationToken) {
            var url = $"/search/repositories?q={Uri.EscapeDataString(query)}" +
                      $"&sort={Uri.EscapeDataString(sort)}";
            var response = await _httpClient.GetAsync(url, cancellationToken);
            response.EnsureSuccessStatusCode();
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<Repository>>(json);
        }

        public void Dispose() {
            _httpClient?.Dispose();
        }
    }
}
```

**Usage in C#:**
```csharp
using var httpClient = new HttpClient();
httpClient.BaseAddress = new Uri("https://api.github.com");
httpClient.DefaultRequestHeaders.Add("User-Agent", "MyApp");
httpClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", "your-token");

using var api = new GitHub(httpClient);

// Get user
var user = await api.GetUser("octocat", CancellationToken.None);
Console.WriteLine($"User: {user.Login}, Repos: {user.PublicRepos}");

// List repos
var repos = await api.ListRepos("octocat", CancellationToken.None);
foreach (var repo in repos) {
    Console.WriteLine($"- {repo.Name}: {repo.Description}");
}

// Create repo
var createRequest = new CreateRepoRequest {
    Name = "my-new-repo",
    Description = "Created with Igor",
    Private = false
};
var newRepo = await api.CreateRepo(createRequest, CancellationToken.None);
Console.WriteLine($"Created: {newRepo.FullName}");
```

---

## Best Practices

### Naming Conventions

1. **Types**: Use PascalCase for records, enums
   ```igor
   record PlayerData { ... }
   enum GameMode { ... }
   ```

2. **Fields**: Use snake_case in Igor (converted by notation attributes)
   ```igor
   record User {
       string user_name;  // C#: UserName, JSON: user_name
   }
   ```

3. **Services**: Use descriptive names
   ```igor
   service AuthenticationService { ... }
   webservice UserManagementApi { ... }
   ```

---

### Attribute Inheritance

Use scope inheritance to avoid repetition:

```igor
[csharp namespace="MyApp.Types"]
[csharp json.enabled]
[csharp field_notation=pascal_case]
[ue4 ustruct uproperty]
module MyTypes {
    // All types inherit above attributes
    record Type1 { ... }
    record Type2 { ... }
}
```

---

### Optional vs Required Fields

Be explicit about optionality:

```igor
record User {
    int id;              // Required
    string username;     // Required
    ?string email;       // Optional (can be null)
    ?int age;            // Optional (can be null)
}
```

---

### Default Values

Use default values for optional configuration:

```igor
record GameConfig {
    int max_players = 10;
    float time_limit = 300.0;
    bool friendly_fire = false;
    string game_mode = "deathmatch";
}
```

---

### Constraints

Add validation constraints:

```igor
record UserProfile {
    [min_length=3, max_length=20]
    string username;

    [min=13, max=120]
    int age;

    [not_empty]
    string email;
}
```

---

### Module Organization

Organize types by domain:

```igor
module PlayerTypes { ... }
module GameplayTypes { ... }
module InventoryTypes { ... }
```

Use `using` to reference other modules:

```igor
using PlayerTypes;

module InventoryTypes {
    record Inventory {
        PlayerTypes.PlayerId player_id;
        list<Item> items;
    }
}
```

---

## Compilation

### Command Line

```bash
igorc.exe --input MyTypes.igor --target csharp --output ./generated/csharp
igorc.exe --input MyTypes.igor --target ue4 --output ./generated/ue4
```

### Multiple Targets

```bash
igorc.exe --input MyTypes.igor --target csharp,ue4 --output ./generated
```

---

## References

- Official Igor Repository: https://github.com/igorlang/igor
- Documentation: https://igor-language.readthedocs.io/
- Latest Release: https://github.com/igorlang/igor/releases/latest

---

**Document Version:** 1.0
**Last Updated:** February 6, 2026
