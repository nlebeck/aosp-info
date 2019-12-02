# A Whirlwind Tour of ART

If you're reading this page, you probably have some high-level knowledge about Java and Android. You might know that Android apps consist of Java code compiled down to DEX bytecode, and that the Android Runtime (ART) is responsible for executing that bytecode (or compiling it down further into native code). But how does ART's high-level functionality correspond to specific lines of code in its massive codebase? This page aims to answer that question, by discussing different high-level features and walking through the source files and methods that implement them.

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

Object allocation is conceptually fairly straightforward: the interpreter asks the heap to allocate memory for an object of a certain size, and the heap makes the allocation and returns a mirror object pointer to the interpreter. It gets a bit complicated in ART due to the different components and cases involved. In general, all of the object allocation machinery is in the `runtime/gc` directory or its subdirectories.

There are multiple types of "memory spaces" in ART, all of which are defined in the `runtime/gc/space` directory. Some of those spaces are "large object spaces," which hold only primitive arrays or strings larger than a certain threshold (I believe the threshold is 12KB for Android 7.1 on a Pixel XL; the function that decides is `Heap::ShouldAllocLargeObject()` in `runtime/gc/heap-inl.h`). There are two types of large object spaces: `LargeObjectMapSpace` and `FreeListSpace`, both declared in `runtime/gc/space/large_object_space.h`. As far as I can tell, any given app has a single large object space that is one of those two types, and I'm not sure how ART decides which type to use. Most normal objects are allocated from a `RosAllocSpace`, which corresponds to the "run-of-slots" `RosAlloc` allocator defined in `runtime/gc/allocator`.

Here is the call chain for a normal (non-large) object allocation. My notes about this code are phrased even more hesitantly than usual, so take it with a grain of salt:

- The interpreter calls `AllocObjectFromCode()` in `runtime/entrypoints/entrypoint_utils-in.h`
- ... which calls `Class::Alloc()` in `runtime/mirror/class-in.h`
- ... which calls `Heap::AllocObjectWithAllocator()` in `runtime/gc/heap-inl.h`
- ... which first calls `RosAllocSpace::AllocThreadLocal()` in `runtime/gc/space/rosalloc_space-inl.h` and then calls `Heap::TryToAllocate()` if that doesn't work
- `RosAllocSpace::AllocThreadLocal()` calls `RosAlloc::AllocFromThreadLocalRun()` in `runtime/gc/allocator/rosalloc-inl.h`
- `Heap::TryToAllocate()` calls `RosAllocSpace::AllocNonvirtual()` in `runtime/gc/space/rosalloc_space.h`
- ... which calls `RosAllocSpace::AllocCommon()` in `runtime/gc/space/rosalloc_space-inl.h`
- ... which calls `RosAlloc::Alloc()` in `runtime/gc/allocator/rosalloc-inl.h`
- ... which calls either `RosAlloc::AllocFromRun()` or `RosAlloc::AllocFromRunThreadUnsafe()` in `runtime/gc/allocator/rosalloc.cc`
- `RosAlloc::AllocFromRunThreadUnsafe()` calls `RosAlloc::AllocFromCurrentRunUnlocked()`, and `RosAlloc::AllocFromRun()` might call `RosAlloc::AllocFromCurrentRunUnlocked()` as well, if the requested allocation is too large for a thread-local run

Here is the call chain for a large object allocation. The call chain is the same for the first three steps, up to the call to `Heap::AllocObjectWithAllocator()`. Then it proceeds as follows:

- `Heap::AllocObjectWithAllocator()` calls `Heap::AllocLargeObject()`
- ... which calls back into `Heap::AllocObjectWithAllocator()`, this time setting `kCheckLargeObject` to `false` and passing in allocator type `kAllocatorTypeLOS`.
- ... which calls `Heap::TryToAllocate()`
- ... which calls `LargeObjectSpace::Alloc()` in `runtime/gc/space/large_object_space.cc` (the concrete method called depends on which subclass of `LargeObjectSpace` is in use, but both are defined in that file)

## Garbage Collection

TODO

## Compiling DEX Bytecode Down to Native Code

TODO
