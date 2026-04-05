<p align="center">
  <a href="./README.md"><img src="https://img.shields.io/badge/EN-English-blue?style=flat-square" alt="English"></a>
  <a href="./README_RU.md"><img src="https://img.shields.io/badge/RU-Русский-blue?style=flat-square" alt="Русский"></a>
  <a href="./README_ZH.md"><img src="https://img.shields.io/badge/ZH-中文-blue?style=flat-square" alt="中文"></a>
  <br><br>
  <img src="https://img.shields.io/badge/version-1.1.0-blue?style=for-the-badge" alt="Version">
  <a href="https://www.nuget.org/packages/FFS.StaticPack/"><img src="https://img.shields.io/badge/NuGet-FFS.StaticPack-004880?style=for-the-badge&logo=nuget" alt="NuGet"></a>
</p>

# Static Pack - C# библиотека бинарной сериализации
- Легковесность
- Производительность
- Без зависимостей
- Без рефлексии
- Без кодогенерации
- Без схемы данных
- Батч-операции для примитивов
- Поддержка Span / Memory / ReadOnlySequence
- Совместимость с Unity и другими C# движками

#### Ограничения и особенности:
> - Полиморфные типы требуют ручной реализации
> - Циклические ссылки требуют ручной реализации

## Оглавление
* [Контакты](#контакты)
* [Установка](#установка)
* [Концепция](#концепция)
* [Быстрый старт](#быстрый-старт)
* [API](#api)
  * [BinaryPackWriter](#binarypackwriter)
  * [BinaryPackReader](#binarypackreader)
  * [BinaryPack](#binarypack)
  * [Стратегии массивов](#стратегии-сериализации-массивов)
* [Пользовательские типы](#регистрация-пользовательских-типов)
* [Лицензия](#лицензия)

# Контакты
* [felid.force.studios@gmail.com](mailto:felid.force.studios@gmail.com)
* [Telegram](https://t.me/felid_force_studios)

# Поддержать проект
Если вам нравится Static Pack и он помогает вашему проекту, вы можете поддержать разработку:

<a href="https://www.buymeacoffee.com/felid.force.studios" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="60"></a>

# Установка
* ### В виде исходников
  Со страницы релизов или как архив из нужной ветки. В ветке `master` стабильная проверенная версия
* ### Установка для Unity
  git модуль `https://github.com/Felid-Force-Studios/StaticPack.git` в Unity PackageManager
  или добавление в `Packages/manifest.json` `"com.felid-force-studios.static-pack": "https://github.com/Felid-Force-Studios/StaticPack.git"`
* ### NuGet
  ```
  dotnet add package FFS.StaticPack
  ```
  Для debug-сборки с проверками:
  ```
  dotnet add package FFS.StaticPack.Debug
  ```
  Пакеты: [FFS.StaticPack](https://www.nuget.org/packages/FFS.StaticPack/) · [FFS.StaticPack.Debug](https://www.nuget.org/packages/FFS.StaticPack.Debug/)

# Концепция
Библиотека предоставляет высокопроизводительные инструменты для бинарной сериализации с поддержкой:
> - Примитивных типов и массивов
> - Многомерных массивов
> - Коллекций (списки, очереди, словари и т.д.)
> - Пользовательских типов
> - Батч-записи/чтения примитивов (2, 3, 4 значения за раз)
> - Span, Memory, ReadOnlySequence для zero-copy операций
> - Прямого чтения/записи файлов
> - Сжатия данных

# Быстрый старт
```csharp
using FFS.Libraries.StaticPack;

BinaryPack.Init();

var buffer = new byte[1024];
var writer = BinaryPackWriter.Create(buffer);

// или создать writer из пула
using var writer = BinaryPackWriter.CreateFromPool(1024);

// Базовые типы:
writer.WriteInt(123);
writer.WriteString16("Привет мир");
writer.WriteArray(new short[] { 1, 2, 3 });
writer.WriteDictionary(new Dictionary<string, DateTime> { { "сегодня", DateTime.Today }, { "завтра", DateTime.Today.AddDays(1) } });

// Батч-запись (один вызов EnsureSize):
writer.WriteFloat(1.0f, 2.0f, 3.0f);  // x, y, z
writer.WriteInt(10, 20, 30, 40);       // 4 значения за раз

// Запись через Span:
Span<float> positions = stackalloc float[] { 1f, 2f, 3f };
writer.WriteUnmanaged<float>(positions);

var reader = writer.AsReader(); // new BinaryPackReader(buffer: buffer, size: writer.Position, position: 0)

var readInt = reader.ReadInt();                           // 123
var readString = reader.ReadString16();                   // "Привет мир"
var readArray = reader.ReadArray<short>();                // [ 1, 2, 3 ]
var readDict = reader.ReadDictionary<string, DateTime>(); // { "сегодня", ... }, { "завтра", ... }

// Батч-чтение:
reader.ReadFloat(out var x, out var y, out var z);
reader.ReadInt(out var a, out var b, out var c, out var d);

// Чтение через Span:
Span<float> dest = stackalloc float[3];
reader.ReadUnmanaged(dest);

// Пользовательские типы:
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
Структура для записи бинарных данных

#### Управление буфером
```csharp
void EnsureSize(uint size);
byte[] CopyToBytes(bool gzip = false);
int CopyToBytes(ref byte[] result, bool gzip = false);
uint MakePoint(uint size);
```

#### Примитивы
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
void WriteVarInt(int value);   // 1-5 байт (только положительные)
void WriteVarShort(short value); // 1-2 байта (только положительные)
```

#### Батч-примитивы
```csharp
// Доступны для: byte, short, ushort, int, uint, float, long, ulong, double
// Один вызов EnsureSize для всех значений
void WriteInt(int v0, int v1);
void WriteInt(int v0, int v1, int v2);
void WriteInt(int v0, int v1, int v2, int v3);
void WriteFloat(float v0, float v1);
void WriteFloat(float v0, float v1, float v2);
void WriteFloat(float v0, float v1, float v2, float v3);
// ... аналогично для всех типов
```

#### Специальные типы
```csharp
void WriteNullable<T>(in T? value) where T : struct;
void WriteDateTime(DateTime value);
void WriteGuid(in Guid value);
void WriteString32(string value);
void WriteString16(string value); // макс. ushort.MaxValue байт
void WriteString8(string value);  // макс. byte.MaxValue байт
```

#### Коллекции
```csharp
void WriteArrayUnmanaged<T>(T[] value) where T : unmanaged; // memcpy
void WriteArray<T>(T[] value);                                // поэлементно
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
void WriteUnmanaged<T>(ReadOnlySpan<T> value) where T : unmanaged;  // прямой memcpy
void WriteUnmanaged<T>(ReadOnlyMemory<T> value) where T : unmanaged;
void WriteSpanUnmanaged<T>(ReadOnlySpan<T> value) where T : unmanaged; // с заголовками массива
void WriteSpan<T>(ReadOnlySpan<T> value); // поэлементно с заголовками
```

#### Работа с файлами
```csharp
void WriteFromFile(string filePath, bool gzip = false, uint bufferSize = 4096);
void FlushToFile(string filePath, bool gzip = false, bool flushToDisk = false);
```

### BinaryPackReader
Структура для чтения бинарных данных

#### Примитивы
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
// TryRead* варианты возвращают bool
```

#### Батч-примитивы
```csharp
// Доступны для: byte, short, ushort, int, uint, float, long, ulong, double
void ReadInt(out int v0, out int v1);
void ReadInt(out int v0, out int v1, out int v2);
void ReadInt(out int v0, out int v1, out int v2, out int v3);
void ReadFloat(out float v0, out float v1, out float v2);
// ... аналогично для всех типов
```

#### Span / Memory
```csharp
void ReadBytes(Span<byte> destination);
ReadOnlySpan<byte> ReadBytesAsSpan(uint count);     // zero-copy
ReadOnlyMemory<byte> ReadBytesAsMemory(uint count);
ReadOnlySpan<byte> RemainingAsSpan();
ReadOnlyMemory<byte> RemainingAsMemory();
void ReadUnmanaged<T>(Span<T> destination) where T : unmanaged;      // прямой memcpy
int ReadSpanUnmanaged<T>(Span<T> destination) where T : unmanaged;   // с заголовками массива
```

#### Коллекции
```csharp
T[] ReadArrayUnmanaged<T>() where T : unmanaged;
T[] ReadArray<T>();
List<T> ReadList<T>();
Queue<T> ReadQueue<T>();
Stack<T> ReadStack<T>();
LinkedList<T> ReadLinkedList<T>();
HashSet<T> ReadHashSet<T>();
Dictionary<K, V> ReadDictionary<K, V>();
// void ReadX<T>(ref X result) варианты для переиспользования
// SkipArray(), SkipList() и т.д. для пропуска
```

### BinaryPack
Центральный реестр сериализаторов

```csharp
static void Init();
static void RegisterWithCollections<T, S>(BinaryWriter<T> writer, BinaryReader<T> reader, S strategy = default)
    where S : IPackArrayStrategy<T>;
static void Register<T>(BinaryWriter<T> writer, BinaryReader<T> reader);
static T Read<T>(this ref BinaryPackReader reader);
static void Write<T>(this ref BinaryPackWriter writer, in T value);
```

### Стратегии сериализации массивов
1. `UnmanagedPackArrayStrategy<T>` - для `unmanaged` типов, прямое копирование памяти
2. `StructPackArrayStrategy<T>` - для структур, поэлементная сериализация
3. `ClassPackArrayStrategy<T>` - для классов, поэлементная сериализация

## Регистрация пользовательских типов

### Пример: Простая структура
```csharp
public struct Vector3 {
    public float X, Y, Z;
}

BinaryPack.RegisterWithCollections(
    (ref BinaryPackWriter writer, in Vector3 v) => {
        writer.WriteFloat(v.X, v.Y, v.Z); // батч-запись
    },
    (ref BinaryPackReader reader) => {
        reader.ReadFloat(out var x, out var y, out var z); // батч-чтение
        return new Vector3 { X = x, Y = y, Z = z };
    },
    new UnmanagedPackArrayStrategy<Vector3>()
);
```

### Пример: Сложный вложенный тип
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

# Лицензия
[MIT license](./LICENSE.md)
