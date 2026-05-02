<p align="center">
  <a href="./README.md"><img src="https://img.shields.io/badge/EN-English-blue?style=flat-square" alt="English"></a>
  <a href="./README_RU.md"><img src="https://img.shields.io/badge/RU-Русский-blue?style=flat-square" alt="Русский"></a>
  <a href="./README_ZH.md"><img src="https://img.shields.io/badge/ZH-中文-blue?style=flat-square" alt="中文"></a>
  <br><br>
  <img src="https://img.shields.io/badge/version-1.2.5-blue?style=for-the-badge" alt="Version">
  <a href="https://www.nuget.org/packages/FFS.StaticPack/"><img src="https://img.shields.io/badge/NuGet-FFS.StaticPack-004880?style=for-the-badge&logo=nuget" alt="NuGet"></a>
</p>

# Static Pack - C# 简洁二进制序列化库
- 轻量级
- 高性能
- 零依赖
- 无反射
- 无代码生成
- 无数据模式
- 批量原始类型操作
- 支持 Span / Memory / ReadOnlySequence
- 兼容 Unity 及其他 C# 引擎

#### 限制与特性:
> - 多态类型需要自定义实现
> - 循环引用需要自定义实现

## 目录
* [联系方式](#联系方式)
* [安装](#安装)
* [概念](#概念)
* [快速开始](#快速开始)
* [API](#api)
  * [BinaryPackWriter](#binarypackwriter)
  * [BinaryPackReader](#binarypackreader)
  * [BinaryPack](#binarypack)
  * [数组策略](#数组序列化策略)
* [自定义类型](#自定义类型注册)
* [许可证](#许可证)

# 联系方式
* [felid.force.studios@gmail.com](mailto:felid.force.studios@gmail.com)
* [Telegram](https://t.me/felid_force_studios)

# 支持项目
如果您喜欢 Static Pack 并且它对您的项目有所帮助，您可以支持开发：

<a href="https://www.buymeacoffee.com/felid.force.studios" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="60"></a>

# 安装
* ### 以源代码形式
  从发布页面或从分支下载归档文件。`master` 分支包含稳定测试版本
* ### Unity 安装
  通过 Unity PackageManager 的 git 模块 `https://github.com/Felid-Force-Studios/StaticPack.git`
  或添加到 `Packages/manifest.json` `"com.felid-force-studios.static-pack": "https://github.com/Felid-Force-Studios/StaticPack.git"`
* ### NuGet
  ```
  dotnet add package FFS.StaticPack
  ```
  用于带断言的调试构建：
  ```
  dotnet add package FFS.StaticPack.Debug
  ```
  包：[FFS.StaticPack](https://www.nuget.org/packages/FFS.StaticPack/) · [FFS.StaticPack.Debug](https://www.nuget.org/packages/FFS.StaticPack.Debug/)

# 概念
本库提供高性能的二进制序列化工具，支持：
> - 原始类型和数组
> - 多维数组
> - 集合（列表、队列、字典等）
> - 自定义类型
> - 批量原始类型读写（一次 2、3、4 个值）
> - Span、Memory、ReadOnlySequence 零拷贝操作
> - 直接文件读写
> - 数据压缩

# 快速开始
```csharp
using FFS.Libraries.StaticPack;

BinaryPack.Init();

var buffer = new byte[1024];
var writer = BinaryPackWriter.Create(buffer);

// 或创建池化 writer
using var writer = BinaryPackWriter.CreateFromPool(1024);

// 基本类型：
writer.WriteInt(123);
writer.WriteString16("Hello world");
writer.WriteArray(new short[] { 1, 2, 3 });
writer.WriteDictionary(new Dictionary<string, DateTime> { { "today", DateTime.Today }, { "tomorrow", DateTime.Today.AddDays(1) } });

// 批量写入（单次 EnsureSize 调用）：
writer.WriteFloat(1.0f, 2.0f, 3.0f);  // x, y, z
writer.WriteInt(10, 20, 30, 40);       // 一次写入 4 个值

// Span 写入：
Span<float> positions = stackalloc float[] { 1f, 2f, 3f };
writer.WriteUnmanaged<float>(positions);

var reader = writer.AsReader(); // new BinaryPackReader(buffer: buffer, size: writer.Position, position: 0)

var readInt = reader.ReadInt();                           // 123
var readString = reader.ReadString16();                   // "Hello world"
var readArray = reader.ReadArray<short>();                // [ 1, 2, 3 ]
var readDict = reader.ReadDictionary<string, DateTime>(); // { "today", ... }, { "tomorrow", ... }

// 批量读取：
reader.ReadFloat(out var x, out var y, out var z);
reader.ReadInt(out var a, out var b, out var c, out var d);

// Span 读取：
Span<float> dest = stackalloc float[3];
reader.ReadUnmanaged(dest);

// 自定义类型：
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
用于写入二进制数据的结构体

#### 缓冲区管理
```csharp
void EnsureSize(uint size);
byte[] CopyToBytes(bool gzip = false);
int CopyToBytes(ref byte[] result, bool gzip = false);
uint MakePoint(uint size);
```

#### 原始类型
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
void WriteVarInt(int value);   // 1-5 字节（仅正数）
void WriteVarShort(short value); // 1-2 字节（仅正数）
```

#### 批量原始类型
```csharp
// 可用类型：byte, short, ushort, int, uint, float, long, ulong, double
// 所有值共用一次 EnsureSize 调用
void WriteInt(int v0, int v1);
void WriteInt(int v0, int v1, int v2);
void WriteInt(int v0, int v1, int v2, int v3);
void WriteFloat(float v0, float v1);
void WriteFloat(float v0, float v1, float v2);
void WriteFloat(float v0, float v1, float v2, float v3);
// ... 其他类型同理
```

#### 特殊类型
```csharp
void WriteNullable<T>(in T? value) where T : struct;
void WriteDateTime(DateTime value);
void WriteGuid(in Guid value);
void WriteString32(string value);
void WriteString16(string value); // 最大 ushort.MaxValue 字节
void WriteString8(string value);  // 最大 byte.MaxValue 字节
```

#### 集合
```csharp
void WriteArrayUnmanaged<T>(T[] value) where T : unmanaged; // memcpy
void WriteArray<T>(T[] value);                                // 逐元素
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
void WriteUnmanaged<T>(ReadOnlySpan<T> value) where T : unmanaged;  // 直接 memcpy
void WriteUnmanaged<T>(ReadOnlyMemory<T> value) where T : unmanaged;
void WriteSpanUnmanaged<T>(ReadOnlySpan<T> value) where T : unmanaged; // 带数组头
void WriteSpan<T>(ReadOnlySpan<T> value); // 带头逐元素
```

#### 文件操作
```csharp
void WriteFromFile(string filePath, bool gzip = false, uint bufferSize = 4096);
void FlushToFile(string filePath, bool gzip = false, bool flushToDisk = false);
```

### BinaryPackReader
用于读取二进制数据的结构体

#### 原始类型
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
// TryRead* 变体返回 bool
```

#### 批量原始类型
```csharp
// 可用类型：byte, short, ushort, int, uint, float, long, ulong, double
void ReadInt(out int v0, out int v1);
void ReadInt(out int v0, out int v1, out int v2);
void ReadInt(out int v0, out int v1, out int v2, out int v3);
void ReadFloat(out float v0, out float v1, out float v2);
// ... 其他类型同理
```

#### Span / Memory
```csharp
void ReadBytes(Span<byte> destination);
ReadOnlySpan<byte> ReadBytesAsSpan(uint count);     // 零拷贝
ReadOnlyMemory<byte> ReadBytesAsMemory(uint count);
ReadOnlySpan<byte> RemainingAsSpan();
ReadOnlyMemory<byte> RemainingAsMemory();
void ReadUnmanaged<T>(Span<T> destination) where T : unmanaged;      // 直接 memcpy
int ReadSpanUnmanaged<T>(Span<T> destination) where T : unmanaged;   // 带数组头
```

#### 集合
```csharp
T[] ReadArrayUnmanaged<T>() where T : unmanaged;
T[] ReadArray<T>();
List<T> ReadList<T>();
Queue<T> ReadQueue<T>();
Stack<T> ReadStack<T>();
LinkedList<T> ReadLinkedList<T>();
HashSet<T> ReadHashSet<T>();
Dictionary<K, V> ReadDictionary<K, V>();
// void ReadX<T>(ref X result) 变体用于复用
// SkipArray(), SkipList() 等用于跳过
```

### BinaryPack
序列化器中央注册表

```csharp
static void Init();
static void RegisterWithCollections<T, S>(BinaryWriter<T> writer, BinaryReader<T> reader, S strategy = default)
    where S : IPackArrayStrategy<T>;
static void Register<T>(BinaryWriter<T> writer, BinaryReader<T> reader);
static T Read<T>(this ref BinaryPackReader reader);
static void Write<T>(this ref BinaryPackWriter writer, in T value);
```

### 数组序列化策略
1. `UnmanagedPackArrayStrategy<T>` - 用于 `unmanaged` 类型，直接内存复制
2. `StructPackArrayStrategy<T>` - 用于结构体，逐元素序列化
3. `ClassPackArrayStrategy<T>` - 用于类，逐元素序列化

## 自定义类型注册

### 示例：简单结构体
```csharp
public struct Vector3 {
    public float X, Y, Z;
}

BinaryPack.RegisterWithCollections(
    (ref BinaryPackWriter writer, in Vector3 v) => {
        writer.WriteFloat(v.X, v.Y, v.Z); // 批量写入
    },
    (ref BinaryPackReader reader) => {
        reader.ReadFloat(out var x, out var y, out var z); // 批量读取
        return new Vector3 { X = x, Y = y, Z = z };
    },
    new UnmanagedPackArrayStrategy<Vector3>()
);
```

### 示例：复杂嵌套类型
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

# 许可证
[MIT license](./LICENSE.md)
