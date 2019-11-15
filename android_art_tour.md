# A Whirlwind Tour of ART

If you're reading this page, you probably have some high-level knowledge about Java and Android. You might know that Android apps consist of Java code compiled down to DEX bytecode, and that something called the Android Runtime (ART) is responsible for executing that bytecode (or compiling it down further into native code). But how does ART actually work? This page aims to show you which specific files and lines of code in the ART codebase implement the high-level functionality that you expect.

The notes are for the version of ART in Android 7.1. I'm not sure how much ART has changed since then. It seemed like in Android 8, ART got new features but kept most of the same code, so hopefully that trend has continued, and this information is still relevant for even newer versions.

Here are links to the relevant AOSP repos at tag `android-7.1.1_r7`:

- [art repo](https://android.googlesource.com/platform/art/+/refs/tags/android-7.1.1_r57)
- [libcore repo](https://android.googlesource.com/platform/libcore/+/refs/tags/android-7.1.1_r57/)

The `art` repo contains the implementation of ART itself, while the `libcore` repo contains, among other things, Android's version of the Java Class Library. Unless otherwise specified, all source file paths are to files in the `art` repo and are relative paths from its root directory.

## Representing Objects

ART internally represents each Java object as a C++ "mirror object," defined in `runtime/mirror/object.h`. The bottom of that file (lines 591-593) defines the fields in an object's header, while the rest of the file declares the methods that ART uses to access and modify an object. Most of those methods are implemented in `runtime/mirror/object-inl.h`, with a few implemented in `runtime/mirror/object.cc`. Arrays, strings, and classes are special types of object that are represented by their own mirror object subclasses, defined in the corresponding files in `runtime/mirror`.

The Java source file defining the top-level `Object` class is in the `libcore` repo at path `ojluni/src/main/java/java/lang/Object.java`. Note that lines 40-41 of this file contain "shadow variable" definitions corresponding to the fields in the mirror object header. If you want to add more fields to the mirror object header, you'll need to add corresponding shadow variables here.

## Executing DEX Bytecode Instructions in the Interpreter

The ART interpreter directly executes DEX bytecode instructions, accessing and modifying mirror objects and DEX registers as necessary. This section walks through the call chain as the interpreter executes a single DEX instruction.

The ART codebase has multiple interpreter implementations. The "mterp interpreter," which ART uses by default, is an optimized interpreter written in assembly code. The "switch interpreter," on the other hand, is basically a giant switch statement. You can change which interpreter ART uses by modifying the value of the variable `kInterpreterImplKind` on line 243 of `runtime/interpreter/interpreter.cc`. The discussion below assumes that we're using the switch interpreter, since its code is the easiest to read.

Let's follow the execution of the DEX bytecode instruction `iget-byte`:

- `Execute()` in `runtime/interpreter/interpreter.cc` calls `ExecuteSwitchImpl()` in `runtime/interpreter/interpreter_switch_impl.cc`
- ... whose execution falls through to the case `Instruction::IGET_BYTE` on line 1195
- ... which calls `DoFieldGet()` in `runtime/interpreter/interpreter_common.cc`
- ... which first gets a pointer to the object by calling `ShadowFrame::GetVRegReference()` in `runtime/stack.h`, and then calls `ArtField::GetByte()` in `runtime/art_field-inl.h`
- ... which uses the `GET_FIELD` macro to call `Object::GetFieldByte()` in `runtime/mirror/object-inl.h`
- ... which calls `Object::GetField()`
- ... which first computes the field’s address as an offset from the object’s address, then casts that address to an Atomic pointer and calls `Atomic::LoadJavaData()` in `runtime/atomic.h`
- ... which calls `std::atomic::load()`

## Object Allocation

TODO

## Garbage Collection

TODO

## Compiling DEX Bytecode Down to Native Code

TODO
