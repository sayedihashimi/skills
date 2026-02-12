---
name: dotnet-pinvoke
description: Correctly call native (C/C++) libraries from .NET using P/Invoke and LibraryImport. Covers function signatures, string marshalling, memory lifetime, SafeHandle, and cross-platform patterns. Use when writing or reviewing any managed-to-native boundary code.
---

# .NET P/Invoke

Calling native code from .NET is powerful but unforgiving. Incorrect signatures, garbled strings, and leaked or accessing freed memory are three of the most common sources of bugs; all of them can manifest as intermittent crashes, silent data corruption, or access violations that appear far from the actual defect.

This skill covers both `DllImport` (available since .NET Framework 1.0) and `LibraryImport` (source-generated, .NET 7+). Both are covered equally because many codebases target older TFMs or must maintain existing `DllImport` declarations. When targeting .NET Framework, always use `DllImport`. When targeting .NET 7+, prefer `LibraryImport` for new code. When native AOT is a requirement, `LibraryImport` is the only option.

## When to Use

- Writing new P/Invoke or `LibraryImport` declarations
- Reviewing or debugging existing native interop code
- Wrapping a C or C++ library for use in .NET
- Diagnosing crashes, memory leaks, or corruption at the managed/native boundary

## When Not to Use

- COM interop (different lifetime and threading model)
- C++/CLI mixed-mode assemblies (avoid in new code; C# interop is faster and more portable)
- Pure managed code with no native dependencies

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Native header or documentation | Yes | C/C++ function signatures, struct definitions, calling conventions |
| Target framework | Yes | Determines whether to use `DllImport` or `LibraryImport` |
| Target platforms | Recommended | Affects type sizes (`long`, `size_t`) and library naming |
| Memory ownership contract | Yes | Who allocates and who frees each buffer or handle |

---

## Workflow

### Step 1: Choose DllImport or LibraryImport

| Aspect | `DllImport` | `LibraryImport` (.NET 7+) |
|--------|-------------|---------------------------|
| **Mechanism** | Runtime marshalling | Source generator (compile-time) |
| **AOT / Trim safe** | No | Yes |
| **String marshalling** | `CharSet` enum | `StringMarshalling` enum |
| **Error handling** | `SetLastError` | `SetLastPInvokeError` |
| **Availability** | .NET Framework 1.0+ | .NET 7+ only |

Use `LibraryImport` for new code on .NET 7+. Use `DllImport` for .NET Framework, .NET Standard, or earlier .NET Core.

### Step 2: Map Native Types to .NET Types

This is where most bugs originate. Every parameter must match exactly.

| C / Win32 Type | .NET Type | Notes |
|----------------|-----------|-------|
| `int` | `int` | Always 32-bit in Win32 ABI |
| `int32_t` | `int` | |
| `uint32_t` | `uint` | |
| `int64_t` | `long` | |
| `uint64_t` | `ulong` | |
| `HRESULT` | `int` | Some tools project this as an enumeration |
| `long` | **`CLong`** | C `long` is 32-bit on Windows, 64-bit on 64-bit Unix — never use `int` or `long`. With `LibraryImport`, requires `[assembly: DisableRuntimeMarshalling]` or you get SYSLIB1051. With `DllImport`, works without it |
| `size_t` | `nuint` | Pointer-sized. Never use `ulong` |
| `intptr_t` | `nint` | Pointer-sized |
| `BOOL` (Win32) | `int` | Not `bool` — Win32 `BOOL` is 4 bytes |
| `bool` (C99) | `[MarshalAs(UnmanagedType.U1)] bool` | Must specify 1-byte marshal |
| `HANDLE`, `HWND` | `SafeHandle` | Prefer over raw `IntPtr` |
| `LPWSTR` / `wchar_t*` | `string` | Must specify UTF-16 encoding |
| `LPSTR` / `char*` | `string` | Must specify ANSI or UTF-8 encoding |
| `void*` | `void*` | |
| `DWORD` | `uint` | |

### Step 3: Write the Declaration

Given a C header:

```c
int32_t process_records(const Record* records, size_t count, uint32_t* out_processed);
```

**DllImport:**

```csharp
[DllImport("mylib")]
private static extern int ProcessRecords(
    [In] Record[] records, nuint count, out uint outProcessed);
```

**LibraryImport:**

```csharp
[LibraryImport("mylib")]
internal static partial int ProcessRecords(
    [In] Record[] records, nuint count, out uint outProcessed);
```

Calling conventions only need to be specified when targeting Windows x86 (32-bit), where `Cdecl` and `StdCall` differ. On x64, ARM, and ARM64, there is a single calling convention and the attribute is unnecessary.

**Agent behavior:** If you detect that Windows x86 is a target — through project properties (e.g., `<PlatformTarget>x86</PlatformTarget>`), runtime identifiers (e.g., `win-x86`), build scripts, comments, or developer instructions — flag this to the developer and recommend explicit calling conventions on all P/Invoke declarations.

```csharp
// DllImport (x86 targets)
[DllImport("mylib", CallingConvention = CallingConvention.Cdecl)]

// LibraryImport (x86 targets)
[LibraryImport("mylib")]
[UnmanagedCallConv(CallConvs = [typeof(CallConvCdecl)])]
```

### Step 4: Handle Strings Correctly

1. **Know what encoding the native function expects.** There is no safe default.
2. **Windows APIs:** Always call the `W` (UTF-16) variant when the function expects a wide string. The `A` variant needs a specific reason and explicit ANSI encoding. The `A` also supports UTF-8 on Windows 10 1903+ if the system code page is UTF-8, but relying on that is fragile and not recommended.
3. **Cross-platform C libraries:** Usually expect UTF-8.
4. **Specify encoding explicitly.** Never rely on `CharSet.Auto`.
5. **Never introduce `StringBuilder` for output buffers.** It has poor performance semantics and is not suitable for general-purpose string buffers.

```csharp
// DllImport — Windows API (UTF-16)
[DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
private static extern int GetModuleFileNameW(
    IntPtr hModule, [Out] char[] filename, int size);

// DllImport — Cross-platform C library (UTF-8)
[DllImport("mylib")]
private static extern int SetName(
    [MarshalAs(UnmanagedType.LPUTF8Str)] string name);

// LibraryImport — UTF-16
[LibraryImport("kernel32", StringMarshalling = StringMarshalling.Utf16,
    SetLastPInvokeError = true)]
internal static partial int GetModuleFileNameW(
    IntPtr hModule, [Out] char[] filename, int size);

// LibraryImport — UTF-8
[LibraryImport("mylib", StringMarshalling = StringMarshalling.Utf8)]
internal static partial int SetName(string name);
```

**String lifetime warning:** Marshalled strings are freed after the call returns. If native code stores the pointer (instead of copying), you must agree on the allocator and the lifetime must be manually managed. On Windows or when targeting .NET Framework the COM related `CoTaskMemAlloc`/`CoTaskMemFree` should be the first choice for cross-boundary ownership, but the library may have its own allocator that must be used instead. On non-Windows target, using the `NativeMemory` APIs are the best option for cross-boundary ownership.

### Step 5: Establish Memory Ownership

When memory crosses the boundary, exactly one side must own it — and both sides must agree.

**Model 1 — Caller allocates, caller frees (safest):**

```csharp
[LibraryImport("mylib")]
private static partial int GetName(
    Span<byte> buffer, nuint bufferSize, out nuint actualSize);

public static string GetName()
{
    Span<byte> buffer = stackalloc byte[256];
    int result = GetName(buffer, (nuint)buffer.Length, out nuint actualSize);
    if (result != 0) throw new InvalidOperationException($"Failed: {result}");
    return Encoding.UTF8.GetString(buffer[..(int)actualSize]);
}
```

**Model 2 — Callee allocates, caller frees (common in Win32):**

```csharp
[LibraryImport("mylib")]
private static partial IntPtr GetVersion();
[LibraryImport("mylib")]
private static partial void FreeString(IntPtr s);

public static string GetVersion()
{
    IntPtr ptr = GetVersion();
    try { return Marshal.PtrToStringUTF8(ptr) ?? throw new InvalidOperationException(); }
    finally { FreeString(ptr); } // Must use the library's own free function
}
```

**Critical rule:** Always free with the matching allocator. Never use `Marshal.FreeHGlobal` or `Marshal.FreeCoTaskMem` on `malloc`'d memory — they use different heaps.

**Model 3 — Handle-based (callee allocates, callee frees):** Use `SafeHandle` (see Step 6).

**Pinning managed objects** — when native code stores the pointer or runs asynchronously:

```csharp
// Synchronous: use fixed
public static unsafe void ProcessSync(byte[] data)
{
    fixed (byte* ptr = data) { ProcessData(ptr, (nuint)data.Length); }
}

// Asynchronous: use GCHandle
var gcHandle = GCHandle.Alloc(data, GCHandleType.Pinned);
// Must keep pinned until native processing completes, then call gcHandle.Free()
```

### Step 6: Use SafeHandle for Native Handles

Raw `IntPtr` leaks on exceptions and has no double-free protection. `SafeHandle` is non-negotiable.

```csharp
internal sealed class MyLibHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    private MyLibHandle() : base(ownsHandle: true) { }

    [LibraryImport("mylib", StringMarshalling = StringMarshalling.Utf8)]
    private static partial MyLibHandle CreateHandle(string config);

    [LibraryImport("mylib")]
    private static partial int UseHandle(MyLibHandle h, ReadOnlySpan<byte> data, nuint len);

    [LibraryImport("mylib")]
    private static partial void DestroyHandle(IntPtr h);

    protected override bool ReleaseHandle() { DestroyHandle(handle); return true; }

    public static MyLibHandle Create(string config)
    {
        var h = CreateHandle(config);
        if (h.IsInvalid) throw new InvalidOperationException("Failed to create handle");
        return h;
    }

    public int Use(ReadOnlySpan<byte> data) => UseHandle(this, data, (nuint)data.Length);
}

// Usage: SafeHandle is IDisposable
using var handle = MyLibHandle.Create("config=value");
int result = handle.Use(myData);
```

### Step 7: Handle Errors

```csharp
// Win32 APIs — check SetLastError
[LibraryImport("kernel32", SetLastPInvokeError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
internal static partial bool CloseHandle(IntPtr hObject);

if (!CloseHandle(handle))
    throw new Win32Exception(Marshal.GetLastPInvokeError());

// HRESULT APIs
int hr = NativeDoWork(context);
Marshal.ThrowExceptionForHR(hr);
```

### Step 8: Handle Callbacks (if needed)

**Preferred (.NET 5+): `UnmanagedCallersOnly`** — avoids delegates entirely, so there is no GC lifetime risk:

```csharp
// C: typedef void (*log_callback)(int level, const char* message);
// C: void set_log_callback(log_callback cb);

[UnmanagedCallersOnly]
private static void LogCallback(int level, IntPtr message)
{
    string msg = Marshal.PtrToStringUTF8(message) ?? string.Empty;
    Console.WriteLine($"[{level}] {msg}");
}

// Pass the function pointer directly — no delegate, no GC concern
[LibraryImport("mylib")]
private static unsafe partial void SetLogCallback(
    delegate* unmanaged<int, IntPtr, void> cb);

// Usage:
unsafe { SetLogCallback(&LogCallback); }
```

The method must be `static`, must not throw exceptions back to native code, and can only use blittable parameter types. `UnmanagedCallersOnly` methods cannot be called from managed code directly.

**Fallback (older TFMs or when instance state is needed): delegate with rooting**

```csharp
// C: typedef void (*log_callback)(int level, const char* message);
[UnmanagedFunctionPointer(CallingConvention.Cdecl)] // Only needed on Windows x86
private delegate void LogCallbackDelegate(int level, IntPtr message);

// CRITICAL: prevent delegate from being garbage collected
private static LogCallbackDelegate? s_logCallback;

public static void EnableLogging(Action<int, string> handler)
{
    s_logCallback = (level, msgPtr) =>
    {
        string msg = Marshal.PtrToStringUTF8(msgPtr) ?? string.Empty;
        handler(level, msg);
    };
    SetLogCallback(s_logCallback);
}
```

If native code stores the function pointer, the delegate **must** stay rooted for its entire lifetime. A collected delegate means a crash.

---

## Blittable Structs

Blittable types have identical managed and native layouts — zero marshalling overhead.

**Blittable:** `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `nint`, `nuint`, and structs of only blittable fields. With `[assembly: DisableRuntimeMarshalling]`, `bool` (1 byte) and `char` (2 bytes, `char16_t`) are also treated as blittable.

**Not blittable (without `DisableRuntimeMarshalling`):** `bool`, `char`, `string`, `decimal`, anything with `MarshalAs`.

```csharp
[StructLayout(LayoutKind.Sequential)]
internal struct Vec3 { public float X, Y, Z; }

[LibraryImport("physics")]
internal static partial void TransformVectors(Span<Vec3> vectors, nuint count, in Vec3 t);
```

### Explicit Layout (Unions)

```csharp
// C: typedef union { int32_t i; float f; } Value;
[StructLayout(LayoutKind.Explicit, Size = 4)]
internal struct Value
{
    [FieldOffset(0)] public int I;
    [FieldOffset(0)] public float F;
}
```

### Packing

If the native struct uses non-default packing, match it:

```csharp
// C: #pragma pack(push, 1)
[StructLayout(LayoutKind.Sequential, Pack = 1)]
internal struct PackedHeader
{
    public byte Magic;
    public uint Size;    // At offset 1, not 4
    public ushort Flags; // At offset 5, not 8
}
```

---

## Cross-Platform Library Loading

Use `NativeLibrary.SetDllImportResolver` for complex scenarios, or conditional compilation for simple cases. Use `CLong`/`CULong` for C `long`/`unsigned long` — the size differs between Windows (32-bit) and 64-bit Unix (64-bit). Note: `CLong`/`CULong` with `LibraryImport` requires `[assembly: DisableRuntimeMarshalling]`; with `DllImport` this is not needed.

```csharp
// Simple: conditional compilation
// WINDOWS, LINUX, MACOS are predefined only when targeting an OS-specific TFM
// (e.g., net8.0-windows). For portable TFMs (e.g., net8.0), these symbols are
// not defined — use the runtime resolver approach below instead.
#if WINDOWS
    private const string LibName = "mylib.dll";
#elif LINUX
    private const string LibName = "libmylib.so";
#elif MACOS
    private const string LibName = "libmylib.dylib";
#endif

// Complex: runtime resolver
NativeLibrary.SetDllImportResolver(typeof(MyLib).Assembly,
    (name, assembly, searchPath) =>
    {
        if (name != "mylib") return IntPtr.Zero;
        string libName = RuntimeInformation.IsOSPlatform(OSPlatform.Windows)
            ? "mylib.dll"
            : RuntimeInformation.IsOSPlatform(OSPlatform.OSX)
                ? "libmylib.dylib" : "libmylib.so";
        NativeLibrary.TryLoad(libName, assembly, searchPath, out var handle);
        return handle;
    });
```

---

## Migrating DllImport to LibraryImport

For codebases targeting .NET 7+, migrating provides AOT compatibility and trimming safety.

1. Add `partial` to the containing class and make the method `static partial`
2. Replace `[DllImport]` with `[LibraryImport]`
3. Replace `CharSet` with `StringMarshalling`
4. Replace `SetLastError = true` with `SetLastPInvokeError = true`
5. Remove `CallingConvention` unless targeting Windows x86
6. Build and fix `SYSLIB1054`–`SYSLIB1057` analyzer warnings

Enable the interop analyzers in your project:

```xml
<PropertyGroup>
    <EnableTrimAnalyzer>true</EnableTrimAnalyzer>
    <EnableAotAnalyzer>true</EnableAotAnalyzer>
</PropertyGroup>
```

---

## Tooling

### CsWin32 (Win32 APIs)

For Win32 P/Invoke, prefer [Microsoft.Windows.CsWin32](https://github.com/microsoft/CsWin32) over hand-written signatures. It source-generates correct, `LibraryImport`-compatible declarations from metadata — eliminating the most common signature bugs.

```bash
dotnet add package Microsoft.Windows.CsWin32
```

Create a `NativeMethods.txt` file in your project root listing the APIs you need, one per line:

```text
CreateFile
ReadFile
CloseHandle
```

The generator produces correct signatures including `SafeHandle` wrappers, correct struct layouts, and proper `SetLastError` usage. No manual type mapping required.

### CsWinRT (WinRT APIs)

For WinRT interop, use [Microsoft.Windows.CsWinRT](https://github.com/microsoft/CsWinRT). It generates .NET projections from Windows Runtime metadata (`.winmd` files), providing type-safe access to WinRT APIs without manual interop code.

```bash
dotnet add package Microsoft.Windows.CsWinRT
```

---

## Validation

### Review checklist

- [ ] Every signature matches the native header exactly (types, sizes)
- [ ] Calling convention specified if targeting Windows x86; omitted otherwise
- [ ] String encoding is explicit — no reliance on defaults or `CharSet.Auto`
- [ ] Memory ownership is documented and matched (who allocates, who frees, with what)
- [ ] `SafeHandle` used for all native handles (no raw `IntPtr` escaping the interop layer)
- [ ] Delegates passed as callbacks are rooted to prevent GC collection
- [ ] `SetLastError`/`SetLastPInvokeError` set for APIs that use OS error codes
- [ ] Struct layout matches native (packing, alignment, field order)
- [ ] `CLong`/`CULong` used for C `long`/`unsigned long` in cross-platform code
- [ ] If using `CLong`/`CULong` with `LibraryImport`, `[assembly: DisableRuntimeMarshalling]` is applied
- [ ] No `bool` without explicit `MarshalAs` (unless `DisableRuntimeMarshalling` is applied, where `bool` is blittable as 1 byte and `MarshalAs` is unnecessary)

### Runnable validation steps

1. **Build with interop analyzers enabled** — confirm zero `SYSLIB1054`–`SYSLIB1057` warnings:
   ```xml
   <EnableTrimAnalyzer>true</EnableTrimAnalyzer>
   <EnableAotAnalyzer>true</EnableAotAnalyzer>
   ```
2. **Verify struct sizes match** — for every struct crossing the boundary, assert `Marshal.SizeOf<T>()` equals the native `sizeof`
3. **Round-trip test** — call the native function with known inputs and verify expected outputs
4. **Test with non-ASCII strings** — pass strings containing characters outside the ASCII range to confirm encoding is correct

## Common Pitfalls

| Pitfall | Impact | Solution |
|---------|--------|----------|
| `int` for `size_t` | Stack corruption on 64-bit | Use `nuint` |
| `long` for C `long` | Wrong on Windows (32-bit) | Use `CLong` / `CULong` (with `LibraryImport`, requires `[assembly: DisableRuntimeMarshalling]`) |
| `bool` without `MarshalAs` | Wrong marshal size | Specify `UnmanagedType.Bool` (4B) or `U1` (1B) |
| Implicit string encoding | Corrupts non-ASCII | Always specify `CharSet` or `StringMarshalling` |
| Wrong allocator for free | Heap corruption | Use the library's own free function |
| Raw `IntPtr` for handles | Leaks on exception | Use `SafeHandle` subclass |
| Delegate callback GC'd | Crash in native code | Keep a rooted reference for the delegate's lifetime |
| Missing `SetLastError` | Stale error codes | Set `SetLastError = true` on Win32 APIs |
| Struct packing mismatch | Fields at wrong offsets | Match `Pack` to native `#pragma pack` |
| Managed object as `void*` | Object moves during GC | Pin with `GCHandle` or `fixed` |

## Failure Modes and Recovery

| Symptom | Likely Cause | Diagnosis |
|---------|-------------|-----------|
| `DllNotFoundException` | Library not found at runtime | Check library name, path, and platform. Use `NativeLibrary.TryLoad` to test loading manually. On Linux, verify `LD_LIBRARY_PATH` or `rpath`. |
| `EntryPointNotFoundException` | Export name mismatch | Inspect the native binary's export table (`dumpbin /exports` on Windows, `nm -D` on Linux). Check for name mangling (C++ without `extern "C"`). |
| `AccessViolationException` | Signature mismatch, use-after-free, or missing pinning | Compare managed and native signatures byte-for-byte. Check struct sizes with `Marshal.SizeOf<T>()` vs native `sizeof`. Verify memory lifetime. |
| Silent data corruption | Wrong type size or encoding | Add temporary logging at the boundary. Compare `Marshal.SizeOf<T>()` to native struct size. Test with known input/output pairs. |
| Intermittent crashes | GC moved an unpinned object or collected a delegate | Ensure callbacks are rooted. Use `GCHandle` or `fixed` for any pointer held across calls. Run under a debugger with managed debugging assistants (MDAs) enabled. |
| Heap corruption on free | Wrong allocator | Confirm which allocator the native side used and free with the matching function. Never mix `malloc`/`free` with `CoTaskMemAlloc`/`CoTaskMemFree` or `Marshal.FreeHGlobal`. |

**General debugging approach:**

1. Reproduce under a debugger with native and managed debugging enabled
2. On .NET 5+, set `COMPlus_EnableDiagnostics=1` and use dotnet-dump or dotnet-trace for post-mortem analysis
3. Verify struct layout: `Marshal.SizeOf<T>()` must equal the native `sizeof` for every struct crossing the boundary
4. (.NET Framework only) Enable [Managed Debugging Assistants](https://learn.microsoft.com/en-us/dotnet/framework/debug-trace-profile/diagnosing-errors-with-managed-debugging-assistants) (MDAs) for `pInvokeStackImbalance` and `invalidOverlappedToPinvoke`

## Resources

- [P/Invoke](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke)
- [LibraryImport source generation](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke-source-generation)
- [Type marshalling](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/type-marshalling)
- [SafeHandle](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.safehandle)
- [NativeLibrary](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.nativelibrary)
- [Best practices](https://learn.microsoft.com/en-us/dotnet/standard/native-interop/best-practices)
