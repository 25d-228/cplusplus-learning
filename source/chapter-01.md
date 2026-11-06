# Chapter 1 — From C files to a native C++ program

Strata will eventually have separate implementations for files, indexes, recovery,
query operators, and scheduling. Before those parts can cooperate, their declarations
must agree, their object files must be linked, and a failed check must actually
make the program fail. This chapter establishes that working cycle with an
accumulation function small enough to inspect one statement at a time.

The program summarizes three unsigned integers. It returns a count and a sum;
`{10, 20, 12}` produces `count=3 sum=42`. An empty input produces two zeros.
There is no engine code to start from. Basic C functions, arrays, pointers, and
shell commands are sufficient prerequisites.

> **Learning objectives**
>
> Separate a header declaration from a function definition and a linked program;
> use a namespace and an initialized aggregate; explain a pointer's valid range;
> select and probe a native C++20 toolchain; build and test with CMake/CTest;
> distinguish a compilation error, a linkage error, and an incorrect result;
> inspect a live call with LLDB.

The implementation allocation is **2.0 hours**, including the small experiments:
0.5 for tool observation and probes, 0.5 for the program, 0.5 for CMake/CTest,
and 0.5 for debugger and failure diagnosis. Reading is separate. An already
configured machine may make setup shorter; a long explanation is not additional
coding time.

## Observe the tools before changing them

Use a native terminal on the M4 Mac. Run these observation commands; none changes
the selected developer tools or installs software:

```sh title="Terminal — observe the native environment"
sw_vers
uname -m
xcode-select --print-path
xcrun --find clang++
xcrun --show-sdk-path
xcrun clang++ --version
xcrun lldb --version
cmake --version
ctest --version
make --version
```

`sw_vers` reports the installed macOS rather than a version assumed by this book.
The native architecture should be `arm64`. An `x86_64` result on an Apple-silicon
machine is a reason to check whether the terminal is running under translation.
Use a native terminal and native tools for this course, then inspect the produced
executable as well; the shell's architecture alone does not prove the executable's
architecture.

`xcode-select` identifies the active developer directory. `xcrun` locates tools
and the software development kit, or SDK, associated with that selection. A
compiler path, compiler banner, standard-library implementation, and deployment
target are different observations. Apple's [C++ support page](https://developer.apple.com/xcode/cpp/)
distinguishes compiler features from library and operating-system support. A
compiler that recognizes `-std=c++20` has not thereby demonstrated every C++20
library facility.

When command-line developer tools are absent, Apple's installation entry point is:

```sh title="Terminal — only when developer tools are absent"
xcode-select --install
```

Complete the local installer, then repeat the observations. A working Command
Line Tools installation is sufficient for these command-line examples; a full
Xcode application is not a requirement. An existing compatible installation
should not be replaced just to match a version in a screenshot. Apple's
[command-line tools FAQ](https://developer.apple.com/library/archive/technotes/tn2339/_index.html)
documents installation and selection; its historical macOS examples are not this
course's minimum operating-system version.

CMake and CTest are separate from the C++ compiler. Use an installed maintained
release, or a macOS package compatible with the installed system from the
[official CMake download page](https://cmake.org/download/). If the application
bundle is installed at `/Applications/CMake.app`, its command-line programs can
be made visible in the current shell with:

```sh title="Terminal — for a CMake application installed at this path"
export PATH="/Applications/CMake.app/Contents/bin:$PATH"
cmake --version
ctest --version
```

This chapter's script declares **CMake 3.20** as its compatibility floor because
its commands use `ctest --test-dir`, [introduced in that release](https://cmake.org/cmake/help/latest/manual/ctest.1.html#cmdoption-ctest-test-dir).
That is a script-interface requirement, not a recommendation to install an old
release. Use compatible installed tools rather than matching a particular compiler
version banner. The actual language/library probes below are the compatibility
check.

Select the compiler executable once in this shell:

```sh title="Terminal — use the selected Apple C++ compiler"
CXX="$(xcrun --find clang++)"
"$CXX" --version
```

`$(...)` substitutes the output of a command. Quoting `"$CXX"` treats the resulting
path as one command name even if it contains spaces. `CXX` holds an executable
path, not a string of compiler flags. A maintained alternative macOS LLVM
installation is also usable: assign its actual `clang++` path, retain a compatible
SDK/library pairing, and run the same probes. Do not mix headers from one
installation with runtime libraries from another to silence an error. Selecting a
different compiler later calls for a separate CMake build directory.

## A small source tree with a real boundary

Work in the learner workspace, normally on `work/strata`, not inside the generated
website. Publication and remote repository access are unnecessary for this
exercise: an ordinary local directory is enough. All terminal commands below
assume the **learner workspace root** as the current directory.

Create only this chapter's directory:

```sh title="Terminal — create the exercise directory"
mkdir -p exercises/chapter-01
```

By the end of the chapter the source tree will be:

```text title="Files written in this chapter"
CMakeLists.txt
exercises/
  chapter-01/
    stats.hpp
    stats.cpp
    main.cpp
    cxx20_probe.cpp
```

Generated objects, executables, and CMake state go under `build/`, not beside the
source files. Add `/build/` to an existing workspace `.gitignore` if it is not
already ignored; do not replace unrelated ignore rules. No later chapter's
source file is needed by the targets introduced here.

The dependency is deliberately simple. `main.cpp` calls a function whose type it
learns from `stats.hpp`. `stats.cpp` supplies the function body. The C++ compiler
can compile either source independently once its included declarations are
available. Linking makes the resulting pieces into one executable.

## Describe the result before writing the loop

Create the following **complete header**, `exercises/chapter-01/stats.hpp`:

```cpp title="exercises/chapter-01/stats.hpp"
#ifndef STRATA_EXERCISES_CHAPTER_01_STATS_HPP
#define STRATA_EXERCISES_CHAPTER_01_STATS_HPP

#include <cstdint>

namespace strata {

struct Stats {
    std::uint32_t count = 0;
    std::uint64_t sum = 0;
};

// For nonzero length, values must name at least length readable elements.
// A null pointer is permitted only when length is zero.
Stats summarize(const std::uint32_t* values, std::uint32_t length);

} // namespace strata

#endif
```

`#ifndef`, `#define`, and `#endif` form an *include guard*. The first inclusion
in a translation unit defines the guard's name; another inclusion in that same
unit skips the guarded text. It prevents repeated definitions of the structure
within that unit. It does not arrange linking or prevent every cross-file
one-definition error.

The standard header `<cstdint>` supplies fixed-width integer type names in
namespace `std`. Here `std::uint32_t` is an unsigned 32-bit integer and
`std::uint64_t` an unsigned 64-bit integer. These exact-width types are available
on the target toolchains; the [header specification](https://eel.is/c++draft/cstdint.syn)
marks them optional on implementations that cannot supply the corresponding
representation. The feature probe will also check the byte sizes we use.

`namespace strata { ... }` groups these declarations. Outside that namespace,
write `strata::Stats` and `strata::summarize`; `::` selects a name from its scope.
A namespace neither allocates an object nor creates a runtime wrapper. Inside
its body, the short name `Stats` already refers to this declaration. Avoid a
header-level `using namespace std;`: a header should not inject an entire
library's names into every source file that includes it.

A `struct` introduces a type; `Stats` can be used as a type name without C's
`struct Stats` spelling or a separate `typedef`. Its members are public. This
particular structure is an *aggregate*: brace initialization can give values to
its members in declaration order. The `= 0` on each member provides a default
initializer, so a member omitted from an aggregate initializer starts at zero.
The [aggregate initialization rules](https://eel.is/c++draft/dcl.init.aggr)
specify that behavior.

These are examples of declarations, not additional required file contents:

```cpp title="Reading the initialization syntax"
strata::Stats empty{};          // count = 0, sum = 0
strata::Stats expected{3, 42};  // count = 3, sum = 42
const strata::Stats fixed{1, 7};
```

The `const` object `fixed` cannot have its fields changed through ordinary
assignment. `Stats empty;` would also use these particular member initializers;
we write braces to make initialization conspicuous. This does **not** mean an
arbitrary local integer such as `int x;` is automatically initialized to zero.

The line ending in `summarize(...);` is a function **declaration**. It specifies
the return type and parameter types, enough for a caller to be type-checked.
It contains no executable loop. A **definition** will provide the body.

### What the pointer promises, and what it cannot prove

The parameter `const std::uint32_t* values` points to integers that this function
may read but not modify through that pointer. It does not own those integers.
The caller must keep the array alive for the call and supply a length that fits
that actual array. Passing an address does not copy the array.

For a positive `length`, the readable range is `values[0]` through
`values[length - 1]`. A null pointer is accepted for an empty range, where the
function must perform no dereference and no pointer arithmetic. A null pointer
with a positive length, or a length longer than the allocation, violates the
interface's precondition. Raw pointers carry no allocation-length information
that this function could use to validate that claim. We will not manufacture
such an invalid access as a normal test.

This is an internal range interface, not a parser for untrusted input. Chapter 2
will place pointer and length together in a view and teach the associated
lifetime rules. A view still cannot make dead storage live again.

### Choose a numeric range that makes the loop exact

This foundation function deliberately limits both an element and the element
count to 32 unsigned bits, while accumulating into 64 bits. It accepts no
negative values. This narrow domain lets us prove that every permitted sum fits,
instead of adding an exception or overflow-status mechanism before those have
been taught.

Let $n$ be the element count, $x_i$ the value at zero-based index $i$, and $S_k$
the sum after consuming $k$ elements. The result we want is

$$
S_0 = 0, \qquad S_{k+1} = S_k + x_k, \qquad
S_n = \sum_{i=0}^{n-1} x_i.
$$

Both $n$ and every $x_i$ are at most $2^{32}-1$. Values are nonnegative, so no
partial sum is greater than the final sum. Its maximum is bounded by

$$
S_n \leq (2^{32}-1)^2
     = 2^{64} - 2^{33} + 1
     < 2^{64}.
$$

An unsigned 64-bit integer represents zero through $2^{64}-1$, so the sum fits.
The count increments at most $n$ times and fits its own type. At the maximum
length, the loop index reaches `length` and stops; it never needs to increment
past that value. The [integer-type rules](https://eel.is/c++draft/basic.fundamental)
define unsigned arithmetic, but our bound avoids relying on wraparound.

This is a range proof, **not** a claim that a four-billion-element array fits
the machine. Actual arrays must still exist. Nor is it a general replacement
for checked arithmetic: the engine later accepts 64-bit amounts, for which a
64-bit accumulator needs explicit overflow checks. Do not widen this exercise's
input type or count type and silently keep the old proof.

## Turn the recurrence into one complete function

Create `exercises/chapter-01/stats.cpp`:

```cpp title="exercises/chapter-01/stats.cpp"
#include "stats.hpp"

namespace strata {

Stats summarize(const std::uint32_t* values, std::uint32_t length) {
    Stats result{};
    for (std::uint32_t i = 0; i < length; ++i) {
        result.sum += values[i];
        ++result.count;
    }
    return result;
}

} // namespace strata
```

Include the function's own header first. The compiler now sees its declaration
and definition together and can diagnose disagreements in this translation
unit. The namespace is reopened, so this body defines `strata::summarize`, not
an unrelated global function with a similar name.

`result` is a local value whose initialized fields hold the state. On entry to
iteration `i`, `result.count` equals `i` and `result.sum` equals $S_i$. The first
statement adds the current element; the second records that one element has
been consumed. At the next loop test both fields describe the same prefix.
Inside the body, between those two statements, the sum has advanced before the
count. That intermediate state is useful to recognize in the debugger.

The left operand of `+=` has type `std::uint64_t`; the 32-bit element is added in
the wider unsigned arithmetic type on the target. This matters. Computing a
32-bit addition first and only then assigning its result to a 64-bit field
would not repair an overflow that had already occurred.

For the three-element fixture, the concrete states are:

| Point in execution | Next index | Count | Sum |
|---|---:|---:|---:|
| Before the loop | 0 | 0 | 0 |
| After reading 10 | 1 | 1 | 10 |
| After reading 20 | 2 | 2 | 30 |
| After reading 12 | 3 | 3 | 42 |

Each array element occupies four bytes on the intended platform. If the first
one is at address $A$, element $i$ begins at $A + 4i$. `values[i]` expresses the
correct element step without handwritten byte-address arithmetic. The array
payload occupies 12 bytes; the two scalar result fields and any structure
padding are separate. We do not serialize the structure's object representation
or assume it has no padding.

For `length == 0`, the first comparison is false, the loop body never runs, and
`result` stays zero. This is why the empty case works with `nullptr`, C++'s null
pointer literal. No special zero-sized C array is needed.

`return result;` returns a **value**, not a pointer to a dead local object. The
caller obtains the two fields as its own result object; the implementation may
eliminate a physical copy. Neither field refers into the input array. There is
no dynamically allocated storage or file to clean up here, and the function
performs no I/O. All source-array storage stays with the caller.

## Make a wrong answer observable

Create `exercises/chapter-01/main.cpp` with the following first block. The next
block is appended to the **same file**; together they are its complete contents.

```cpp title="exercises/chapter-01/main.cpp"
#include "stats.hpp"

#include <cstdint>
#include <iostream>

bool check_stats(const char* name,
                 strata::Stats actual,
                 strata::Stats expected) {
    if (actual.count == expected.count && actual.sum == expected.sum) {
        return true;
    }
    std::cerr << "FAIL " << name
              << ": expected count=" << expected.count
              << " sum=" << expected.sum
              << ", got count=" << actual.count
              << " sum=" << actual.sum << '\n';
    return false;
}
```

The helper takes `actual` and `expected` **by value**. Each parameter is its own
small `Stats` object. It tests both fields; a correct sum with a wrong count is
not a pass. The `bool` return type has values `true` and `false`.

`<iostream>` declares standard streams. In this context `<<` inserts the next
value into the stream; it is not an integer bit shift. Chaining insertions
builds one diagnostic. `std::cerr` is the standard error stream, while normal
results go to `std::cout`. `"FAIL "` is a string literal and `'\n'` is a newline
character. `name` points to a caller-supplied string literal that remains alive;
the helper neither owns nor frees it.

Append this complete `main` function:

```cpp title="exercises/chapter-01/main.cpp"
int main() {
    const std::uint32_t values[] = {10, 20, 12};
    const strata::Stats actual = strata::summarize(values, 3);
    if (!check_stats("three values", actual, {3, 42})) {
        return 1;
    }

    const strata::Stats empty = strata::summarize(nullptr, 0);
    if (!check_stats("empty", empty, {0, 0})) {
        return 1;
    }

    const std::uint32_t zero[] = {0};
    if (!check_stats("one zero", strata::summarize(zero, 1), {1, 0})) {
        return 1;
    }

    const std::uint32_t wide[] = {4'294'967'295U, 4'294'967'295U};
    if (!check_stats("wide sum", strata::summarize(wide, 2),
                     {2, 8'589'934'590ULL})) {
        return 1;
    }

    std::cout << "count=" << actual.count << " sum=" << actual.sum << '\n';
    std::cout << "count=" << empty.count << " sum=" << empty.sum << '\n';
    std::cout << "all checks passed\n";
    return 0;
}
```

`main` is the program entry point. It remains in the global namespace. Its zero
return value means success; on the target platform the nonzero value `1` reports
failure to the calling shell and test runner. The logical-not operator `!` turns
a failed check into the branch that exits unsuccessfully.

At the first call, the array expression `values` converts to a pointer to its
first element. The `3` is a count of elements, not bytes. `const` prevents
accidental mutation of the local fixtures. The brace expression `{3, 42}` is
used to initialize the helper's `Stats` parameter: count first, sum second.
All arguments are evaluated before the function body runs, but this code does
not rely on an ordering between independent argument evaluations.

The apostrophes in `4'294'967'295` are digit separators for readability, not
characters in the integer's value. `U` requests an unsigned integer literal;
`ULL` requests an unsigned `long long` literal large enough for the expected
wide sum on the supported tools. The element value is $2^{32}-1$. Adding it
twice gives 8,589,934,590, which does not fit 32 bits but does fit 64. A mistakenly
narrow accumulator can pass the `42` fixture and still fail this one.

The one-zero fixture distinguishes “one element whose value is zero” from “no
elements.” Both sums are zero; their counts differ. The checks are ordinary
control flow, not a call hidden inside `assert`. Defining `NDEBUG` disables
[`assert` evaluation](https://eel.is/c++draft/assertions); it does not disable
these function calls or their failure returns. Assertions can document internal
invariants later, but required test work must remain observable in Release.

## Compile pieces, then link a program

A source file plus the text included into it, after preprocessing, forms the
input for a translation unit. In our conventional header-based build, each
`.cpp` is compiled separately. The header participates in both compilations;
it is not a third executable component. The language's
[separate-translation model](https://eel.is/c++draft/lex.separate) permits these
units to be translated and later linked.

For this program, the build has the following shape:

```text title="Two compilations and one link"
main.cpp  + stats.hpp + library declarations -> main.o
stats.cpp + stats.hpp                        -> stats.o
main.o + stats.o + standard-library support -> foundation-01
```

An *object file* contains generated code/data and information about symbols it
defines or still needs. The body of `strata::summarize` is in `stats.o`.
`main.o` contains a call that needs that definition. The C++ compiler driver
selects the appropriate standard-library and runtime linkage when it performs
the final link. Use `clang++`, not a direct invocation of `ld` or an improvised
list of C++ runtime libraries.

Run each command from the workspace root, stopping at an unexpected error:

```sh title="Terminal — separate native compilations"
mkdir -p build/manual

"$CXX" -std=c++20 -arch arm64 -O0 -g \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -I exercises/chapter-01 \
  -c exercises/chapter-01/stats.cpp -o build/manual/stats.o

"$CXX" -std=c++20 -arch arm64 -O0 -g \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -I exercises/chapter-01 \
  -c exercises/chapter-01/main.cpp -o build/manual/main.o

"$CXX" -arch arm64 build/manual/main.o build/manual/stats.o \
  -o build/manual/foundation-01

file build/manual/foundation-01
./build/manual/foundation-01
echo "program exit status=$?"
```

Here `-std=c++20` selects the language mode, `-arch arm64` selects the macOS
architecture, `-O0` disables ordinary optimization for this debugging build,
and `-g` emits debugging information. `-c` stops before linking, and `-o`
names the output file. The `-I` option adds a header search directory. Because
our quoted header is beside both source files, their local include search also
finds it; a future consumer in another directory will need the advertised include
path. The [Clang command guide](https://clang.llvm.org/docs/CommandGuide/clang.html#stage-selection-options)
describes the stage-selection switches.

The warning options ask the compiler to diagnose suspicious constructs,
nonstandard extensions, and conversions that can change a value or its sign.
They are diagnostic help, not a proof of memory safety or correctness. An
intentional conversion still needs an explanation rather than an added cast
whose only purpose is silencing a warning. We do not make every warning from
every compiler version a mandatory build failure.

`file` should identify a Mach-O executable for ARM64 on the target Mac. Exact
wording varies. The program itself should print exactly:

```text title="Expected program output"
count=3 sum=42
count=0 sum=0
all checks passed
```

The following shell line should report `program exit status=0`. `$?` is the
status of the **immediately preceding command**; running an unrelated command
first loses the status you intended to inspect. The printed results show the
two main fixtures, while the zero-value and wide-sum checks pass silently.

These are native macOS commands. On a separate Linux machine the C++ sources
also work with a compatible compiler, but Apple's `xcrun` and `-arch arm64`
options are not Linux setup instructions. A successful Linux execution is not
evidence of native Mac toolchain support.

### An include guard is not a linker fix

Now deliberately omit the object that contains `summarize`. Do **not** alter the
source files or the working executable:

```sh title="Terminal — a deliberately incomplete link"
if "$CXX" -arch arm64 build/manual/main.o \
    -o build/manual/missing-definition; then
  printf 'Unexpected success: inspect which objects were linked.\n'
else
  printf 'Expected link failure: summarize has no linked definition.\n'
fi
```

Compilation of `main.cpp` already succeeded: its header supplied a declaration
with the correct type. The final link cannot supply the called function's body.
On macOS, look for an undefined-symbol diagnostic mentioning `strata::summarize`;
the spelling of the integer types may reflect their underlying implementation
types. The test succeeds as an experiment when **linking fails for that missing
symbol**, not merely for any unrelated command error.

Repair the link by including `stats.o`, as in the successful command. Including
`stats.cpp` from `main.cpp` is not the repair. If that body is also compiled
separately, it can create a duplicate non-inline definition. Our intended
organization is one ordinary function declaration visible to callers and one
linked ordinary function definition. The [one-definition rules](https://eel.is/c++draft/basic.def.odr)
also permit matching class definitions in multiple translation units under
specified conditions; that is why a guarded structure definition belongs in
this header. Templates and inline functions have further rules, introduced when
we need them.

Similarly, writing the body outside `namespace strata` would define a different
function. A linker error is not an instruction to change `strata::summarize`'s
declaration until a differently named symbol happens to match. Compare the
intended signature, definition, namespace, and actual linked objects.

## Probe the language and library that later chapters need

The statistics function uses only a small portion of C++. It can compile even
when a later required library facility is missing. Before adopting the toolchain,
compile a separate **capability probe** for C++20 concepts and `std::span`.

Two small pieces of syntax appear here before their deeper treatment. In
`template <std::integral T>`, `T` names a type parameter and `std::integral`
restricts it to integer types. Calling `echo_integer(10)` selects the version
with `T` equal to `int`; its parameter and return value are both `int`. This is
a language constraint, not a runtime tag test.

`std::span<const int>` is a library type that refers to a contiguous range of
read-only integers without owning them. Its constructor can obtain the length
from the local array, `.size()` reports that length, and `[i]` reads an element.
In this probe the array stays alive for the entire use of the span. Chapter 2
will develop borrowing and invalidation; Chapter 5 will develop templates and
constraints. No engine design depends on learning those subjects from this probe
alone.

Create `exercises/chapter-01/cxx20_probe.cpp` with this first block, then append
the second block to the same file:

```cpp title="exercises/chapter-01/cxx20_probe.cpp"
#include <climits>
#include <concepts>
#include <cstdint>
#include <iostream>
#include <span>
#include <version>

static_assert(__cplusplus >= 202002L, "C++20 language mode is required");
static_assert(CHAR_BIT == 8, "The course byte layouts use 8-bit bytes");
static_assert(sizeof(void*) == 8, "Use a 64-bit target");
static_assert(sizeof(std::uint32_t) == 4);
static_assert(sizeof(std::uint64_t) == 8);
static_assert(std::integral<int>);
static_assert(!std::integral<double>);

template <std::integral T>
T echo_integer(T value) {
    return value;
}
```

`static_assert` asks the compiler to reject the translation unit when a condition
known at compile time is false. Unlike the runtime `assert` macro, it is not
disabled by `NDEBUG`. `sizeof` measures a type in C++ bytes; `CHAR_BIT`, from
`<climits>`, states how many bits a byte has. Checking both keeps the
four-byte/eight-byte claims explicit. A 64-bit pointer check does not distinguish
ARM64 from x86-64, so still inspect the executable's architecture.

`__cplusplus` is the language-mode macro. `202002L` is the C++20 marker; `L` is
an integer literal suffix, not a compiler version. The two `std::integral`
checks cover an accepted and a rejected type. The function definition remains
in this one source file, so no template-linkage issue is being concealed.

Append the complete entry point:

```cpp title="exercises/chapter-01/cxx20_probe.cpp"
int main() {
    const int values[] = {10, 20, 12};
    const std::span<const int> view{values};
    if (view.size() != 3 || echo_integer(view[1]) != 20 ||
        view[0] + view[1] + view[2] != 42) {
        std::cerr << "C++20 probe failed\n";
        return 1;
    }

    std::cout << "__cplusplus=" << __cplusplus << '\n';
#if defined(_LIBCPP_VERSION)
    std::cout << "standard library: libc++ " << _LIBCPP_VERSION << '\n';
#elif defined(__GLIBCXX__)
    std::cout << "standard library: libstdc++ " << __GLIBCXX__ << '\n';
#else
    std::cout << "standard library: vendor not identified by this probe\n";
#endif
    std::cout << "C++20 probe passed\n";
    return 0;
}
```

`||` is short-circuit logical OR: when the size test fails, the remaining
operands are not evaluated. With the actual fixture, the size is three and every
read is in bounds. `#if defined(...)` selects diagnostic output based on the
library's identification macro. These vendor numbers are observations, not a
cross-vendor compatibility scale. The code's successful compilation **and
execution** are more useful than comparing them to an arbitrary threshold.

Build and run the complete probe:

```sh title="Terminal — native feature probe"
"$CXX" -std=c++20 -arch arm64 -O0 -g \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  exercises/chapter-01/cxx20_probe.cpp -o build/manual/cxx20-probe
./build/manual/cxx20-probe
```

In explicit C++20 mode the language marker should be `202002`. The library line
depends on the installed toolchain; read the reported value rather than expecting
a fixed library version. The last line must be `C++20 probe passed`, and the process must
exit successfully. This exercises the facilities actually used here, not every
feature carrying the C++20 label. The [span overview](https://eel.is/c++draft/span.overview)
and [integral concept](https://eel.is/c++draft/concepts.arithmetic) are precise
references for the two probe mechanisms.

A missing `<span>` header, an unrecognized constrained template, a compile-time
assertion, and a runtime loader error point to different problems. Confirm the
selected compiler, language flag, SDK, library, and produced architecture before
changing versions. The probe intentionally does not require modules, C++23
`std::expected`, `std::format`, or `std::jthread`.

## Give the build graph to CMake

Repeating the manual commands explains the machinery, but hand-maintaining every
object dependency will become unreliable as the engine grows. CMake describes a
graph of *targets*. The build tool then compiles changed inputs and links their
consumers. CTest runs the registered executable tests; it is not the compiler.

Create the root `CMakeLists.txt`. The following two blocks, in order, are the
complete file. There is no generated file to edit and no future target to add.

```cmake title="CMakeLists.txt"
cmake_minimum_required(VERSION 3.20)
project(Strata LANGUAGES CXX)

set(CMAKE_CXX_EXTENSIONS OFF)
enable_testing()

add_library(foundation-01-stats STATIC
    exercises/chapter-01/stats.cpp
)
target_include_directories(foundation-01-stats PUBLIC
    "${CMAKE_CURRENT_SOURCE_DIR}/exercises/chapter-01"
)
target_compile_features(foundation-01-stats PUBLIC cxx_std_20)

add_executable(foundation-01 exercises/chapter-01/main.cpp)
target_link_libraries(foundation-01 PRIVATE foundation-01-stats)
add_test(NAME foundation-01 COMMAND foundation-01)
```

`project(... LANGUAGES CXX)` enables C++ toolchain detection. The first target
is a **static library**: an archive of compiled objects, not a running service
or a new implementation dependency. CMake compiles `stats.cpp` once into that
library and links its needed code into `foundation-01`.

`target_include_directories` advertises the header location. The quoted
`${CMAKE_CURRENT_SOURCE_DIR}` expands to the directory containing this root
CMake file; it does not depend on which directory the build tool uses later.
`PUBLIC` means the requirement applies both to this target and to targets that
consume it. Thus `foundation-01` acquires the include directory and the C++20
minimum through `target_link_libraries`. `PRIVATE` on that link says that the
executable uses the library without making a public interface promise to further
consumers.

CMake's [`cxx_std_20` meta-feature](https://cmake.org/cmake/help/latest/manual/cmake-compile-features.7.html#requiring-language-standards)
requests a compiler mode at least as new as C++20. It does not guarantee each
library facility; that is why the probe remains a target. `CMAKE_CXX_EXTENSIONS`
being `OFF` requests a non-extension language mode rather than, for example,
a GNU dialect. It is not a proof that every extension would be rejected.

[`add_test(NAME ... COMMAND ...)`](https://cmake.org/cmake/help/latest/command/add_test.html)
accepts an executable target name and substitutes its built path. Do not hard-code
an absolute binary path from this machine. `enable_testing()` makes the root
build tree testable. We use the process return status as the check, not a fragile
search for the words “passed” in its output.

Append the probe target and the warning policy:

```cmake title="CMakeLists.txt"
add_executable(cxx20-probe exercises/chapter-01/cxx20_probe.cpp)
target_compile_features(cxx20-probe PRIVATE cxx_std_20)
add_test(NAME cxx20-probe COMMAND cxx20-probe)

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    foreach(target IN ITEMS foundation-01-stats foundation-01 cxx20-probe)
        target_compile_options(${target} PRIVATE
            -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion
        )
    endforeach()
endif()
```

The probe does not call the statistics library, so it declares its own language
requirement. CMake's `foreach` iterates through target names; `${target}` expands
to the current one. The compiler-ID condition matches Apple Clang, upstream
Clang, and GNU for these warning switches. It does not claim to configure every
vendor's diagnostic interface. Options are attached to our targets rather than
injected into an unrelated dependency's build.

### Configure, build, then test

Use separate build trees for Debug and Release. A **single-configuration** generator
is selected explicitly here so binary locations and `CMAKE_BUILD_TYPE` have one
meaning. The supplied Apple command-line tools provide `make`; Ninja is a valid
alternative when installed, not a prerequisite.

```sh title="Terminal — native Debug build"
cmake -S . -B build/debug -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/debug --parallel 4
ctest --test-dir build/debug --output-on-failure --no-tests=error
./build/debug/foundation-01
file build/debug/foundation-01
```

`-S .` names the source tree and `-B build/debug` names an out-of-source build
tree. `-D` supplies a configuration value; the compiler path is recorded in the
CMake cache. `--parallel 4` permits up to four build jobs, not four runtime
threads in the program. `--no-tests=error` prevents an accidentally empty test
selection from masquerading as success. Read the result: this chapter registers
**two tests**, named `foundation-01` and `cxx20-probe`.

The expected CTest result is two passing tests. Use the program directly for its
exact three-line output, or request verbose test output:

```sh title="Terminal — inspect one test and the actual build commands"
ctest --test-dir build/debug -R '^foundation-01$' -V --no-tests=error
cmake --build build/debug --clean-first --verbose
```

The regular expression selects exactly the named test. The second command
cleans and rebuilds **generated outputs in this build tree**, not source files,
so the compile and link invocations are visible even after a successful build.
Inspect the compiler path, selected language mode, debug flags, architecture,
and static library on the link line. The
[CMake command-line manual](https://cmake.org/cmake/help/latest/manual/cmake.1.html)
describes source/build trees and build mode; the
[CTest manual](https://cmake.org/cmake/help/latest/manual/ctest.1.html) describes
selection and failure reporting.

Do not run CTest after a failed build and interpret an old executable's pass as
a new result. Build and test are distinct operations. In an automated shell
sequence, connect them with `&&` so testing happens only after a successful build.

Now configure a separate optimized tree:

```sh title="Terminal — native Release build"
cmake -S . -B build/release -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/release --parallel 4 && \
  ctest --test-dir build/release --output-on-failure --no-tests=error
./build/release/foundation-01
```

The same answers must pass. This is a correctness comparison, not a meaningful
performance benchmark of a three-element loop. The exact optimization switches
belong to the actual toolchain configuration; inspect a verbose build rather
than claiming that a configuration name proves a particular flag set.

Changing compiler or generator inside an existing cached build directory can
leave an invalid configuration. Use a newly named directory for that change.
An Xcode or Ninja Multi-Config generator is also usable, but its workflow uses
`--config Debug` when building and `-C Debug` when testing; its executable path
can include the configuration. Do not combine those path assumptions with the
single-configuration commands above.

## Follow the state in LLDB

The output verifies the result. A debugger lets us inspect *how* it was reached.
Use the Debug executable and keep its build directory and object files available.
Do not start with an optimized build where locals or entire calls may have been
eliminated.

Launch the debugger from the workspace root:

```sh title="Terminal — start LLDB on the Debug executable"
xcrun lldb ./build/debug/foundation-01
```

Enter the next commands at the LLDB prompt, not in the shell. In a fresh session,
the first breakpoint normally has ID 1. Use the actual ID LLDB reports if the
session already contains breakpoints.

```text title="LLDB — stop in the accumulation function"
breakpoint set --name strata::summarize
run
frame variable length
frame variable values[0] values[1] values[2]
thread backtrace
```

The first call should have `length = 3` and elements `10`, `20`, and `12`.
The backtrace should contain `strata::summarize` called from `main`. Addresses,
line numbers, and displayed underlying integer type names depend on the build;
they are not expected constants.

`frame variable` reads parameters and locals using debug information. It supports
simple member and subscript paths, so the pointer expression can be inspected
as three elements for this **known valid first call**. Do not use the same
three-element inspection at the empty-input call. A function breakpoint normally
stops before its first source statement; `result` is not meaningful until its
initialization has executed. The [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html#examining-stack-frame-state)
explains this inspection interface.

Place a second breakpoint at the addition itself. Disable the entry breakpoint
first so that subsequent tests do not repeatedly stop at function entry:

```text title="LLDB — stop before an addition"
breakpoint disable 1
breakpoint set -f stats.cpp -p 'result[.]sum'
continue
frame variable i length result
```

Here `-f` selects a source file and `-p` searches its lines with a regular
expression. `[.]` matches the literal dot in `result.sum`. The expression avoids
depending on the chapter's line numbers; it should match the one addition line
in `stats.cpp`. This [source-pattern breakpoint](https://lldb.llvm.org/use/map.html#set-a-breakpoint-by-regular-expression-on-source-file-contents)
should be breakpoint 2 in this fresh session. Check its resolved location before
continuing.

At the first hit, `i` is zero and both result fields are zero. The highlighted
addition has **not executed yet**. Step over it, inspect, then step over the count
increment:

```text title="LLDB — inspect the intermediate state"
thread step-over
frame variable result
thread step-over
frame variable result
```

After the addition, expect `sum = 10` and `count = 0`. After the count increment,
expect `sum = 10` and `count = 1`. This temporary mismatch is normal: the fields
advance in two source statements. No other thread reads them, and the function
returns only the finished value.

Continue to the next two hits of the addition breakpoint:

```text title="LLDB — inspect the remaining iterations"
continue
frame variable i result
continue
frame variable i result
breakpoint disable 2
continue
quit
```

Before the second addition, expect `i = 1`, `count = 1`, `sum = 10`. Before the
third, expect `i = 2`, `count = 2`, `sum = 30`. Disabling breakpoint 2 allows the
last addition and the remaining tests to finish. The normal output should appear
and the process should exit successfully. These are expected program states,
not a requirement that every debugger version print the same transcript.

If a breakpoint has no resolved location, confirm the executable path, spelling
of the namespace, and debug information. If a changed line does not match the
running program, leave LLDB, rebuild successfully, and launch the rebuilt binary.
Do not keep editing source while assuming a stopped old process has changed.
[LLDB's breakpoint troubleshooting](https://lldb.llvm.org/use/troubleshooting.html#file-and-line-breakpoints-are-not-getting-hit)
discusses debug information and source-path mismatches.

A launch failure, missing debugger runtime, or denied local debugging permission
is an environment problem to diagnose separately; it is not a passed walkthrough
or a failed arithmetic check. Use the installed developer tools and ordinary
local authorization flow. Do not disable operating-system security protections
as a general debugging recipe.

## Prove that the check can fail in Release

A test system that has only ever displayed green output has not yet demonstrated
its failure path. Make one temporary change in `main.cpp`: in
`check_stats("three values", actual, {3, 42})`, change **only the expected sum**
from `42` to `41`. Keep the function and input unchanged. Save the file.

Build the changed source and run just that test in both configurations:

```sh title="Terminal — the intentionally wrong expectation must fail"
cmake --build build/debug --parallel 4 && \
  ctest --test-dir build/debug -R '^foundation-01$' \
    --output-on-failure --no-tests=error

cmake --build build/release --parallel 4 && \
  ctest --test-dir build/release -R '^foundation-01$' \
    --output-on-failure --no-tests=error

./build/release/foundation-01
echo "program exit status=$?"
```

Both selected CTest runs must report a failure, and the direct executable must
return status `1`. CTest's own nonzero exit code need not equal the executable's
`1`. The executable's diagnostic should be:

```text title="Expected diagnostic for the intentionally wrong expectation"
FAIL three values: expected count=3 sum=41, got count=3 sum=42
```

This failure is useful precisely because the algorithm is still correct. We
changed the oracle and observed its control over the process status. The compiler
may optimize the implementation, but it cannot replace this required failure
with success while preserving the program's behavior. No `assert` macro
controls the check.

Restore the expected sum to `42`, rebuild both configurations, and run their full
test sets again. Both tests must return to passing before continuing. Do not
“repair” the deliberate failed expectation by changing the correct accumulator.

### Locate a failure at the right layer

These symptoms suggest different first investigations:

| Symptom | Likely boundary | First useful action |
|---|---|---|
| `stats.hpp` is not found | Header lookup | Check the filename, source location, and actual include search path. |
| A type or member is unknown | Compilation | Read the first diagnostic and compare the included declaration with its use. |
| `strata::summarize` is undefined | Linkage | Confirm that the definition has the right namespace/signature and that its object or library is linked. |
| A duplicate definition is reported | Linkage | Look for a `.cpp` included as text and also compiled, or a non-inline function body repeated in headers. |
| The wide-sum fixture fails | Arithmetic | Inspect the accumulator's type and where widening occurs, not only the printed `42` example. |
| CTest finds no tests | Test discovery | Check the build-tree path and root `enable_testing()`; do not accept an empty run. |
| A known wrong expectation still passes | Build/test selection | Confirm the saved source, successful rebuild, chosen binary, and configuration. |
| LLDB cannot inspect useful locals | Debug configuration | Use the current Debug binary with debug information, and verify breakpoint/source locations. |

Do not begin with a blanket reinstall or deletion of every build directory.
Read the first meaningful error, identify the stage, and change the smallest
relevant input. A compiler's successful exit, a linker's successful exit, and a
test's successful exit answer three different questions.

## A legal subrange and an empty range

For the final variation, predict the result of starting at the **second** element
of `{10, 20, 12}` and consuming two elements. Insert these checks in `main`, after
the existing checks and before the successful `std::cout` output:

```cpp title="Variation — insert into main.cpp before the output"
if (!check_stats("suffix", strata::summarize(values + 1, 2), {2, 32})) {
    return 1;
}
if (!check_stats("empty with live pointer",
                 strata::summarize(values, 0), {0, 0})) {
    return 1;
}
```

`values + 1` advances by one element, or four bytes under the verified layout.
The available suffix is `{20, 12}`, so the result is count two and sum 32. The
second call has a non-null pointer but consumes no elements, again producing
zeros. The function's empty-range behavior depends on the count, not on the
pointer being null.

`values + 2` with length two would exceed this array. Do not execute that call
and hope a passing run validates it. The pointer/length precondition must be
established by the caller. Rebuild and run Debug and Release after adding the
legal variation; successful output remains the same. This small extension is
inside the chapter's implementation allocation.

## What carries forward

Keep the five source/build files, including the two variation checks. You now
have a repeatable native compilation, a static-library link boundary, an active
runtime check, a C++20/library probe, and a debugger path from an input pointer
to a returned value. No files, threads, or dynamic buffers are owned by the
accumulator; there is nothing for its caller to free.

The [roadmap](https://25d-228.github.io/cplusplus-learning/index.html) places
values, references, containers, and views next. That chapter will add a separate
exercise and retain these targets. Its starting code is exactly the five files
specified here, not a future storage engine or an unpublished helper library.

## Technical references

The mechanisms above are developed in the text; the references give precise
contracts and command details. The public C++ working draft evolves beyond
C++20, so only the facilities actually used here belong to this chapter's
baseline. A current documentation site's version label is not a tool-version
requirement.

- **Native environment:** [Apple C++ language support](https://developer.apple.com/xcode/cpp/),
  [Apple command-line tools FAQ](https://developer.apple.com/library/archive/technotes/tn2339/_index.html),
  and [CMake downloads](https://cmake.org/download/). The Apple FAQ is archived;
  its sample installation paths and old system versions are not prescribed here.
- **Translation and declarations:** [separate translation](https://eel.is/c++draft/lex.separate),
  [one-definition rules](https://eel.is/c++draft/basic.def.odr), and
  [aggregate initialization](https://eel.is/c++draft/dcl.init.aggr).
- **Types and checks:** [`<cstdint>`](https://eel.is/c++draft/cstdint.syn),
  [integer ranges](https://eel.is/c++draft/basic.fundamental),
  [assertion disabling](https://eel.is/c++draft/assertions),
  [span ownership](https://eel.is/c++draft/span.overview), and
  [arithmetic concepts](https://eel.is/c++draft/concepts.arithmetic).
- **Build and test:** [Clang stage-selection options](https://clang.llvm.org/docs/CommandGuide/clang.html#stage-selection-options),
  [CMake language requirements](https://cmake.org/cmake/help/latest/manual/cmake-compile-features.7.html#requiring-language-standards),
  [extension mode](https://cmake.org/cmake/help/latest/prop_tgt/CXX_EXTENSIONS.html),
  [Apple architecture selection](https://cmake.org/cmake/help/latest/variable/CMAKE_OSX_ARCHITECTURES.html),
  [CMake command line](https://cmake.org/cmake/help/latest/manual/cmake.1.html),
  [`add_test`](https://cmake.org/cmake/help/latest/command/add_test.html), and
  [CTest command line](https://cmake.org/cmake/help/latest/manual/ctest.1.html).
- **Debugger:** [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html),
  [command map, including source-pattern breakpoints](https://lldb.llvm.org/use/map.html),
  and [breakpoint troubleshooting](https://lldb.llvm.org/use/troubleshooting.html).
