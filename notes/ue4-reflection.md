# Unreal Engine 4 reflection: from a UObject to a property offset

Notes on how UE4's reflection system lets the engine find "where is field X inside this object". Based on the public engine source (`Engine/Source/Runtime/CoreUObject/`). Layouts differ between engine versions and build configurations, so always check the headers of the version you work with.

## 1. Why reflection exists

C++ has no runtime type information that is rich enough for an editor. UE4 adds it with macros (`UCLASS`, `UPROPERTY`, `UFUNCTION`) and a code generator (Unreal Header Tool). The generated metadata drives the editor details panel, serialization, garbage collection, replication and Blueprints.

## 2. The chain

```
GUObjectArray            "phone book"   flat list of every live UObject
     |
UObjectBase              "ID card"      one per object: name, class, outer
     |  ClassPrivate
UClass (derives UStruct) "blueprint"    parent class, field list, functions
     |  PropertyLink / ChildProperties
FProperty                "field entry"  name, type, Offset_Internal
```

| Level | Useful members (UE 4.25+) |
|---|---|
| `UObjectBase` | `ObjectFlags`, `InternalIndex`, `ClassPrivate`, `NamePrivate`, `OuterPrivate` |
| `UStruct` (base of `UClass`) | `SuperStruct`, `Children` (functions), `ChildProperties`, `PropertiesSize`, `PropertyLink` |
| `FProperty` | `ArrayDim`, `ElementSize`, `PropertyFlags`, `Offset_Internal`, `PropertyLinkNext` |

`Offset_Internal` is the byte offset of the field inside an instance of the owning class. That is the number reflection gives you.

## 3. An engine-side example

Reading a named field through reflection, the way tools and Blueprint glue do it:

```cpp
UClass* Cls = Obj->GetClass();
FProperty* Prop = Cls->FindPropertyByName(TEXT("TeamID"));
if (Prop)
{
    void* ValuePtr = Prop->ContainerPtrToValuePtr<void>(Obj);   // = Obj + Offset_Internal
    // cast according to Prop's type, e.g. FIntProperty
}
```

Walking all fields: `for (TFieldIterator<FProperty> It(Cls); It; ++It) { ... }`.

## 4. A way to remember it

| Reflection | Building analogy |
|---|---|
| `UObjectBase` | "this building is called Vanke City" (you know the name) |
| `UClass` | the building's floor plan (you know which rooms exist) |
| `FProperty` | "master bedroom, 3rd floor, 2nd door on the left, 25 m2" (exact place and size) |

## 5. Version notes

- Up to 4.24 the field type is `UProperty` (a `UObject`). From 4.25 it is `FProperty`, part of a lighter `FField` hierarchy, and `UStruct` keeps them in `ChildProperties`.
- The `FName` storage changed in 4.23 (name pool made of blocks), which matters when you decode names by hand.
- Packaged games may be built with modified engines. If member order differs from the headers, trust the actual build, not the stock layout.

## Where to read the source

`UObjectBase.h`, `Class.h`, `UnrealType.h`, `UObjectArray.h` under `Engine/Source/Runtime/CoreUObject/Public/UObject/`.
