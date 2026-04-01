# DataContract Deserialization – Populating an Array

Sample project demonstrating how to correctly deserialize a WCF XML message into a C# object where a `[DataMember]` field is an **array**, using `DataContractSerializer`.

Created as an answer to the Stack Overflow question:
[WCF XML deserialization not populating array](https://stackoverflow.com/questions/57313063/wcf-xml-deserialization-not-populating-array)

---

## Problem

When deserializing a WCF XML message with `DataContractSerializer`, an array-typed `[DataMember]` field (e.g., `Application_SA[]`) is not populated — it remains `null` or empty even though the XML clearly contains the expected child elements.

The root cause is how `DataContractSerializer` expects arrays to be represented in XML. By default it looks for a **wrapper element** whose name matches the member name, containing individual items whose element name matches the item type. If the XML structure doesn't match this expectation exactly, the array is silently skipped.

### The Fix

Map the array **directly** without a wrapping object on top of the array. The XML must have a wrapper element (`<Application_SAList>`) containing repeated `<Application_SA>` child elements, and the C# data contract must declare the member as a plain array — not a wrapper class containing an array:

```csharp
// ✅ Correct – array directly as a DataMember
[DataMember]
public Application_SA[] Application_SAList;

// ❌ Incorrect – wrapping the array in an extra class causes the array not to populate
```

---

## Solution Overview

The application reads `Message.xml` from disk and deserializes it into `ApplicationRequestImpl` using `DataContractSerializer`, then prints the deserialized values to the console.

### Data Contracts

| Class                    | Purpose                                                                 |
|--------------------------|-------------------------------------------------------------------------|
| `ApplicationRequestImpl` | Root data contract. Contains an `Application` object and an array of `Application_SA` items. |
| `Application`            | Holds a single `Amount` field.                                          |
| `Application_SA`         | Represents one security entry with a `Security_Key` field.             |

All classes use `[DataContract(Namespace = "http://xyz")]` to match the XML namespace in `Message.xml`.

### XML Input (`Message.xml`)

```xml
<ApplicationRequestImpl xmlns="http://xyz">
  <Application>
    <Amount>50000.0</Amount>
  </Application>
  <Application_SAList>
    <Application_SA>
      <Security_Key>1588</Security_Key>
    </Application_SA>
    <Application_SA>
      <Security_Key>123</Security_Key>
    </Application_SA>
  </Application_SAList>
</ApplicationRequestImpl>
```

`<Application_SAList>` is the wrapper element and each `<Application_SA>` is a direct child — this structure is what allows `DataContractSerializer` to populate the `Application_SA[]` array correctly.

### Deserialization (`Program.cs`)

```csharp
var serializer = new DataContractSerializer(typeof(ApplicationRequestImpl));

using var file = File.OpenRead("./Message.xml");
using var xml = XmlReader.Create(file);

ApplicationRequestImpl result = (ApplicationRequestImpl)serializer.ReadObject(xml);

Console.WriteLine($"Amount is {result.Application.Amount}.");

for (int i = 0; i < result.Application_SAList.Length; i++) {
    Console.WriteLine($"Security key at position {i} is {result.Application_SAList[i].Security_Key}.");
}
```

**Expected output:**
```
Amount is 50000.0.
Security key at position 0 is 1588.
Security key at position 1 is 123.
```

---

## Project Structure

```
.
└── src/
    ├── Data/
    │   ├── Application.cs              # DataContract with Amount field
    │   ├── ApplicationRequestImpl.cs   # Root DataContract with Application and Application_SA array
    │   └── Application_SA.cs           # DataContract with Security_Key field
    ├── Message.xml                     # Sample XML input message
    └── Program.cs                      # Entry point – deserializes XML and prints results
```

---

## Key Concepts

- **`[DataContract]`** – marks a class as serializable by `DataContractSerializer`. The `Namespace` must match the XML namespace exactly.
- **`[DataMember]`** – marks a field or property to be included in serialization/deserialization.
- **Array deserialization** – `DataContractSerializer` maps an array member named `Foo` by looking for a `<Foo>` wrapper element containing repeated `<Bar>` items, where `Bar` is the item type name. No extra wrapper class is needed.
- **`DataContractSerializer` vs `XmlSerializer`** – `DataContractSerializer` is the default serializer for WCF and has stricter rules around namespaces and element naming than `XmlSerializer`.

---

## Prerequisites

- [.NET SDK](https://dotnet.microsoft.com/download) (project targets .NET — see `DataContractDeserialization.csproj`)
- No external dependencies or NuGet packages required

## Running the Project

```bash
cd src
dotnet run
```