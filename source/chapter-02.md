# Chapter 2 — Values, references, containers, and views

A query operator will eventually receive a buffer, examine part of it, and hand
results to another operator. Before doing that, it must answer three questions:
who owns the elements, which access path may change them, and what operation
ends the validity of a borrowed range? A pointer and a length cannot answer all
three. Neither can a modern-looking type used without its lifetime contract.

This chapter makes those questions executable. A value parameter changes its own
copy of `{3, 5, 8}` without changing its caller. A mutable view changes the caller's
middle element to 7. A read-only view then observes a sum of 18. A growing vector
and a text field show why a view is neither an owner nor a snapshot.

**Starting code:** keep the complete five-file exercise from
[Chapter 1](https://25d-228.github.io/cplusplus-learning/chapter-01.html), including
its final subrange checks. Keep `foundation-01-stats`, `foundation-01`, and
`cxx20-probe` in the root CMake file. This chapter adds a separate executable; it
does not change those sources or require any engine code.

> **Learning objectives**
>
> Distinguish an owning value from a reference and a copied view; use `const`,
> `auto`, and range-based loops deliberately; explain array and vector storage;
> write mutable and read-only span interfaces; preserve an owner's lifetime;
> predict invalidation before changing a container; distinguish text length from
> null termination; use a focused diagnostic without putting undefined behavior
> into the normal test suite.

The implementation allocation is **2.0 hours**: 1.0 for the programs, 0.5 for the
interface variations, and 0.5 for invalidation diagnosis and tests. Reading is
separate. There is no speed target for these tiny fixtures.

## One normal program and two isolated diagnostics

Run commands from the learner workspace root, using the working native C++20
compiler from Chapter 1. In a new shell on the Mac, restore its selection:

```sh title="Terminal — prepare this chapter"
CXX="$(xcrun --find clang++)"
mkdir -p exercises/chapter-02
```

The new files are:

```text title="Files added or extended"
CMakeLists.txt                          append one executable and test
exercises/chapter-02/values.cpp         the normal, passing program
exercises/chapter-02/readonly_rejected.cpp  compile-only negative example
exercises/chapter-02/dangling_span.cpp   sanitizer-only negative example
```

Every C++ block labelled **`exercises/chapter-02/values.cpp`**, in reading order,
appends to that one file. Together they form its complete contents. Other labelled
files are independent. The negative examples are deliberately **not** sources of
`foundation-02` and are not ordinary CTest entries. No globbing of all `.cpp`
files should silently add them to the build.

## A fixed array is an owning value

`std::array<std::uint32_t, 3>` is a library type containing three unsigned 32-bit
integers. The angle brackets supply arguments to a library *class template*:
the element type and the element count. Using this existing type does not require
writing a template. Chapter 5 will develop that separate skill.

Unlike a C array parameter, a parameter of this `std::array` type does not
silently turn into a pointer. Passing it by value gives the function its own
three-element value. The count is part of the type, so an array of four elements
is a different type. The elements are contiguous, and the array's `.size()` is
always three; see the [array overview](https://eel.is/c++draft/array.overview).

Start the normal source file:

```cpp title="exercises/chapter-02/values.cpp"
#include <array>
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <limits>
#include <span>
#include <string>
#include <string_view>
#include <vector>

namespace strata {

using Triple = std::array<std::uint32_t, 3>;

bool check(bool condition, const char* name) {
    if (condition) {
        return true;
    }
    std::cerr << "FAIL " << name << '\n';
    return false;
}

Triple with_middle(Triple values, std::uint32_t replacement) {
    values[1] = replacement;
    return values;
}
```

`using Triple = ...;` gives the existing type a shorter name; it creates no
object and adds no runtime layer. An object declared as `Triple values{3, 5, 8}`
contains those three integers. `Triple zeros{}` instead initializes all three
to zero. Do not infer that a bare `Triple uninitialized;` initializes its integer
elements: keep initialization explicit.

The helper `check` is the same active-control-flow idea as Chapter 1. A failed
condition prints a label and returns `false`; later callers propagate that to
`main`. It is not disabled by a Release build. `<limits>` and the view/string
headers will be used below, after their interfaces are explained.

`with_middle` receives an owning value. Its assignment changes its **parameter**,
not the caller's array. Returning that array gives the caller an owning result,
not a pointer into a dead local object. For an input named `original`, the
semantic trace is:

| Moment | `original` | Function's value / returned value |
|---|---|---|
| Before the call | `{3, 5, 8}` | No parameter yet |
| Parameter initialized | `{3, 5, 8}` | `{3, 5, 8}` |
| After replacing index 1 | `{3, 5, 8}` | `{3, 99, 8}` |
| After return | `{3, 5, 8}` | Caller owns `{3, 99, 8}` |

This is an explanation of observable values, not a promise that machine code
performs two physical copies. An optimizer may remove unnecessary copying while
preserving those observations. Three copied integers are also a very different
cost from copying a million-element vector. Avoid turning “pass by value” into
either a universal mistake or a universal optimization.

## A reference names the caller's object

An `&` in a parameter declaration changes the relationship. `Triple&` is an
*lvalue reference*: it binds to an existing `Triple`, and uses of the parameter
access that same object. There is no separate array payload for the parameter.
A normal reference must refer to a live suitable object; it is not a nullable
optional argument. The [reference rules](https://eel.is/c++draft/dcl.ref) specify
the language relationship, without promising a particular reference representation.

`const Triple&` also refers to the existing array, but it disallows modifying
its elements **through that reference**. It does not promise that no other access
path can change a non-const owner. Append these functions:

```cpp title="exercises/chapter-02/values.cpp"
void set_middle(Triple& values, std::uint32_t replacement) {
    values[1] = replacement;
}

std::uint32_t read_middle(const Triple& values) {
    return values[1];
}
```

The call syntax is `set_middle(original, 7)`, not `set_middle(&original, 7)`.
The declaration supplies the reference binding. Inside the function, ordinary
`values[1]` accesses the caller's element; there is no explicit dereference.
This differs from a C pointer interface while preserving the ability to mutate
a caller-owned object.

`read_middle` returns an integer value. That returned integer does not borrow
from the array and remains usable after the array is gone. Returning a
`const Triple&` or a span would be a different lifetime promise.

### `auto` deduces a type, not an ownership policy

In a local declaration, `auto` asks the compiler to deduce the type from the
initializer. It does not mean dynamically typed, and it does not universally
mean reference. For the array `original`, the following spellings express
different relationships:

| Declaration | Relationship |
|---|---|
| `auto copy = original;` | A new `Triple`, with its own three integers |
| `auto& alias = original;` | A mutable reference to `original` |
| `const auto& observed = original;` | A read-only reference to `original` |
| `const auto copy = original;` | A new, non-mutable `Triple` value |

For these ordinary value initializers, plain `auto` does not carry over an
initializer's reference or its top-level `const`. The explicit `&` and `const`
in the declaration matter; the [placeholder deduction rules](https://eel.is/c++draft/dcl.spec.auto)
give the full specification. We are not using `decltype(auto)` or forwarding
references here.

Append a complete check of these relationships:

```cpp title="exercises/chapter-02/values.cpp"
bool check_values() {
    Triple original{3, 5, 8};
    const auto changed = with_middle(original, 99);
    if (!check(original == Triple{3, 5, 8}, "value parameter preserves caller") ||
        !check(changed == Triple{3, 99, 8}, "returned value owns changed array")) {
        return false;
    }
    std::cout << "values: caller middle=" << original[1]
              << " returned middle=" << changed[1] << '\n';

    auto detached = original;
    detached[0] = 50;
    if (!check(original[0] == 3 && detached[0] == 50, "auto value is independent")) {
        return false;
    }

    auto& alias = original;
    const auto& observed = original;
    set_middle(alias, 7);
    if (!check(read_middle(observed) == 7 && original[1] == 7,
               "const reference observes mutation through another alias")) {
        return false;
    }
    std::cout << "references: observed middle=" << read_middle(observed) << '\n';

    const Triple fixed{3, 5, 8};
    auto writable_copy = fixed;
    writable_copy[0] = 42;
    return check(fixed[0] == 3 && writable_copy[0] == 42,
                 "copying a const owner does not make the copy const");
}
```

`std::array` equality compares the elements, so `original == Triple{3, 5, 8}`
checks the complete fixture rather than just its middle value. `&&` requires
both conditions; `||` short-circuits the failure checks. Every subscript here
is within the known three-element array.

The crucial observation is that `observed` sees 7 even though it was declared
before the mutation. A const reference is not a captured historical value. Nor
is `const` a synchronization mechanism; this entire exercise is single-threaded.
In the engine, an immutable snapshot will require ownership and publication
rules in addition to read-only access syntax.

### Assignment through a reference does not rebind it

A reference is bound when it is initialized. Later assignment through it assigns
to its object; it does not select a new referent. The
[reference initialization rules](https://eel.is/c++draft/dcl.init.ref) make this
distinction explicit. Predict which array changes in the following variation,
then append the complete check:

```cpp title="exercises/chapter-02/values.cpp"
bool check_reference_assignment() {
    Triple left{3, 5, 8};
    Triple right{10, 20, 30};
    auto& alias = left;
    alias = right;
    alias[0] = 99;
    return check(left == Triple{99, 20, 30} && right == Triple{10, 20, 30},
                 "reference assignment changes its original referent");
}
```

After `alias = right`, `left` contains a copy of `right`. After the next line,
`left[0]` becomes 99. `alias` still names `left`. This matters when an API accepts
a reference to an owner: assigning a new owner value through that parameter can
replace the caller's contents, not merely change a local handle.

## A range loop still chooses between values and references

A range-based loop visits the elements of a range in order. In
`for (auto value : values)`, the loop variable is a new value for each iteration.
In `for (auto& value : values)`, it is a reference to the current element. The
colon separates the loop variable's declaration from the range expression.

Append this experiment:

```cpp title="exercises/chapter-02/values.cpp"
bool check_range_loops() {
    Triple values{3, 5, 8};
    std::uint64_t copied_sum = 0;
    for (auto value : values) {
        value += 1U;
        copied_sum += value;
    }
    if (!check(values == Triple{3, 5, 8} && copied_sum == 19,
               "range-for by value changes only each local copy")) {
        return false;
    }

    for (auto& value : values) {
        value += 1U;
    }
    std::uint64_t observed_sum = 0;
    for (const auto& value : values) {
        observed_sum += value;
    }
    if (!check(values == Triple{4, 6, 9} && observed_sum == 19,
               "range-for by reference changes the elements")) {
        return false;
    }
    std::cout << "range-for: copied sum=" << copied_sum
              << " mutated sum=" << observed_sum << '\n';
    return true;
}
```

The first loop produces local values 4, 6, and 9, but the array remains 3, 5,
and 8. The second loop writes 4, 6, and 9 into the array. The final loop borrows
each element read-only. `1U` is the unsigned literal one. These three small
values make every increment and sum exact; no overflow behavior is being used.

Copying a small integer for reading is normally appropriate. `const auto&` is
also a useful spelling for reading a larger record without copying it, but it
is not inherently faster for every type. More importantly, `auto` applied to an
element that is itself a view copies that view, not its referenced payload.
The type being copied determines the copy's meaning.

The [range-for specification](https://eel.is/c++draft/stmt.ranged) describes
iteration using a beginning position and an end position obtained before the
loop. An iterator is a position used to traverse a container; for contiguous
storage it behaves in many ways like a pointer, though its actual type need
not be a pointer. `end()` denotes the position after the final element and is
not itself an element. Do not structurally modify a vector inside a loop over
that vector: even an append without reallocation invalidates the old end
iterator. Element assignments, such as the second loop above, do not change
the range's size or storage.

## A span combines an address and an element count

`std::span<T>` describes a contiguous sequence of `T` objects owned elsewhere.
The spelling without a second template argument uses a count stored at runtime.
It is useful to reason about the descriptor as a start address plus an element
count, without assuming a particular binary representation or `sizeof(span)`.
The [span overview](https://eel.is/c++draft/span.overview) defines this non-owning
relationship.

Constructing `std::span<std::uint32_t>{owner}` from our array derives the range
from the array itself. It avoids manually repeating a possibly wrong count.
Copying the span copies the description, not the integers. Destroying the span
does not destroy its owner's elements. A span does not pin storage, extend an
owner's lifetime, or repair an invalid raw pointer supplied to a constructor.

The element type controls write access. `std::span<const std::uint32_t>` reads
integers but cannot assign them through its subscript. A mutable-element span
can be converted to a read-only-element span; the reverse conversion is not a
way to obtain write permission.

### Read the position of `const`

These types make two distinct choices:

| Type | Can the descriptor be assigned a different range? | Can an element be assigned through it? |
|---|---|---|
| `std::span<std::uint32_t>` | Yes | Yes |
| `const std::span<std::uint32_t>` | No | Yes |
| `std::span<const std::uint32_t>` | Yes | No |
| `const std::span<const std::uint32_t>` | No | No |

A const span is a fixed descriptor; it need not describe const elements. Span's
[subscript operation](https://eel.is/c++draft/span.elem) is const-qualified for
the descriptor while returning access to the declared element type. In contrast,
a const array or const vector gives read-only access to its owned elements.
Do not generalize a container's const behavior to every handle-like type.

For a read-only input range, pass `std::span<const T>` by value. A
`const std::span<T>&` only makes the descriptor read-only and still permits
writes to its elements. Passing a descriptor by reference is not necessary to
avoid copying the whole payload: that payload was never copied by span.

### Check an index before modifying an element

`std::size_t`, supplied by `<cstddef>`, is an unsigned size type. It is the
natural type for a span's element count and index. An index is not a byte offset.
For a valid index `i`, `values[i]` names one element. In the C++20 interface used
here, `operator[]` does not provide a portable checked-error result for an invalid
index. The current draft also describes newer facilities; this program does
not depend on later `span::at` or library hardening modes.

Append the complete mutation function:

```cpp title="exercises/chapter-02/values.cpp"
bool replace_at(std::span<std::uint32_t> values,
                std::size_t index,
                std::uint32_t replacement) {
    if (index >= values.size()) {
        return false;
    }
    values[index] = replacement;
    return true;
}
```

Its contract is precise: for a **live, valid input span**, success changes exactly
one element; an index at or beyond the end returns `false` and changes nothing.
An empty span rejects every index. The function does not retain the span after
return. The owner must remain alive, with this range valid, throughout the call.
The bounds check cannot establish whether already-dangling storage is alive.

The span parameter is copied by value. Assignment through its subscript still
reaches the caller's element, unlike assignment through the owning value parameter
in `with_middle`. This is why “value parameter” alone does not tell us whether
an operation affects caller-owned bytes.

### Preserve the accumulator's range proof

Chapter 1 used 32-bit elements and a 32-bit count to prove that a 64-bit sum fits.
A span's count is `std::size_t`, so adopting a span must not silently discard the
count bound. We will reject counts above the original bound and keep the output
unchanged on rejection. The return value reports success; a mutable reference
carries the numeric result. This also distinguishes an empty sum of zero from
a rejected call.

`std::numeric_limits<T>::max()` returns the greatest representable value of the
numeric type `T`; see [numeric limits](https://eel.is/c++draft/numeric.limits.members).
Here `::max()` calls a type-associated function; no limit object is needed. Append
the reader and its complete test helper:

```cpp title="exercises/chapter-02/values.cpp"
bool sum_values(std::span<const std::uint32_t> values,
                std::uint64_t& result) {
    if (values.size() > std::numeric_limits<std::uint32_t>::max()) {
        return false;
    }
    std::uint64_t total = 0;
    for (const auto value : values) {
        total += value;
    }
    result = total;
    return true;
}

bool check_sum(std::span<const std::uint32_t> values,
               std::uint64_t expected,
               const char* name) {
    std::uint64_t actual = 0;
    if (!sum_values(values, actual)) {
        std::cerr << "FAIL " << name << ": element count rejected\n";
        return false;
    }
    if (actual == expected) {
        return true;
    }
    std::cerr << "FAIL " << name << ": expected sum=" << expected
              << ", got sum=" << actual << '\n';
    return false;
}
```

Let $n$ be the accepted count and $x_i$ the unsigned value at index $i$. The
loop's accumulator after $k$ iterations is the sum of the first $k$ elements.
The guard enforces $n \leq 2^{32}-1$, and the element type enforces
$x_i \leq 2^{32}-1$. Therefore

$$
0 \leq \sum_{i=0}^{n-1} x_i \leq (2^{32}-1)^2 < 2^{64}.
$$

All terms are nonnegative, so every partial sum also fits. `total` is already
64-bit when an element is added. A failed count check returns before assigning
`result`. An accepted empty range performs no reads and assigns zero. Read-only
here means the **input elements** cannot be changed through this interface;
the separate result reference is intentionally mutable.

This is a bounded 32-bit-element exercise, not the later engine's general
64-bit-amount aggregate. Do not claim to exercise the large-count rejection by
constructing a fake span over a tiny array: the span would already violate its
valid-range precondition. The guard is directly inspectable; the runnable
fixtures test empty, ordinary, and wide-element sums using real storage.

### Slicing changes the description, not the owner

A span's `subspan(offset, count)` returns a new view. Let $n$ be the original
length, $o$ the offset, and $c$ the requested length, all measured in elements.
The required inequalities are

$$
o \leq n, \qquad c \leq n-o.
$$

Using `c <= n - o` **after** checking `o <= n` avoids an overflowing `o + c`
validation expression. It also prevents unsigned subtraction from wrapping.
For `{3, 7, 8}`, choosing $o=1$ and $c=2$ describes `{7, 8}`. Its sum is 15.
Choosing $o=2$ and $c=2$ violates the second inequality and must not be executed.
At $o=n$, the only permitted count is zero. The
[subview specification](https://eel.is/c++draft/span.sub) gives this contract;
`subspan` is not a replacement for validating untrusted offsets.

On the verified four-byte element layout, a view starting at address $A$ with
length three describes elements at $A$, $A+4$, and $A+8$. The suffix starts at
$A+4$ and contains two elements. It does not allocate another eight-byte payload.
Neither descriptor owns those bytes.

Append the complete span exercise:

```cpp title="exercises/chapter-02/values.cpp"
bool check_spans() {
    Triple owner{3, 5, 8};
    const std::span<std::uint32_t> writable{owner};
    const std::span<const std::uint32_t> read_only{writable};
    const auto copied_view = writable;
    if (!check(copied_view.data() == writable.data() && copied_view.size() == 3,
               "copying a span preserves the same borrowed range")) {
        return false;
    }

    if (!check(replace_at(writable, 1, 7), "middle replacement accepted") ||
        !check(owner == Triple{3, 7, 8}, "span changes its owner")) {
        return false;
    }
    std::uint64_t total = 0;
    if (!check(sum_values(read_only, total) && total == 18,
               "read-only span observes the updated sum")) {
        return false;
    }

    const auto suffix = read_only.subspan(1, 2);
    if (!check_sum(suffix, 15, "suffix sum") ||
        !check_sum(read_only.subspan(read_only.size(), 0), 0, "empty suffix")) {
        return false;
    }
    if (!check(!replace_at(writable, writable.size(), 99), "end index rejected") ||
        !check(owner == Triple{3, 7, 8}, "rejected replacement preserves owner")) {
        return false;
    }

    writable[0] = 4;
    if (!check(owner[0] == 4 && read_only[0] == 4,
               "const descriptor still permits mutable-element access")) {
        return false;
    }
    writable[0] = 3;

    const std::span<std::uint32_t> empty{};
    if (!check(!replace_at(empty, 0, 1), "empty span rejects replacement") ||
        !check_sum(empty, 0, "default empty span")) {
        return false;
    }
    const std::array<std::uint32_t, 2> wide{
        std::numeric_limits<std::uint32_t>::max(),
        std::numeric_limits<std::uint32_t>::max()
    };
    if (!check_sum(std::span<const std::uint32_t>{wide}, 8'589'934'590ULL,
                   "sum wider than one element")) {
        return false;
    }
    std::cout << "spans: size=" << read_only.size() << " sum=" << total << '\n';
    return true;
}
```

`.data()` exposes the address stored by a range, and `.size()` exposes its count.
The pointer comparison occurs while both views and the owner are valid; it
checks that a copied descriptor still describes the same storage. It is not a
technique for diagnosing a pointer after its storage has been released.

The two read-only descriptors were created before the middle replacement. They
observe it because their elements are the same elements owned by `owner`. The
`const` descriptor also allows the direct assignment `writable[0] = 4` because
its element type is mutable. The code restores that element to 3, keeping the
main fixture `{3, 7, 8}`. `total` is an independent integer result, not a live
sum that recomputes itself when an element changes.

An empty range is identified by a zero length, not by an assumed null address.
A default empty span and an empty suffix are both valid inputs. No element may
be accessed in either. The owner remains alive until this function returns, and
none of its borrowed descriptors escapes.

## A vector owns storage that can move

`std::vector<std::uint32_t>` owns a dynamically sized contiguous sequence.
The vector object and its element allocation are distinct: a local vector can
own a buffer elsewhere in memory. Copying this integer vector creates independent
elements; copying a span of it does not. The standard library manages the
allocation and releases it on normal destruction. Do not `delete` the address
returned by `.data()`. The [vector overview](https://eel.is/c++draft/vector.overview)
describes automatic storage management and contiguous storage for this element
type. We are not using the specialized `vector<bool>` representation.

Three quantities must remain distinct:

| Quantity | Meaning |
|---|---|
| `size()` | Number of current elements; valid subscript indices are below it |
| `capacity()` | Number of elements that fit before an append needs reallocation |
| `max_size()` | A container limit, not available physical memory or an allocation promise |

`reserve(8)` asks for capacity of at least eight; it does not create eight
elements. Writing `v[7]` into a vector of size three is invalid even when its
capacity is eight. `resize(8)` changes the element count to eight. With this
integer type and the default allocator, newly appended elements are initialized
to zero. Shrinking removes the trailing elements. The
[capacity operations](https://eel.is/c++draft/vector.capacity) specify these
different effects.

Brace and parenthesis initialization also express different requests here:
`std::vector<std::uint32_t> a{3}` contains one element equal to 3;
`std::vector<std::uint32_t> b(3)` contains three zero-valued elements. The
initializer-list form describes element values, while the count constructor
creates that many elements. Neither is universally the “safer spelling”; choose
the constructor that means what the program requires. The
[vector constructor specification](https://eel.is/c++draft/vector.cons) describes
the count forms.

### Size is not allocated payload

Let $n$ be the size, $c$ the capacity, and $e$ the bytes per element. The logical
element payload is $ne$ bytes; capacity accounts for $ce$ bytes of element
storage. If an observed vector has $n=3$, $c=8$, and $e=4$, those quantities are
12 and 32 bytes. They exclude the vector object, allocator overhead, and any
other process memory. This is a capacity illustration, not a claim that a
particular `reserve(8)` must report exactly eight.

In the engine, charging only size would let retained spare capacity escape the
memory budget. Here, it also explains why a successful reserve is not a hard
cap: appending past capacity can request another allocation. A bounded builder
must enforce its own admission limit.

A successful allocation is not guaranteed. Vector construction, copying, and
growth can throw if resources are unavailable. This foundation executable has
no recovery handler for allocation failure; such an uncaught exception is a
failed run, not a passing ownership test. Chapters 3 and 4 teach owned-resource
cleanup and explicit error boundaries. No manual resource-release code is
hidden in these container examples.

### A live vector object can still invalidate its views

A borrow can expire before its owner object is destroyed. Reallocation replaces
the element storage and invalidates references, pointers, and iterators into
the old allocation. Appending without reallocation preserves existing element
access, but does not enlarge a previously created span and does invalidate the
old vector end iterator. Erasing elements can invalidate positions even without
reallocation. See the [vector modifier rules](https://eel.is/c++draft/vector.modifiers).

Use the following distinctions when predicting a mutation:

| Operation | Consequence for earlier access paths |
|---|---|
| Assign an existing integer element | Existing views remain valid and observe the new value |
| Append with spare capacity | Existing element references remain valid; an earlier span keeps its old length |
| Grow beyond capacity or reserve above capacity | All earlier element access paths expire |
| Insert without reallocation | Positions before insertion survive; those at or after it do not |
| Erase or shrink | Removed elements cannot be accessed; earlier prefix elements can remain valid |
| Destroy the owner | No nonempty view into its elements remains usable |

A span spanning removed elements is not safe just because its first element
survives. An unchanged numeric address is not proof of a valid object or a
still-correct logical record. Our default discipline is simple: finish borrowing,
perform a structural mutation, and then construct fresh views from the live
owner. Use a narrower exception only when its actual invalidation rule is part
of the interface.

Append the first vector check. Its extra braces form a nested scope; locals in
that scope cease to exist at its closing brace. This deliberately ends the first
borrow before the later forced reallocation.

```cpp title="exercises/chapter-02/values.cpp"
bool check_vectors() {
    std::vector<std::uint32_t> owner{3, 5, 8};
    auto copied_owner = owner;
    copied_owner[1] = 99;
    if (!check(owner[1] == 5 && copied_owner[1] == 99,
               "copying an integer vector copies its elements")) {
        return false;
    }

    const std::vector<std::uint32_t> three_zeros(3);
    const std::vector<std::uint32_t> one_three{3};
    if (!check(three_zeros.size() == 3 && three_zeros[0] == 0 &&
                   three_zeros[1] == 0 && three_zeros[2] == 0 &&
                   one_three.size() == 1 && one_three[0] == 3,
               "count construction differs from list construction")) {
        return false;
    }

    owner.reserve(8);
    if (!check(owner.size() == 3 && owner.capacity() >= 8,
               "reserve changes available storage, not element count")) {
        return false;
    }
    {
        const std::span<const std::uint32_t> prefix{owner};
        const auto& first = owner[0];
        owner.push_back(13);
        if (!check(first == 3 && prefix.size() == 3 && owner.size() == 4,
                   "append within capacity preserves old elements, not old end") ||
            !check_sum(prefix, 16, "earlier span still describes three elements") ||
            !check_sum(std::span<const std::uint32_t>{owner}, 29, "new full span")) {
            return false;
        }
        std::cout << "vector: prefix size=" << prefix.size()
                  << " owner size=" << owner.size() << '\n';
    }
```

This block opens `check_vectors`; the next block completes the **same function**.
The only intentionally split function in the file is split at this scope
boundary, not at an unexplained operation.

The observed capacity is at least eight, so appending a fourth element cannot
require reallocation. `prefix` continues to describe `{3, 5, 8}`; it is not a
subscription to the vector's current size. The freshly constructed full span
also includes 13. Their sums are 16 and 29. We do not retain or use the vector's
old `end()` iterator.

To force reallocation, ask for **more than the actual capacity**, not a guessed
growth threshold. The capacity guard below also establishes that adding one
stays within `max_size()` and cannot wrap the size type. Reallocation can still
fail to allocate; the guard is not a memory-availability test.

Append the remainder of `check_vectors`:

```cpp title="exercises/chapter-02/values.cpp"
    const auto old_size = owner.size();
    const auto old_capacity = owner.capacity();
    if (!check(old_capacity < owner.max_size(), "room for a capacity-growth request")) {
        return false;
    }
    owner.reserve(old_capacity + 1);
    if (!check(owner.size() == old_size && owner.capacity() > old_capacity,
               "reserve above actual capacity reallocates without adding elements")) {
        return false;
    }
    std::uint64_t grown_sum = 0;
    {
        const std::span<const std::uint32_t> fresh{owner};
        if (!check(sum_values(fresh, grown_sum) && grown_sum == 29,
                   "fresh view reads the reallocated storage")) {
            return false;
        }
    }

    owner.resize(2);
    owner.resize(4);
    std::uint64_t resized_sum = 0;
    {
        const std::span<const std::uint32_t> fresh{owner};
        if (!check(fresh.size() == 4 && fresh[0] == 3 && fresh[1] == 5 &&
                       fresh[2] == 0 && fresh[3] == 0,
                   "resize removes trailing values, then creates zero elements") ||
            !check(sum_values(fresh, resized_sum) && resized_sum == 8,
                   "new elements are not the removed old values")) {
            return false;
        }
    }
    std::cout << "vector: reacquired sum=" << grown_sum
              << " resized sum=" << resized_sum << '\n';
    return true;
}
```

Every new view is obtained from `owner` **after** its relevant mutation. We never
compare, print, or dereference an old pointer after reallocation. There is no
assumption about whether the implementation grows by a factor of two or reports
an exact requested capacity.

The final pair of `resize` calls starts with `{3, 5, 8, 13}`, removes the last
two elements, then appends two initialized integers. The result is `{3, 5, 0, 0}`,
not the old four-element value. Reserved bytes do not preserve a removed element's
logical existence. This distinction will matter when a query batch reuses its
allocation for a different set of rows.

## A string view is a borrowed character range, not a C string

`std::string` owns a character sequence. `std::string_view` describes a read-only
character range owned elsewhere. Think of the latter as the text counterpart
of a read-only span: copying it does not copy the characters, and its `.size()`
is a length, not a search for a terminating zero. Its elements are `char` units;
a byte count is not generally a count of human-readable Unicode characters.
Our fixtures use ASCII and one explicit zero byte.

We will return the part of a line before the first `=`. The input is borrowed
for the call. The result continues borrowing from the **same character owner**.
No separator means the entire input; a leading separator or empty input yields
an empty name. No allocation is required for that extraction.

`find('=')` returns an index or the special value `std::string_view::npos` when
there is no match. Test that sentinel before treating it as an index. `substr(0,
separator)` on a **string view** returns another view, not an owned string. Its
position is valid here, and its length comes from the successful search. The
[searching](https://eel.is/c++draft/string.view.find) and
[substring](https://eel.is/c++draft/string.view.ops) contracts describe these
operations.

Append the extraction function and an owning-result alternative:

```cpp title="exercises/chapter-02/values.cpp"
std::string_view field_name(std::string_view line) {
    const auto separator = line.find('=');
    if (separator == std::string_view::npos) {
        return line;
    }
    return line.substr(0, separator);
}

std::string owned_field_name(std::string_view line) {
    return std::string{field_name(line)};
}
```

Returning a copied descriptor from a function is not itself a bug. The parameter
`line` dies on return, but the characters were not owned by that parameter in the
first place. The caller must keep their actual owner alive and avoid invalidating
its storage for as long as it uses the result.

The second function explicitly copies the selected characters into a new
`std::string`, using its [view-compatible constructor](https://eel.is/c++draft/string.cons). That result owns its contents and can outlive the input owner.
The copy costs work proportional to the selected length and can require an
allocation. A later subsystem retaining a name after its parsing buffer is
reused needs this kind of ownership, not a more elaborate dangling view.

### A six-character view need not end at a zero byte

For the owner `"amount=7"`, the field view has length six. The next character in
the owner's sequence is `=`, not `\0`. Printing the view with `std::cout << view`
uses the view's length, as specified by its
[stream insertion operation](https://eel.is/c++draft/string.view.io). Passing `view.data()` alone to an interface requiring a
null-terminated C string loses that length and can consume unrelated characters
or read beyond the owner. The [string-view data contract](https://eel.is/c++draft/string.view.access)
explicitly does not promise null termination.

To pass an ordinary text field to a C-string API, create an owning `std::string`
from the view and use that string's `.c_str()` **while it remains alive**. That
addresses termination, not arbitrary binary input: an embedded zero still
terminates a C string early. Prefer pointer-plus-length APIs when zero bytes
are meaningful data. Do not read `view[view.size()]` to check termination; it is
outside the view's element range.

Append the complete string checks:

```cpp title="exercises/chapter-02/values.cpp"
bool check_strings() {
    std::string line{"amount=7"};
    const auto borrowed = field_name(line);
    const std::string saved{borrowed};
    if (!check(borrowed == "amount" && borrowed.size() == 6 &&
                   line[borrowed.size()] == '=',
               "field is a counted prefix, not a null-terminated string")) {
        return false;
    }
    line[0] = 'A';
    if (!check(borrowed == "Amount" && saved == "amount",
               "borrowed text observes mutation, owned text is independent")) {
        return false;
    }
    std::cout << "strings: borrowed=" << borrowed << " owned=" << saved << '\n';

    if (!check(field_name("plain") == "plain" && field_name("=7").empty() &&
                   field_name(std::string_view{}).empty(),
               "text boundary cases")) {
        return false;
    }
    const std::array<char, 5> bytes{'a', '\0', 'b', '=', '9'};
    const std::string_view counted{bytes.data(), bytes.size()};
    const auto binary_name = field_name(counted);
    if (!check(binary_name.size() == 3 && binary_name[0] == 'a' &&
                   binary_name[1] == '\0' && binary_name[2] == 'b',
               "explicit text length includes embedded zero bytes")) {
        return false;
    }

    const auto independent = owned_field_name(std::string{"region=2"});
    return check(independent == "region", "owned result outlives temporary input");
}
```

`std::string` can supply a view of its characters to the parameter. The explicit
pointer/count constructor used for `counted` instead derives its length from the
five-element array. It does not scan for a zero byte; both constructor forms
are described in [string-view construction](https://eel.is/c++draft/string.view.cons).
All subscripts are guarded by a known size or a preceding short-circuit size
check. The lack of a terminator in `bytes` is harmless because none of our
operations requires one.

Assigning `line[0]` changes an existing character without a structural change.
Other operations such as appending or replacing the string can invalidate its
character access paths; consult the [string invalidation rules](https://eel.is/c++draft/string.require)
rather than assuming every non-growing string operation is safe for a saved
view. The program does not mutate the owner's structure while borrowing it.

The last fixture supplies a temporary string to `owned_field_name`. That input
survives through the call, and the selected characters are copied before its
destruction at the end of the statement. The returned string is independent.
The superficially similar borrowed form is **not** safe to retain:

```cpp title="Prediction only — do not add this dangling view to the normal program"
std::string_view bad = field_name(std::string{"amount=7"});
// The temporary owner has been destroyed at this semicolon.
// Later reading bad would access characters outside their owner's lifetime.
```

A `const` qualifier on `bad` would not help. Copying it into another view would
not help either. A named owner kept alive, or an owning result, changes the
lifetime relationship. The [temporary lifetime rules](https://eel.is/c++draft/class.temporary)
explain the end-of-statement destruction here. Binding a reference directly to
some temporaries has separate lifetime-extension rules; constructing a view is
not such a binding to its underlying owner.

Also distinguish `line.substr(...)` when `line` is a **string** from the same
method name on a **string view**. A [string substring](https://eel.is/c++draft/string.substr) is a new owning string.
Saving a view of that temporary substring can dangle. Establish the exact type
before reasoning from a method name or from `auto`.

## Connect the checks to one executable

The remaining block closes the namespace and supplies the program entry point.
All called functions have been defined above; there are no hidden helpers or
future engine dependencies.

```cpp title="exercises/chapter-02/values.cpp"
} // namespace strata

int main() {
    if (!strata::check_values() ||
        !strata::check_reference_assignment() ||
        !strata::check_range_loops() ||
        !strata::check_spans() ||
        !strata::check_vectors() ||
        !strata::check_strings()) {
        return 1;
    }
    std::cout << "all checks passed\n";
    return 0;
}
```

A failure stops subsequent checks and returns nonzero. No essential operation
is hidden in `assert`. The C++20 features already probed in Chapter 1 remain the
baseline; no compiler upgrade or newer language mode is required merely for
this chapter.

Append the following block **after the existing contents** of the root
`CMakeLists.txt`. Do not replace Chapter 1's targets or warning settings:

```cmake title="CMakeLists.txt"
add_executable(foundation-02 exercises/chapter-02/values.cpp)
target_compile_features(foundation-02 PRIVATE cxx_std_20)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(foundation-02 PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion
    )
endif()
add_test(NAME foundation-02 COMMAND foundation-02)
```

The [executable target](https://cmake.org/cmake/help/latest/command/add_executable.html)
contains only `values.cpp`. It does not link `foundation-01-stats`: the value/view
exercise supplies its own small functions. The old warning loop ran before this
target existed, so this new target receives its warning options explicitly.
CMake's root `enable_testing()` and extension policy are already supplied by
Chapter 1.

Reconfigure the same single-configuration build trees with the same compiler
and generator. These are native macOS commands, not Linux setup instructions:

```sh title="Terminal — cumulative Debug tests"
cmake -S . -B build/debug -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/debug --parallel 4 && \
  ctest --test-dir build/debug --output-on-failure --no-tests=error
./build/debug/foundation-02
```

```sh title="Terminal — cumulative Release tests"
cmake -S . -B build/release -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/release --parallel 4 && \
  ctest --test-dir build/release --output-on-failure --no-tests=error
./build/release/foundation-02
```

Stop at an unexpected configuration or build failure; do not treat a surviving
old executable as the new result. The complete test set now contains **three**
entries: `foundation-01`, `cxx20-probe`, and `foundation-02`. All should pass in
both configurations. The new executable's expected output is:

```text title="Expected foundation-02 output"
values: caller middle=5 returned middle=99
references: observed middle=7
range-for: copied sum=19 mutated sum=19
spans: size=3 sum=18
vector: prefix size=3 owner size=4
vector: reacquired sum=29 resized sum=8
strings: borrowed=Amount owned=amount
all checks passed
```

The reference-assignment, rejection, empty, wide-sum, and embedded-zero cases
pass silently. No address or implementation-specific capacity appears in the
expected output.

For one brief test-harness check, temporarily change **only** the span assertion
`sum_values(read_only, total) && total == 18` to expect `19`. Build Release and
run that test:

```sh title="Terminal — a wrong expected sum must fail"
cmake --build build/release --parallel 4 && \
  ctest --test-dir build/release -R '^foundation-02$' \
    --output-on-failure --no-tests=error
```

The test must fail with `FAIL read-only span observes the updated sum`. Restore
18, rebuild, and return the full test sets to passing. This failure exercise
changes no ownership logic and belongs within the chapter's test allocation.

## Confirm read-only access at compile time

Runtime tests cannot demonstrate an operation that the type system forbids.
Create this separate complete source file. It has no `main` because it will only
be compiled to an object, and its compilation is deliberately expected to fail:

```cpp title="exercises/chapter-02/readonly_rejected.cpp"
#include <cstdint>
#include <span>

void forbidden_write(std::span<const std::uint32_t> values) {
    values[0] = 7;
}
```

The error is assignment to a const element. No runtime call is made, so this
experiment does not manufacture an out-of-bounds access to an empty span. Compile
it independently after the normal program builds:

```sh title="Terminal — expect a const-element compilation error"
mkdir -p build/diagnostics
if "$CXX" -std=c++20 -arch arm64 -Wall -Wextra -Wpedantic \
    -c exercises/chapter-02/readonly_rejected.cpp \
    -o build/diagnostics/readonly-rejected.o; then
  printf 'Unexpected compilation success: inspect the element type.\n'
else
  printf 'Inspect the diagnostic: assignment to a const element is expected.\n'
fi
```

A missing header or invalid compiler option is not the expected result. Read
the diagnostic at the assignment. Changing the parameter to
`const std::span<std::uint32_t>` would permit the element assignment, because it
changes descriptor constness rather than element constness. Do not make that
change to the real read-only interface just to silence the error.

## Observe invalidation in a disposable sanitizer process

The normal vector check obeys the borrowing contract. A separate process can
show the consequence of violating it without contaminating that program. The
next source deliberately retains a view across an operation that **forces
reallocation if it succeeds**, then reads the stale view. Use it only as the following
sanitizer experiment, never as ordinary application logic or a passing test.

Create the entire diagnostic file:

```cpp title="exercises/chapter-02/dangling_span.cpp"
#include <cstdint>
#include <iostream>
#include <span>
#include <vector>

int main() {
    std::vector<std::uint32_t> owner{3, 5, 8};
    const std::span<const std::uint32_t> stale{owner};
    const auto old_capacity = owner.capacity();
    if (old_capacity >= owner.max_size()) {
        std::cerr << "Diagnostic could not request a larger capacity\n";
        return 2;
    }
    owner.reserve(old_capacity + 1);
    std::cout << stale[1] << '\n'; // Deliberate use after reallocation.
    return 0;
}
```

`reserve` may fail to allocate; that is a different outcome from reaching the
stale read. **If it returns successfully**, the larger request forces
reallocation. The vector object is still alive, and the saved view still has
length three, but its element address refers to the old released allocation.
Checking only `stale.size()` would not make `stale[1]` valid.

Compile and link with the same compatible native Clang driver and
AddressSanitizer enabled:

```sh title="Terminal — build the isolated invalid-access diagnostic"
mkdir -p build/diagnostics
"$CXX" -std=c++20 -arch arm64 -O1 -g \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -fsanitize=address -fno-omit-frame-pointer \
  exercises/chapter-02/dangling_span.cpp \
  -o build/diagnostics/dangling-span
```

Run only after that build succeeds. Preserve the failing process's status
without a pipeline replacing it:

```sh title="Terminal — inspect the expected sanitizer failure"
if ASAN_OPTIONS=halt_on_error=1 ./build/diagnostics/dangling-span \
    > build/diagnostics/dangling-span.log 2>&1; then
  printf 'No sanitizer failure observed; this does not make the stale view valid.\n'
else
  status=$?
  printf 'Diagnostic process exit status=%s\n' "$status"
fi
cat build/diagnostics/dangling-span.log
```

The useful expected observation is an AddressSanitizer **heap-use-after-free**
report at the stale read, with allocation and release stacks involving the
vector. The exact address, line layout, and status number are not portable
constants. A loader failure, unsupported sanitizer runtime, allocation failure,
or the explicit status 2 guard is not that observation. The
[Clang AddressSanitizer documentation](https://clang.llvm.org/docs/AddressSanitizer.html#usage)
describes instrumentation and linking; its
[report-symbolization section](https://clang.llvm.org/docs/AddressSanitizer.html#symbolizing-the-reports)
discusses readable source locations. On macOS, `xcrun dsymutil
build/diagnostics/dangling-span` can help produce source-location information
when the report lacks it; it is not a repair for the memory error.

There is no defined “old value” that this invalid program must print. A run
that seems to print 5 does not establish correctness. Sanitizers are diagnostics,
not the definition of validity; the invalidation rule already determines the
bug. Reacquiring a span **from `owner` after `reserve`**, as the normal program
does, is the repair. Increasing an earlier reserve only postpones the problem
unless the interface also enforces a no-growth borrowing interval.

Finally, build the **normal** program separately with memory diagnostics:

```sh title="Terminal — the safe program must still pass"
"$CXX" -std=c++20 -arch arm64 -O1 -g \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -fsanitize=address,undefined -fno-omit-frame-pointer \
  exercises/chapter-02/values.cpp \
  -o build/diagnostics/foundation-02-sanitized
./build/diagnostics/foundation-02-sanitized
```

The combined build also enables
[UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html).
Expected behavior is the normal eight-line output, success status, and no
sanitizer report. Do not use this diagnostic binary for performance measurements.
A toolchain lacking the requested runtime has an unperformed diagnostic check,
not a passed one; the ordinary Debug/Release checks remain separate.

## Diagnose the relationship, not just the line that crashed

| Symptom | Likely misunderstanding | Useful next inspection |
|---|---|---|
| A loop's changes disappear | Its element variable is a value | Compare `auto` with `auto&`, then inspect the owner after the loop |
| A supposed read-only parameter permits writes | `const` applies to a descriptor | Read the span's element type, not only its outer qualifiers |
| An older span misses a new vector element | A view's length was mistaken for a live subscription | Inspect its creation time and reacquire the intended range |
| A reserved vector rejects or crashes on indexing | Capacity was used as size | Establish `index < size()`; create elements before indexing them |
| A string field contains extra text in a C API | A counted range was treated as terminated | Preserve the length or create a suitable owning terminated string |
| A view fails after a function returns | The actual owner was local or temporary | Trace the character/element owner, not merely the returned descriptor |
| A view fails after growth | Storage was reallocated while borrowed | End the borrow before mutation and reconstruct from the live owner |
| Failure appears only under memory diagnostics | Invalid behavior happened to look plausible before | Follow the invalidation/lifetime rule; do not suppress the report |

Bounds, lifetime, and meaning are separate checks. A range can be in bounds
inside its old allocation yet dead. A live, in-bounds element can also hold a
different row after a buffer is reused. A future operator contract will need
to state both how long its batch is valid and what advancing that operator does
to previous views.

## What carries forward

Keep all three new sources and the appended CMake target. Only `values.cpp`
belongs to `foundation-02`; the other two remain named diagnostic examples.
Keep Chapter 1's sources and targets unchanged. The normal code now demonstrates
owning copies, mutable and const references, returned values, range loops,
array/vector storage, borrowed spans, text views, and safe reacquisition after
structural mutation.

For each future borrowed interface, state the owner, permitted mutation,
valid interval, and invalidating operations. A span makes range boundaries
visible; it does not enforce those lifetime promises. A value type's copy can
provide independent contents, while a copied descriptor still aliases its
original contents. That distinction is the bridge to the
[roadmap's](https://25d-228.github.io/cplusplus-learning/index.html) next chapter:
**Lifetime, RAII, and move-only owners**. It will implement an owner and trace
resource release rather than merely using ownership supplied by a container.

## Technical references

These references specify the mechanisms developed above. The public working
draft evolves beyond C++20; newer overloads, hardening language, and later library
facilities visible there are not additional course requirements. This chapter
uses C++20-compatible interfaces and gives all code needed for its exercises.

- **References and deduction:** [reference declarations](https://eel.is/c++draft/dcl.ref),
  [reference initialization and assignment](https://eel.is/c++draft/dcl.init.ref),
  [`auto` deduction](https://eel.is/c++draft/dcl.spec.auto),
  [range-based for](https://eel.is/c++draft/stmt.ranged), and
  [numeric limits](https://eel.is/c++draft/numeric.limits.members).
- **Owned and borrowed ranges:** [array](https://eel.is/c++draft/array.overview),
  [vector](https://eel.is/c++draft/vector.overview),
  [vector constructors](https://eel.is/c++draft/vector.cons),
  [vector capacity](https://eel.is/c++draft/vector.capacity),
  [vector modifiers](https://eel.is/c++draft/vector.modifiers),
  [span ownership](https://eel.is/c++draft/span.overview),
  [span element access](https://eel.is/c++draft/span.elem), and
  [span subviews](https://eel.is/c++draft/span.sub).
- **Text and lifetime:** [string-view construction](https://eel.is/c++draft/string.view.cons),
  [search](https://eel.is/c++draft/string.view.find),
  [substrings](https://eel.is/c++draft/string.view.ops),
  [element/data access](https://eel.is/c++draft/string.view.access),
  [string constructors](https://eel.is/c++draft/string.cons),
  [string substrings](https://eel.is/c++draft/string.substr),
  [view stream insertion](https://eel.is/c++draft/string.view.io),
  [string invalidation](https://eel.is/c++draft/string.require), and
  [temporary lifetimes](https://eel.is/c++draft/class.temporary).
- **Build and diagnosis:** [CMake executable targets](https://cmake.org/cmake/help/latest/command/add_executable.html),
  [Clang AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html), and
  [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html).
