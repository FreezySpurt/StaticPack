<p align="center">
  <a href="./README.md"><img src="https://img.shields.io/badge/EN-English-blue?style=flat-square" alt="English"></a>
  <a href="./README_RU.md"><img src="https://img.shields.io/badge/RU-Русский-blue?style=flat-square" alt="Русский"></a>
  <a href="./README_ZH.md"><img src="https://img.shields.io/badge/ZH-中文-blue?style=flat-square" alt="中文"></a>
  <br><br>
  <img src="https://img.shields.io/badge/version-1.2.6-blue?style=for-the-badge" alt="Version">
  <a href="https://www.nuget.org/packages/FFS.StaticPack/"><img src="https://img.shields.io/badge/NuGet-FFS.StaticPack-004880?style=for-the-badge&logo=nuget" alt="NuGet"></a>
</p>

# Static Pack - C# Simple binary serialization library
- Lightweight
- Performance
- No dependencies
- No reflections
- No codegen
- No scheme
- Batch primitive operations
- Span / Memory / ReadOnlySequence support
- Compatible with Unity and other C# engines

#### Limitations and Features:
> - Polymorphic types require custom implementation
> - Cyclic references require custom implementation

## Table of Contents
* [Contacts](#contacts)
* [Installation](#installation)
* [Concept](#concept)
* [Quick start](#quick-start)
* [API](#api)
  * [BinaryPackWriter](#binarypackwriter)
  * [BinaryPackReader](#binarypackreader)
  * [BinaryPack](#binarypack)
  * [Array strategies](#array-serialization-strategies)
* [Custom types](#registration-of-custom-types)
* [License](#license)

# Contacts
* [felid.force.studios@gmail.com](mailto:felid.force.studios@gmail.com)
* [Telegram](https://t.me/felid_force_studios)

# Support the project
If you like Static Pack and it helps your project, you can support its development:

<a href="https://www.buymeacoffee.com/felid.force.studios" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="60"></a>

# Installation
* ### As source code
  From the release page or as an archive from the branch. In the `master` branch there is a stable tested version
* ### Installation for Unity
  git module `https://github.com/Felid-Force-Studios/StaticPack.git` in Unity PackageManager
  or adding it to `Packages/manifest.json` `"com.felid-force-studios.static-pack": "https://github.com/Felid-Force-Studios/StaticPack.git"`
* ### NuGet
  ```
  dotnet add package FFS.StaticPack
  ```
  For debug build with assertions:
  ```
  dotnet add package FFS.StaticPack.Debug
  ```
  Packages: [FFS.StaticPack](https://www.nuget.org/packages/FFS.StaticPack/) · [FFS.StaticPack.Debug](https://www.nuget.org/packages/FFS.StaticPack.Debug/)

# Concept
The library provides high-performance tools for binary serialization with support for:
> - Primitive types and arrays
> - Multidimensional arrays
> - Collections (lists, queues, dictionaries, etc.)
> - Custom types
> - Batch primitive writes/reads (2, 3, 4 values at once)
> - Span, Memory, ReadOnlySequence for zero-copy operations
> - Direct file reading/writing
> - Data compression

# Quick start
```csharp
using FFS.Libraries.StaticPack;

BinaryPack.Init();

var buffer = new byte[1024];
var writer = BinaryPackWriter.Create(buffer);

// or create pooled writer
using var writer = BinaryPackWriter.CreateFromPool(1024);

// Base types:
writer.WriteInt(123);
writer.WriteString16("Hello world");
writer.WriteArray(new short[] { 1, 2, 3 });
writer.WriteDictionary(new Dictionary<string, DateTime> { { "today", DateTime.Today }, { "tomorrow", DateTime.Today.AddDays(1) } });

// Batch writes (single EnsureSize call):
writer.WriteFloat(1.0f, 2.0f, 3.0f);  // x, y, z
writer.WriteInt(10, 20, 30, 40);       // 4 values at once

// Span writes:
Span<float> positions = stackalloc float[] { 1f, 2f, 3f };
writer.WriteUnmanaged<float>(positions);

var reader = writer.AsReader(); // new BinaryPackReader(buffer: buffer, size: writer.Position, position: 0)

var readInt = reader.ReadInt();                           // 123
var readString = reader.ReadString16();                   // "Hello world"
var readArray = reader.ReadArray<short>();                // [ 1, 2, 3 ]
var readDict = reader.ReadDictionary<string, DateTime>(); // { "today", ... }, { "tomorrow", ... }

// Batch reads:
reader.ReadFloat(out var x, out var y, out var z);
reader.ReadInt(out var a, out var b, out var c, out var d);

// Span reads:
Span<float> dest = stackalloc float[3];
reader.ReadUnmanaged(dest);

// Custom types:
public struct Person {
    public string Name;
    public int Age;
    public DateTime BirthDate;
    
    public static void Write(ref BinaryPackWriter writer, in Person value) {
        writer.WriteString16(value.Name);
        writer.WriteInt(value.Age);
        writer.WriteDateTime(value.BirthDate);
    }
    
    public static Person Read(ref BinaryPackReader reader) {
        return new Person {
            Name = reader.ReadString16(),
            Age = reader.ReadInt(),
            BirthDate = reader.ReadDateTime()
        };
    }
}

BinaryPack.RegisterWithCollections<Person, StructPackArrayStrategy<Person>>(Person.Write, Person.Read);

writer.Write(new Person { Name = "Alice", Age = 20, BirthDate = DateTime.Now });

var person = reader.Read<Person>();
```


## API

### BinaryPackWriter
Structure for writing binary data

#### Buffer management
```csharp
void EnsureSize(uint size);
byte[] CopyToBytes(bool gzip = false);
int CopyToBytes(ref byte[] result, bool gzip = false);
uint MakePoint(uint size);
```

#### Primitives
```csharp
void WriteByte(byte value);
void WriteSbyte(sbyte value);
void WriteBool(bool value);
void WriteShort(short value);
void WriteUshort(ushort value);
void WriteChar(char value);
void WriteInt(int value);
void WriteUint(uint value);
void WriteLong(long value);
void WriteUlong(ulong value);
void WriteFloat(float value);
void WriteDouble(double value);
void WriteVarInt(int value);   // 1-5 bytes (positive only)
void WriteVarShort(short value); // 1-2 bytes (positive only)
```

#### Batch primitives
```csharp
// Available for: byte, short, ushort, int, uint, float, long, ulong, double
// Single EnsureSize call for all values
void WriteInt(int v0, int v1);
void WriteInt(int v0, int v1, int v2);
void WriteInt(int v0, int v1, int v2, int v3);
void WriteFloat(float v0, float v1);
void WriteFloat(float v0, float v1, float v2);
void WriteFloat(float v0, float v1, float v2, float v3);
// ... same pattern for all types
```

#### Special types
```csharp
void WriteNullable<T>(in T? value) where T : struct;
void WriteDateTime(DateTime value);
void WriteGuid(in Guid value);
void WriteString32(string value);
void WriteString16(string value); // max ushort.MaxValue bytes
void WriteString8(string value);  // max byte.MaxValue bytes
```

#### Collections
```csharp
void WriteArrayUnmanaged<T>(T[] value) where T : unmanaged; // memcpy
void WriteArray<T>(T[] value);                                // per-element
void WriteList<T>(List<T> value, int count = -1);
void WriteQueue<T>(Queue<T> value);
void WriteStack<T>(Stack<T> value);
void WriteLinkedList<T>(LinkedList<T> value);
void WriteHashSet<T>(HashSet<T> value);
void WriteDictionary<K, V>(Dictionary<K, V> value);
```

#### Span / Memory / Sequence
```csharp
void WriteBytes(ReadOnlySpan<byte> value);
void WriteBytes(ReadOnlyMemory<byte> value);
void WriteBytes(in ReadOnlySequence<byte> value);
void WriteUnmanaged<T>(ReadOnlySpan<T> value) where T : unmanaged;  // raw memcpy
void WriteUnmanaged<T>(ReadOnlyMemory<T> value) where T : unmanaged;
void WriteSpanUnmanaged<T>(ReadOnlySpan<T> value) where T : unmanaged; // with array headers
void WriteSpan<T>(ReadOnlySpan<T> value); // per-element with headers
```

#### File handling
```csharp
void WriteFromFile(string filePath, bool gzip = false, uint bufferSize = 4096);
void FlushToFile(string filePath, bool gzip = false, bool flushToDisk = false);
```

### BinaryPackReader
Structure for reading binary data

#### Primitives
```csharp
byte ReadByte();
sbyte ReadSByte();
bool ReadBool();
short ReadShort();
ushort ReadUshort();
char ReadChar();
int ReadInt();
uint ReadUint();
long ReadLong();
ulong ReadUlong();
float ReadFloat();
double ReadDouble();
int ReadVarInt();
short ReadVarShort();
// TryRead* variants return bool
```

#### Batch primitives
```csharp
// Available for: byte, short, ushort, int, uint, float, long, ulong, double
void ReadInt(out int v0, out int v1);
void ReadInt(out int v0, out int v1, out int v2);
void ReadInt(out int v0, out int v1, out int v2, out int v3);
void ReadFloat(out float v0, out float v1, out float v2);
// ... same pattern for all types
```

#### Span / Memory
```csharp
void ReadBytes(Span<byte> destination);
ReadOnlySpan<byte> ReadBytesAsSpan(uint count);     // zero-copy
ReadOnlyMemory<byte> ReadBytesAsMemory(uint count);
ReadOnlySpan<byte> RemainingAsSpan();
ReadOnlyMemory<byte> RemainingAsMemory();
void ReadUnmanaged<T>(Span<T> destination) where T : unmanaged;      // raw memcpy
int ReadSpanUnmanaged<T>(Span<T> destination) where T : unmanaged;   // with array headers
```

#### Collections
```csharp
T[] ReadArrayUnmanaged<T>() where T : unmanaged;
T[] ReadArray<T>();
List<T> ReadList<T>();
Queue<T> ReadQueue<T>();
Stack<T> ReadStack<T>();
LinkedList<T> ReadLinkedList<T>();
HashSet<T> ReadHashSet<T>();
Dictionary<K, V> ReadDictionary<K, V>();
// void ReadX<T>(ref X result) variants for reuse
// SkipArray(), SkipList(), etc. for skipping
```

### BinaryPack
Central registry of serializers

```csharp
static void Init();
static void RegisterWithCollections<T, S>(BinaryWriter<T> writer, BinaryReader<T> reader, S strategy = default)
    where S : IPackArrayStrategy<T>;
static void Register<T>(BinaryWriter<T> writer, BinaryReader<T> reader);
static T Read<T>(this ref BinaryPackReader reader);
static void Write<T>(this ref BinaryPackWriter writer, in T value);
```

### Array serialization strategies
1. `UnmanagedPackArrayStrategy<T>` - for `unmanaged` types, direct memory copy
2. `StructPackArrayStrategy<T>` - for structures, element-by-element serialization
3. `ClassPackArrayStrategy<T>` - for classes, element-by-element serialization

## Registration of custom types

### Example: Simple structure
```csharp
public struct Vector3 {
    public float X, Y, Z;
}

BinaryPack.RegisterWithCollections(
    (ref BinaryPackWriter writer, in Vector3 v) => {
        writer.WriteFloat(v.X, v.Y, v.Z); // batch write
    },
    (ref BinaryPackReader reader) => {
        reader.ReadFloat(out var x, out var y, out var z); // batch read
        return new Vector3 { X = x, Y = y, Z = z };
    },
    new UnmanagedPackArrayStrategy<Vector3>()
);
```

### Example: Complex nested type
```csharp
public class GameState {
    public Player[] Players;
    public Dictionary<int, Item> Inventory;
    public int Level;
}

public struct Player {
    public string Name;
    public Vector3 Position;
}

public struct Item {
    public int Id;
    public float Durability;
}

public static class GameSerializers {
    public static void RegisterAll() {
        BinaryPack.RegisterWithCollections(
            (ref BinaryPackWriter writer, in Vector3 v) => writer.WriteFloat(v.X, v.Y, v.Z),
            (ref BinaryPackReader reader) => {
                reader.ReadFloat(out var x, out var y, out var z);
                return new Vector3 { X = x, Y = y, Z = z };
            },
            new UnmanagedPackArrayStrategy<Vector3>());

        BinaryPack.RegisterWithCollections(
            (ref BinaryPackWriter writer, in Item item) => writer.WriteInt(item.Id),
            (ref BinaryPackReader reader) => new Item { Id = reader.ReadInt() },
            new UnmanagedPackArrayStrategy<Item>());

        BinaryPack.RegisterWithCollections(
            (ref BinaryPackWriter writer, in Player player) => {
                writer.WriteString16(player.Name);
                writer.Write(player.Position);
            },
            (ref BinaryPackReader reader) => new Player {
                Name = reader.ReadString16(),
                Position = reader.Read<Vector3>()
            },
            new StructPackArrayStrategy<Player>());

        BinaryPack.RegisterWithCollections(
            (ref BinaryPackWriter writer, in GameState state) => {
                writer.WriteInt(state.Level);
                writer.WriteArray(state.Players);
                writer.WriteDictionary(state.Inventory);
            },
            (ref BinaryPackReader reader) => new GameState {
                Level = reader.ReadInt(),
                Players = reader.ReadArray<Player>(),
                Inventory = reader.ReadDictionary<int, Item>()
            },
            new ClassPackArrayStrategy<GameState>());
    }
}
```

# License
[MIT license](./LICENSE.md)