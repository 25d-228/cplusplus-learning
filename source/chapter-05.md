# Chapter 5 — Templates, concepts, and ordering contracts

Two sorted inputs can be merged without sorting them again. The visible result
looks simple: `{1, 4}` and `{2, 4}` become `{1, 2, 4, 4}`. The interface is more
interesting. Who owns the inputs? What does “before” mean? Which copy of an
equivalent key comes first? What survives if a comparison throws after the
result already contains two elements?

We will implement the merge loop ourselves, parameterize its element and
comparison types, and return an owned vector. A C++20 concept will reject an
argument that cannot be called as a comparator. Separate runtime checks will
expose comparators that compile but do not describe a coherent ordering. The
same distinction will matter when a skip list, sorted table, or compaction loop
relies on an ordering it cannot prove from a function's type.

Start from the restored workspace of
[Chapter 4](https://25d-228.github.io/cplusplus-learning/chapter-04.html), including
all earlier CMake targets. Chapter 2 supplies values, references, vectors, and
spans; Chapter 3 supplies ownership and moves; Chapter 4 supplies exceptions,
optional values, and prepare-then-commit. No engine source exists yet. This
chapter does not include an earlier `.cpp` file or assume an unwritten utility
header.

> **Learning objectives**
>
> Write and instantiate a function template; explain callable objects and lambda
> captures; constrain expressions with a concept; distinguish syntactic checks
> from ordering laws; implement a stable merge over borrowed inputs; identify
> the copies, allocations, and exception boundary of an owned result.

The allocation is **2.0 implementation hours**: 0.5 for syntax experiments,
0.75 for the merge, 0.5 for ordering checks, and 0.25 for the expected-failure
compilation. Reading is separate. This exercise deliberately materializes its
result; it is not yet the bounded-buffer merge used for disk runs.

## Three files, one executable

Retain the compatible native C++20 compiler selected in Chapter 1. When Apple's
compiler is your chosen toolchain, prepare the directory with:

```sh title="Terminal — prepare the exercise"
CXX="$(xcrun --find clang++)"
mkdir -p exercises/chapter-05
```

Write these files in the existing learner workspace:

| File | Responsibility |
|---|---|
| `exercises/chapter-05/ordered_merge.hpp` | Complete constrained merge and its small helpers |
| `exercises/chapter-05/merge_demo.cpp` | Runnable fixtures, a bounded ordering sampler, and `main` |
| `exercises/chapter-05/non_callable.cpp` | Isolated compilation that must fail; not a normal build source |

Every block labelled with the same filename appends to that file **in reading
order**, even when a block for the other file appears between them. The root
`CMakeLists.txt` block appends one target. Blocks labelled as syntax explanations
or temporary edits are not additional source files. The header closes its
namespace and include guard after the merge definition; the executable closes
its namespace immediately before `main`.

## Decide what merging promises

Call the left input $L$ and the right input $R$. Their lengths are $n$ and $m$;
valid element indices are $0$ through $n-1$ and $0$ through $m-1$, respectively.
The predicate $a \prec b$ means “the comparator says that $a$ comes before $b$.”
It is not automatically the built-in integer `<` operator.

Both inputs must be sorted under that **same** predicate. The algorithm keeps
indices $i$ and $j$ at the first unconsumed elements. When both are available,
choose the right element only if $R_j \prec L_i$. Otherwise choose the left
one. This “otherwise” includes equivalent keys, so ties prefer the left input.
After one side is exhausted, copy the remaining suffix of the other side.

For the numeric fixture, the merge decisions are:

| Before the decision | Comparison | Element appended | Remaining inputs |
|---|---|---:|---|
| `i=0`, `j=0` | `2 < 1` is false | Left `1` | `{4}` and `{2,4}` |
| `i=1`, `j=0` | `2 < 4` is true | Right `2` | `{4}` and `{4}` |
| `i=1`, `j=1` | `4 < 4` is false | Left `4` | Empty and `{4}` |
| Left exhausted | No comparison | Right `4` | Both empty |

This is a merge, not a set union: both occurrences of 4 survive. Each input's
internal order also survives. To state stability across inputs, imagine their
concatenation as left followed by right; equivalent elements retain that order.
The [standard merge specification](https://eel.is/c++draft/alg.merge) gives a
useful reference for stable merging. Here we implement the index loop rather
than delegate it to `std::merge`.

The ownership contract is equally explicit. Inputs are borrowed read-only spans
whose elements remain alive and unchanged throughout the call. The output owns
newly constructed element values. The comparator is borrowed only until the
call returns and is never retained. Input spans may overlap because this
algorithm does not write through them. Another alias, callback, or thread must
not change their values or invalidate their storage during the call.

For the numeric and tagged records in this chapter, copying an element copies
all its data. Generic copying does not promise a deep clone of arbitrary pointer
members: a vector of pointers would own pointer values, not the objects they
point at. We are making the result's element ownership clear, not granting
ownership of everything an element can reach.

## A callable object is an object with a call operation

Begin the executable with its headers, check helper, and a concrete comparator:

```cpp title="exercises/chapter-05/merge_demo.cpp"
#include "ordered_merge.hpp"

#include <array>
#include <cstddef>
#include <cstdint>
#include <exception>
#include <iostream>
#include <limits>
#include <optional>
#include <span>
#include <stdexcept>
#include <vector>

namespace strata {

using Number = std::uint32_t;

bool check(bool condition, const char* name) {
    if (condition) {
        return true;
    }
    std::cerr << "FAIL " << name << '\n';
    return false;
}

struct NumericLess {
    bool operator()(const Number& a, const Number& b) const noexcept {
        return a < b;
    }
};
```

`Number` is an alias for `std::uint32_t`, not a new numeric type. `NumericLess`
is a class with a public `operator()`. For an object named `less`, the expression
`less(a, b)` invokes that member, just as `less.operator()(a, b)` would. The final
`const` means calling it does not require mutable access to the comparator
object. This comparator owns no state. `noexcept` is appropriate here because
comparing the two unsigned integers cannot throw.

A function object is a convenient way to package behavior with state when state
is needed. It does not require virtual functions, inheritance, dynamic
allocation, or a `std::function` wrapper. Our template will use its actual type.
Lambda expressions will give us shorter ways to construct such objects, but
first separate **type parameters** from ordinary function parameters.

## A template describes a family of functions

The following small definition illustrates the syntax; do not add it to a file:

```cpp title="Reading a function template — not additional source"
template<typename T, typename Compare>
bool comes_before(const T& a, const T& b, const Compare& before) {
    return before(a, b);
}
```

`T` and `Compare` name types. They are not integers or runtime variables.
`typename` in this template parameter list says “the argument here is a type”;
`class T` would have the same meaning and would not restrict `T` to class types.
The ordinary parameters are still `a`, `b`, and `before`, each with its usual
lifetime and reference rules. See the
[template parameter grammar](https://eel.is/c++draft/temp.param).

For a call with two `Number` values and a `NumericLess` object, the compiler can
deduce `T = Number` and `Compare = NumericLess`. Using the function requires a
specialization with those types substituted into its declaration and body.
Another element or comparator type can require another specialization. This
is compile-time selection, not a runtime test of which type an object contains.
It also does not promise one emitted machine-code copy per specialization:
optimization and linking can change emitted code without changing semantics.

The unconstrained example assumes that `before(a, b)` makes sense. Passing the
integer `7` as `before` does not. Instead of discovering that only while
instantiating the body, put the callable requirement at our real interface.

Start the header with its own dependencies; do not rely on the including `.cpp`
to supply them:

```cpp title="exercises/chapter-05/ordered_merge.hpp"
#ifndef STRATA_FOUNDATION_ORDERED_MERGE_HPP
#define STRATA_FOUNDATION_ORDERED_MERGE_HPP

#include <concepts>
#include <cstddef>
#include <span>
#include <stdexcept>
#include <type_traits>
#include <vector>

namespace strata {
```

Now append a concept, which names a compile-time condition on types:

```cpp title="exercises/chapter-05/ordered_merge.hpp"
template<typename Compare, typename T>
concept BooleanComparator = requires(const Compare& before,
                                     const T& a, const T& b) {
    { before(a, b) } -> std::same_as<bool>;
};
```

Read the definition from the inside out. The parameters after `requires` are
names used to describe expressions; they do not construct a comparator or execute
a comparison. They ask whether a call through a **const comparator reference**
with two **const element references** is valid. The braces around `before(a, b)`
introduce a *compound requirement*. The arrow constrains the expression's type;
it is not a lambda return-type declaration here.

[`std::same_as<bool>`](https://eel.is/c++draft/concept.same) requires the call expression to have type `bool` in this
case. An integer three-way comparison result is deliberately rejected, as is a
`bool&` result. We are not treating “negative means before” as truthiness: a
negative and a positive integer would both convert to `true`. The concept is
narrower than accepting every Boolean-convertible proxy. That narrowness makes
this interface unambiguous without teaching a proxy-reference framework.

The `requires` expression evaluates to a compile-time Boolean when its template
arguments are substituted. It does not run the function to see whether a
particular answer is correct. The
[compound-requirement rules](https://eel.is/c++draft/expr.prim.req.compound)
specify that the expression is unevaluated. We did **not** write a `noexcept`
requirement, so a throwing comparator is allowed.

The name `BooleanComparator` describes only this callable shape. It does not
claim that a comparator obeys ordering laws. `a <= b` has the required type but
is unsuitable for a strict ordering. We will demonstrate that mismatch safely
rather than pass it to a library sort whose requirements it violates.

## Validate counts without overflowing first

The result needs space for $n+m$ elements. Let $M$ be the result vector's
`max_size()`, measured in elements rather than bytes. Before computing the sum,
require $n \le M$ and $m \le M-n$:

$$
n + m \le M \quad \Longleftrightarrow \quad m \le M-n
\qquad \text{when } 0 \le n \le M.
$$

This is a check on representable size, not a claim that an allocation of that
size will succeed. For $M=10$, left length 7 leaves room for right length 3;
right length 4 must be rejected. Check left length 11 before trying to subtract
it from 10, because unsigned subtraction would wrap.

Append this complete helper to the header:

```cpp title="exercises/chapter-05/ordered_merge.hpp"
inline std::size_t checked_merged_size(std::size_t left,
                                     std::size_t right,
                                     std::size_t limit) {
    if (left > limit || right > limit - left) {
        throw std::length_error("merged result exceeds its element limit");
    }
    return left + right;
}
```

Built-in `||` short-circuits: when `left > limit`, the subtraction on the right
is not evaluated. Otherwise `limit - left` is valid, and accepting `right`
proves that the final addition is representable and within the supplied limit.
The helper throws `std::length_error`, which classifies an impossible requested
length separately from an allocation failure.

This is an ordinary, non-template function defined in a header. `inline` permits
matching definitions in multiple translation units under the language's
one-definition rules; the include guard only prevents duplicate inclusion
**inside one** translation unit. `inline` does not force call-site machine-code
substitution. See the [inline specification](https://eel.is/c++draft/dcl.inline).

The scalar helper also gives us an honest boundary test. We can test counts
near the maximum `size_t` without allocating a giant vector or fabricating a
span that claims to cover memory which does not exist.

## Check sortedness under the supplied comparator

With a valid strict weak ordering, a range is sorted when no adjacent successor
comes before its predecessor. We check that property on both inputs before
reserving the result's element storage:

```cpp title="exercises/chapter-05/ordered_merge.hpp"
template<typename T, typename Compare>
    requires BooleanComparator<Compare, T>
void require_sorted(std::span<const T> input, const Compare& before) {
    for (std::size_t i = 1; i < input.size(); ++i) {
        if (before(input[i], input[i - 1])) {
            throw std::invalid_argument("ordered_merge input is not sorted");
        }
    }
}
```

Here `requires BooleanComparator<Compare, T>` is a **requires clause** attached
to the function template. It uses the named condition defined earlier; it is
not another block of expression checks. A candidate whose constraint is false
is not viable for this call. The language distinguishes this clause from the
`requires (...) { ... }` expression used inside our concept; see
[constrained declarations](https://eel.is/c++draft/temp.constr.decl).

For `{1,4}`, the helper asks `before(4,1)`, which is false under numeric ascending
order. For `{4,1}`, it asks `before(1,4)`, which is true, so it rejects the input.
Equal adjacent keys are allowed when neither is before the other. Zero- and
one-element ranges have no adjacent pair and perform no comparisons.

This linear scan does **not** establish that the comparator is a strict weak
ordering. It cannot turn a cyclic predicate into an ordering, and its local
checks do not prove global sortedness without the ordering laws. Nor can it
check that a borrowed span denotes live memory. Those are separate obligations.

## Implement the stable merge

Append this definition and the header's closing lines:

```cpp title="exercises/chapter-05/ordered_merge.hpp"
template<typename T, typename Compare>
    requires std::copy_constructible<T>
          && std::same_as<T, std::remove_cvref_t<T>>
          && BooleanComparator<Compare, T>
std::vector<T> ordered_merge(std::span<const T> left,
                             std::span<const T> right,
                             const Compare& before) {
    std::vector<T> result;
    using Size = typename std::vector<T>::size_type;
    const Size total = checked_merged_size(left.size(), right.size(),
                                          result.max_size());
    require_sorted(left, before);
    require_sorted(right, before);
    result.reserve(total);

    std::size_t i = 0;
    std::size_t j = 0;
    while (i < left.size() && j < right.size()) {
        if (before(right[j], left[i])) {
            result.push_back(right[j]);
            ++j;
        } else {
            result.push_back(left[i]);
            ++i;
        }
    }
    while (i < left.size()) {
        result.push_back(left[i]);
        ++i;
    }
    while (j < right.size()) {
        result.push_back(right[j]);
        ++j;
    }
    return result;
}

}  // namespace strata
#endif
```

The first constraint is `std::copy_constructible<T>`. We read `const T` elements
and construct output values from them, so move-only ownership is not supported
by this interface. The library concept includes construction requirements and
a semantic expectation that copying preserves the source's value; it does not
promise that copying is cheap or nonthrowing. See
[`copy_constructible`](https://eel.is/c++draft/concept.copyconstructible).
A different merge that consumes move-only inputs would need writable inputs
and a different failure contract. It is not obtained by adding `std::move` to
this read-only algorithm.

`std::remove_cvref_t<T>` is a type transformation from `<type_traits>`: it strips
references and top-level `const`/`volatile` qualifiers. Requiring it to be the
same type as `T` keeps the owning vector's element type unqualified. Pass
`Number`, not `const Number`, as the template argument; read-only access already
comes from `span<const T>`. This check does not cast away qualifiers or change
an object. The [`remove_cvref` definition](https://eel.is/c++draft/meta.trans.other)
is a reference for the type transformation.

Inside the function, `using Size = typename std::vector<T>::size_type;` names the
vector's size type. This is another use of `typename`: it marks a dependent
qualified name as a type. The name depends on `T`, unlike the already-known
`std::size_t`. Use this explicit spelling rather than treating a nested type
as a runtime member. C++20 can infer type names in some type-only contexts;
that does not make `typename` a cast or a value declaration. See
[template name resolution](https://eel.is/c++draft/temp.res).

There are three distinct preparation steps: check the total, validate the
left/right sortedness, then reserve capacity. `reserve(total)` changes capacity,
not size; no output elements exist yet. It reserves **at least** the requested
capacity, not necessarily exactly that much. During the subsequent appends,
size never exceeds the reserved total, so those appends do not reallocate the
vector's element storage. Element copying can still allocate internally or
throw for other element types. The relevant rules are in
[vector capacity](https://eel.is/c++draft/vector.capacity).

The main loop uses `before(right[j], left[i])`. Only a strictly earlier right
element wins. If the answer is false, left wins even when the elements are
equivalent. The two suffix loops need no comparisons because each input is
already sorted. Every index increment follows the corresponding successful
`push_back`; no iteration consumes an element without first constructing its
output value.

### The loop invariant explains both the result and its bound

At the top of each merge iteration:

$$
0 \le i \le n,\qquad 0 \le j \le m,\qquad
\operatorname{size}(\text{result}) = i+j.
$$

The result is the stable merge of exactly the consumed prefixes $L[0,i)$ and
$R[0,j)$. The half-open notation means that the end index is excluded. Initially
both prefixes and the result are empty. Choosing the earlier remaining head
appends the next valid element; choosing left for equivalence preserves the
left-before-right tie rule. Incrementing its index extends exactly that
consumed prefix by one. Thus the invariant is preserved. When both suffixes
are exhausted, $i=n$ and $j=m$, and the result has exactly $n+m$ elements.

For nonempty inputs, the main loop makes at most $n+m-1$ comparisons: each
comparison consumes one element, and the loop must stop with at least one
element left for a suffix when it has consumed $n+m-1$ elements. With either
input empty it makes zero comparisons. Our validation adds
$\max(n-1,0)+\max(m-1,0)$ comparisons on successful inputs. If $C$ is the number
of main-loop comparisons, the total callback count is:

$$
C_{\text{total}} = \max(n-1,0)+\max(m-1,0)+C.
$$

For two inputs of length two, validation costs one call per input; the trace
above costs three merge calls. The observed total must therefore be five, not
three. Each of the four values is copied into the result exactly once. The
algorithm has linear element work, but the time taken by a comparison or an
element copy still matters; generic syntax does not make these operations
constant-cost for every possible type.

### Returning ownership does not require forwarding machinery

`left[i]` and `right[j]` are const lvalue expressions. Of the vector's
`push_back(const T&)` and `push_back(T&&)` overloads, the copy-taking operation
is the applicable one here. Casting one of those expressions with `std::move`
would preserve its constness and would not create permission to consume a
move-only owner. These overloads are specified in
[vector modifiers](https://eel.is/c++draft/vector.modifiers).

The local `result` is different: it is a non-const owner that the function is
finished using. `return result;` allows construction directly in the caller's
result object or the applicable move when elision is not performed. It does
not return a reference to the local vector or copy a span pointing into it.
Do not write `return std::move(result);` as a ritual; the ordinary spelling
already enables the intended return behavior. See
[return statements](https://eel.is/c++draft/stmt.return).

Our comparator parameter is `const Compare&`, not a forwarding reference. The
algorithm synchronously borrows it and invokes it as a const lvalue. A temporary
such as `NumericLess{}` lives through that call. No comparator is stored for
later execution. We do not need `Compare&&`, `std::forward`, type erasure, or a
custom callable hierarchy merely to select one comparison operation.

## Definition visibility and deduction are different questions

The header contains the complete template definitions so an including
translation unit can instantiate them with its actual element and comparator
types. Moving those definitions to a separately compiled `.cpp` and exposing
only declarations would not, by itself, arrange instantiation for arbitrary
caller-defined comparator types. Explicit instantiation can support a known
finite set, but it is not our extensible interface. This is different from
Chapter 1's non-template statistics function, whose one compiled definition
could be linked from a library. See
[implicit instantiation](https://eel.is/c++draft/temp.inst).

Deduction happens earlier, when a call is matched to the function template.
The merge's input parameters are `std::span<const T>`, not `std::array<T,N>`.
In the C++20 interface used here, deduction does not first convert an arbitrary
array or vector into a span in order to discover `T`. Use either of two clear
call forms:

| Call form | What supplies the types |
|---|---|
| `ordered_merge<Number>(left_array, right_array, less)` | `T` is explicit; array-to-span conversions then apply; `Compare` is deduced |
| `ordered_merge(left_view, right_view, less)` | Both views already have type `std::span<const Number>`; types can be deduced |

A span with mutable elements or a static extent can likewise be a different
pattern from our const-element, dynamic-extent parameter. Avoid relying on a
conversion to solve template deduction: use the explicit element type or
construct the exact views. See
[function-call template deduction](https://eel.is/c++draft/temp.deduct.call).

This is also a useful distinction in diagnosing overload selection. A call
can fail because deduction could not match the parameter shape, because a
constraint made a candidate nonviable, or because code inside a chosen
specialization is invalid. These are not interchangeable “template errors.”
Adding casts indiscriminately can hide the first symptom without fixing the
interface.

## Count calls with a lambda that borrows a counter

A lambda expression creates a callable object with an unnamed closure type.
`auto` can name a variable of that type without knowing its spelling. Empty
brackets `[]` mean no local captures; `[&calls]` means the closure borrows the
local `calls` variable. The parameter list and body have their usual meanings.
Without `mutable`, the ordinary lambda's call operator is const; mutating a
**referenced external counter** does not mutate the closure's stored state.
See [closure types](https://eel.is/c++draft/expr.prim.lambda.closure).

Append the integer fixture to `merge_demo.cpp`:

```cpp title="exercises/chapter-05/merge_demo.cpp"
bool check_integer_merge() {
    const std::array<Number, 2> left{1, 4};
    const std::array<Number, 2> right{2, 4};
    const std::vector<Number> expected{1, 2, 4, 4};
    auto result = ordered_merge<Number>(left, right, NumericLess{});
    if (!check(result == expected, "integer merge")) {
        return false;
    }

    std::size_t calls = 0;
    const auto counted_less = [&calls](const Number& a, const Number& b) {
        ++calls;
        return a < b;
    };
    const std::span<const Number> left_view{left};
    const std::span<const Number> right_view{right};
    const auto counted = ordered_merge(left_view, right_view, counted_less);
    if (!check(counted == expected && calls == 5,
               "deduction and comparison count")) {
        return false;
    }

    result[0] = 99;
    if (!check(left[0] == 1 && right[0] == 2 && counted[0] == 1,
               "result owns independent integer elements")) {
        return false;
    }
    std::cout << "merge: 1 2 4 4; comparisons=" << calls << '\n';
    return true;
}
```

The first call uses a named function-object type and an explicit element type.
The second uses exact dynamic-extent spans, allowing deduction, and a lambda.
Both invoke the same source algorithm. They can have different template
specializations because their comparator types differ.

The lambda increments the referenced counter and always returns the ordinary
`a < b` answer. The counter remains alive for the complete merge call. Its value
is diagnostic state, not part of the ordering. This custom interface permits
such diagnostics and will not roll them back on failure. It still requires
consistent answers for the same unchanged values whenever comparisons return
normally. `const` access to the closure is not a promise that it cannot affect
anything reachable through references.

The final mutation sets the first element of one result to 99. Neither input
nor the independently constructed second result changes. We have tested the
ownership boundary, not just the sorted integer sequence. The `check` helper
uses ordinary control flow, so the checks remain active with `NDEBUG` in a
Release build.

For this fixture there are four logical output elements. With the 32-bit
`Number` representation checked in the earlier foundations, their nominal
payload is $4\times4=16$ bytes. Allocated element capacity may be larger than
four; allocator bookkeeping, stack objects, and input storage are separate.
For these ordinary element types, capacity $K$ corresponds to $K\,\mathrm{sizeof}(T)$
bytes of element storage, not total process memory. Do not apply that formula
to the [specialized `vector<bool>` representation](https://eel.is/c++draft/vector.bool.pspc)
or to the extra allocations an owning element may make.

This in-memory result grows with the total input size. It does not enforce the
engine's later memory budget or replace a streaming spill merge. Reusing its
tie rule does not require reusing its materialization strategy.

## Exercise empty, overlapping, and oversized inputs

Append the boundary checks:

```cpp title="exercises/chapter-05/merge_demo.cpp"
bool check_boundaries() {
    const std::span<const Number> empty{};
    const std::array<Number, 2> values{1, 4};
    const std::vector<Number> expected{1, 4};
    if (!check(ordered_merge<Number>(empty, empty, NumericLess{}).empty()
               && ordered_merge<Number>(values, empty, NumericLess{}) == expected
               && ordered_merge<Number>(empty, values, NumericLess{}) == expected,
               "empty inputs")) {
        return false;
    }

    const std::array<Number, 3> shared{1, 2, 3};
    const std::span<const Number> view{shared};
    const auto overlap = ordered_merge(view.first(2), view.subspan(1), NumericLess{});
    if (!check(overlap == std::vector<Number>{1, 2, 2, 3},
               "overlapping read-only inputs")) {
        return false;
    }

    const auto maximum = std::numeric_limits<std::size_t>::max();
    if (!check(checked_merged_size(maximum - 2, 2, maximum) == maximum,
               "count exactly at limit")) {
        return false;
    }
    bool sum_rejected = false;
    bool left_rejected = false;
    try {
        (void)checked_merged_size(maximum, 1, maximum);
    } catch (const std::length_error&) {
        sum_rejected = true;
    }
    try {
        (void)checked_merged_size(11, 0, 10);
    } catch (const std::length_error&) {
        left_rejected = true;
    }
    if (!check(sum_rejected && left_rejected, "oversized count rejection")) {
        return false;
    }
    std::cout << "boundaries: empty, overlap, and checked sizes passed\n";
    return true;
}
```

An empty span has zero elements; neither validation nor a suffix loop indexes
it. When just one input is empty, the other is copied in order. A large reserve
or a special empty sentinel is unnecessary.

The overlap case uses two valid views into `{1,2,3}`: the first two elements and
the suffix beginning at index one. The value 2 is intentionally in both inputs,
so `{1,2,2,3}` is correct. The input owner is unchanged throughout the call. This
would need a different contract for a merge that wrote back into either input.

The remaining checks use only scalar counts. `maximum - 2` plus 2 is accepted;
`maximum` plus 1 is rejected before addition; left count 11 against limit 10 is
rejected before subtraction. We do **not** create a `maximum`-length span or try
to allocate that many elements. These checks establish arithmetic behavior,
not available-memory behavior. Chapter 4 already supplied a deterministic
allocation-failure seam without requiring machine exhaustion.

The `(void)` casts explicitly discard a return value in paths where the expected
observation is an exception. They do not suppress exceptions or change control
flow. Each catch records that the expected exception occurred; an unexpected
exception will reach `main` and fail the program.

## Equivalent keys are not identical records

The duplicate integer 4 cannot reveal which input supplied it. Attach a small
payload tag and compare only the key:

```cpp title="exercises/chapter-05/merge_demo.cpp"
struct Tagged {
    Number key;
    char tag;
};

bool check_stability() {
    const std::array<Tagged, 3> left{{{1, 'a'}, {4, 'b'}, {4, 'c'}}};
    const std::array<Tagged, 3> right{{{2, 'x'}, {4, 'y'}, {4, 'z'}}};
    const auto by_key = [](const Tagged& a, const Tagged& b) {
        return a.key < b.key;
    };
    const auto result = ordered_merge<Tagged>(left, right, by_key);
    const std::array<Number, 6> keys{1, 2, 4, 4, 4, 4};
    const std::array<char, 6> tags{'a', 'x', 'b', 'c', 'y', 'z'};
    if (!check(result.size() == tags.size(), "tagged result size")) {
        return false;
    }
    for (std::size_t i = 0; i < result.size(); ++i) {
        if (!check(result[i].key == keys[i] && result[i].tag == tags[i],
                   "equivalent-key order")) {
            return false;
        }
    }
    if (!check(!by_key(left[1], right[1]) && !by_key(right[1], left[1])
               && left[1].tag != right[1].tag, "equivalence is not identity")) {
        return false;
    }
    std::cout << "stable tags: a x b c y z\n";
    return true;
}
```

The nested braces initialize an array of small aggregates. Each `Tagged` contains
a key and a character; no packed binary layout or disk encoding is implied.
The lambda has no captures and reads only `key`. Thus `{4,'b'}` and `{4,'y'}`
are **equivalent under this comparator** even though their tags differ.

For any comparator `before`, define equivalence by:

$$
a \sim b \quad\Longleftrightarrow\quad
\neg(a\prec b)\;\land\;\neg(b\prec a).
$$

The symbol $\neg$ means “not,” and $\land$ means “and.” Equality of all fields
is not required. Among the key-4 records, the merge must emit left tags `b,c`
before right tags `y,z`, preserving each input's order. The complete tag order
is therefore `a,x,b,c,y,z`.

A loop that instead chooses the left element only when `before(left,right)`
is true will choose the right element on ties. Its numeric result can look
perfect, but these tags reveal the lost left-before-right stability. This is
why verifying only sorted keys is not enough to verify a merge contract.

In the later engine, key ordering and version precedence will be explicit parts
of a record comparator. “Left wins ties” is not a substitute for defining which
sequence number is newer. This exercise establishes stable mechanics without
silently specifying the engine's version-resolution policy.

## Captured state becomes part of the ordering contract

A captured value and a captured reference behave differently. Append this
complete check:

```cpp title="exercises/chapter-05/merge_demo.cpp"
bool check_captured_direction() {
    bool descending = true;
    const auto copied_direction = [descending](const Number& a, const Number& b) {
        return descending ? a > b : a < b;
    };
    const auto borrowed_direction = [&descending](const Number& a, const Number& b) {
        return descending ? a > b : a < b;
    };
    descending = false;
    if (!check(copied_direction(4, 1) && borrowed_direction(1, 4),
               "capture by value versus reference")) {
        return false;
    }
    const std::array<Number, 2> left{4, 1};
    const std::array<Number, 2> right{4, 2};
    const auto result = ordered_merge<Number>(left, right, copied_direction);
    if (!check(result == std::vector<Number>{4, 4, 2, 1}, "descending merge")) {
        return false;
    }
    std::cout << "captured order: 4 4 2 1\n";
    return true;
}
```

Both closures are created when `descending` is true. The value capture stores
that value in its closure. Changing the original variable to false does not
change the stored copy, so `copied_direction(4,1)` remains true. The reference
capture still refers to the original variable, so `borrowed_direction(1,4)`
now returns true instead. See the
[lambda capture rules](https://eel.is/c++draft/expr.prim.lambda.capture).

The ternary expression selects between `a > b` and `a < b`; its result is
Boolean in either branch. The two descending inputs `{4,1}` and `{4,2}` match
the copied comparator's direction, so their result is `{4,4,2,1}`. Changing
comparator state after sorting inputs can make those inputs unsorted under the
new state. Changing it during merging is worse: there is no single relation
under which the loop can be reasoned about.

The borrowed-direction closure is used only while its referenced variable is
alive. Returning it from this function would not extend that variable's
lifetime. A captured pointer or span has the same limitation: copying a view
into a closure does not copy or retain its underlying storage. Later tasks that
outlive their submitting scope will need an explicit ownership policy.

Our concept requires const-callability. A lambda marked `mutable` ordinarily
has a non-const call operator and would not satisfy that interface. There is no
need to make this lambda mutable merely to increment the earlier referenced
counter. Conversely, accepting a const-callable object is not a complete purity
or lifetime check.

## The laws the compiler cannot establish for you

A strict weak ordering is a relation that orders equivalence classes while
allowing different objects to share a class. The required implications are:

| Law | Meaning |
|---|---|
| Irreflexivity | No value comes before itself: $\neg(a\prec a)$ |
| Asymmetry | If $a\prec b$, then $\neg(b\prec a)$ |
| Transitivity of order | If $a\prec b$ and $b\prec c$, then $a\prec c$ |
| Transitivity of equivalence | If $a\sim b$ and $b\sim c$, then $a\sim c$ |

Asymmetry follows from irreflexivity and transitivity, but checking it separately
produces a simpler two-value diagnostic. For numeric `<`, the equivalence
classes are individual numeric values. For our tagged comparator, all records
with the same key form one class regardless of tag.

The standard's [`strict_weak_order` concept](https://eel.is/c++draft/concept.strictweakorder)
has semantic requirements beyond call-expression validity. The ability to
instantiate a concept expression does not let the compiler prove those laws
for arbitrary function bodies. The standard also distinguishes stable,
equality-preserving behavior from merely callable expressions; see
[equality preservation](https://eel.is/c++draft/concepts.equality).
We use our explicitly syntactic `BooleanComparator` for the diagnostic tool,
not a standard constrained algorithm whose semantic requirements are knowingly
violated by the test predicates.

Consider three failures on values `0,1,2`. The predicate `a <= b` says `0 <= 0`,
so irreflexivity fails immediately. A cycle saying `0` before `1`, `1` before `2`,
and `2` before `0` has no self-edge or two-way edge, but `0` before `1` and `1`
before `2` do not imply `0` before `2`. Finally, “before only when the gap exceeds
one” relates `0` to `2`, but leaves `0` equivalent to `1` and `1` equivalent to
`2`. That fails transitivity of equivalence. Testing only self-comparisons would
miss both of the latter defects.

### A bounded sampler returns a counterexample, not a proof

Add the diagnostic result types to the executable:

```cpp title="exercises/chapter-05/merge_demo.cpp"
enum class OrderLaw {
    irreflexivity,
    asymmetry,
    transitivity,
    equivalence_transitivity
};

struct OrderViolation {
    OrderLaw law;
    std::size_t i;
    std::size_t j;
    std::size_t k;
};
```

The three indices identify positions in the sample, not copied payload values.
For a one-value failure they are the same; for a two-value failure the unused
third position repeats the second. `std::optional<OrderViolation>` will
represent either an observed counterexample or absence of one.

The sampler below accepts at most eight values. For a sample of length $s$, it
constructs a Boolean matrix $B$ with logical shape $[s,s]$:

$$
B_{ij} = \operatorname{before}(\text{values}[i],\text{values}[j]).
$$

It stores that active matrix in a fixed $8\times8$ nested array; unused cells
are irrelevant. Building the matrix takes $s^2$ callback invocations. Checking
triples takes at most $s^3$ iterations. This small cubic diagnostic belongs in
the fixture, not in the merge's normal input path.

Append the complete sampler:

```cpp title="exercises/chapter-05/merge_demo.cpp"
template<typename T, typename Compare>
    requires BooleanComparator<Compare, T>
std::optional<OrderViolation> sample_order(std::span<const T> values,
                                           const Compare& before) {
    if (values.size() > 8) {
        throw std::invalid_argument("order sample contains more than eight values");
    }
    std::array<std::array<bool, 8>, 8> less{};
    const auto n = values.size();
    for (std::size_t i = 0; i < n; ++i) {
        for (std::size_t j = 0; j < n; ++j) {
            less[i][j] = before(values[i], values[j]);
        }
    }
    for (std::size_t i = 0; i < n; ++i) {
        if (less[i][i]) {
            return OrderViolation{OrderLaw::irreflexivity, i, i, i};
        }
    }
    for (std::size_t i = 0; i < n; ++i) {
        for (std::size_t j = 0; j < n; ++j) {
            if (less[i][j] && less[j][i]) {
                return OrderViolation{OrderLaw::asymmetry, i, j, j};
            }
        }
    }
    const auto equivalent = [&less](std::size_t i, std::size_t j) {
        return !less[i][j] && !less[j][i];
    };
    for (std::size_t i = 0; i < n; ++i) {
        for (std::size_t j = 0; j < n; ++j) {
            for (std::size_t k = 0; k < n; ++k) {
                if (less[i][j] && less[j][k] && !less[i][k]) {
                    return OrderViolation{OrderLaw::transitivity, i, j, k};
                }
                if (equivalent(i, j) && equivalent(j, k) && !equivalent(i, k)) {
                    return OrderViolation{OrderLaw::equivalence_transitivity, i, j, k};
                }
            }
        }
    }
    return std::nullopt;
}
```

The size guard executes before any matrix indexing. `less[i][j]` first selects
row `i` from the outer `std::array`, then element `j` from that row. The aggregate
initialization `{}` starts every Boolean cell as false; the fill loop overwrites
every cell that will be read. This is a fixed stack object, not `vector<bool>`.

The first scan checks the diagonal. The second looks for paired true entries
across the diagonal. The triple loop checks the two transitivity implications:
`B[i][j] && B[j][k] && !B[i][k]` is precisely an order counterexample. The local
`equivalent` lambda derives equivalence from the already-filled matrix, so
checking it makes no additional callback calls. Its borrowed `less` array
outlives the lambda's use.

The sampler invokes no sorting, merging, binary search, or associative-container
operation. Its own contract permits arbitrary defined Boolean answers; such
answers are the data being inspected. A predicate may violate ordering laws
without violating this sampler's preconditions. The predicate must still be
safe to call on every pair in the sample, must not invalidate the sample, and
may propagate an exception. No test harness can make an out-of-bounds predicate
safe merely by calling it a comparator.

Passing the sampler means only that **this table of observed answers** contains
no counterexample. It does not cover values outside the sample, prove stable
answers on repeated calls, or verify a borrowed object's lifetime. An empty
sample has no counterexample regardless of the predicate. Even on a nonempty
sample, a time-varying predicate might happen to produce a consistent table
once. Use sampling to find bugs, and use the relation's definition and domain
to justify correctness.

### Make each bad relation fail for a concrete reason

Append the law checks:

```cpp title="exercises/chapter-05/merge_demo.cpp"
bool check_order_laws() {
    const std::array<Number, 3> sample{0, 1, 2};
    const auto non_strict = [](Number a, Number b) { return a <= b; };
    const auto both_directions = [](Number a, Number b) { return a != b; };
    const auto cycle = [](Number a, Number b) {
        return (a == 0 && b == 1) || (a == 1 && b == 2) || (a == 2 && b == 0);
    };
    const auto gap = [](Number a, Number b) { return a < b && b - a > 1; };
    static_assert(BooleanComparator<NumericLess, Number>);
    static_assert(BooleanComparator<decltype(non_strict), Number>);
    static_assert(!BooleanComparator<int, Number>);

    const auto good = sample_order<Number>(sample, NumericLess{});
    const auto reflexive = sample_order<Number>(sample, non_strict);
    const auto symmetric = sample_order<Number>(sample, both_directions);
    const auto cyclic = sample_order<Number>(sample, cycle);
    const auto gap_result = sample_order<Number>(sample, gap);
    if (!check(!good, "ordinary less has no sampled violation")
        || !check(reflexive && reflexive->law == OrderLaw::irreflexivity
                  && reflexive->i == 0, "non-strict witness")
        || !check(symmetric && symmetric->law == OrderLaw::asymmetry,
                  "asymmetry witness")
        || !check(cyclic && cyclic->law == OrderLaw::transitivity
                  && cyclic->i == 0 && cyclic->j == 1 && cyclic->k == 2,
                  "cycle witness")
        || !check(gap_result && gap_result->law == OrderLaw::equivalence_transitivity
                  && gap_result->i == 0 && gap_result->j == 1 && gap_result->k == 2,
                  "equivalence witness")) {
        return false;
    }
    const std::span<const Number> empty{};
    if (!check(!sample_order(empty, NumericLess{}), "empty sample has no witness")) {
        return false;
    }
    bool large_rejected = false;
    try {
        const std::array<Number, 9> large{};
        (void)sample_order<Number>(large, NumericLess{});
    } catch (const std::invalid_argument&) {
        large_rejected = true;
    }
    if (!check(large_rejected, "sample-size guard")) {
        return false;
    }
    std::cout << "order samples: < has no witness; <=, !=, cycle, gap rejected\n";
    return true;
}
```

`decltype(non_strict)` names the type of that lambda variable without executing
it. The compile-time assertion deliberately confirms that `<=` satisfies the
**syntactic** concept. The assertion for `int` confirms that an integer is not
a callable comparator. A `static_assert` checks a constant expression during
compilation, unlike the runtime `check` calls that evaluate particular values.

For the cycle, the expected witness indices are `(0,1,2)`: `0` precedes `1`,
`1` precedes `2`, but `0` does not precede `2`. For the gap relation, the same
indices witness `0 ~ 1` and `1 ~ 2` but not `0 ~ 2`. The expression
`a < b && b - a > 1` computes the subtraction only when `a < b`; it avoids
unsigned wraparound in the gap calculation. The `!=` predicate supplies a
separate asymmetry failure on any two distinct sample values.

The optional is checked before its fields are accessed with `->`. A returned
witness is an ordinary owned value; it does not retain the sample or the
comparator. The sample cap is tested with nine actual elements, not a made-up
length. None of these four invalid predicates is passed to `ordered_merge` or
to a standard ordering algorithm.

## A valid comparator can still receive unsorted input

Comparator laws and input sortedness are separate checks. Use the valid
`NumericLess` with an invalid input on each side in turn:

```cpp title="exercises/chapter-05/merge_demo.cpp"
bool check_unsorted_input() {
    const std::array<Number, 2> sorted{1, 4};
    const std::array<Number, 2> unsorted{4, 1};
    bool left_rejected = false;
    bool right_rejected = false;
    try {
        (void)ordered_merge<Number>(unsorted, sorted, NumericLess{});
    } catch (const std::invalid_argument&) {
        left_rejected = true;
    }
    try {
        (void)ordered_merge<Number>(sorted, unsorted, NumericLess{});
    } catch (const std::invalid_argument&) {
        right_rejected = true;
    }
    if (!check(left_rejected && right_rejected, "unsorted input rejection")) {
        return false;
    }
    std::cout << "unsorted input: left and right rejected\n";
    return true;
}
```

Both calls must throw `std::invalid_argument` rather than return a misleading
result. The left and right cases ensure that neither validation call was
accidentally omitted. The checks occur before `reserve`, although exception
message construction itself is not a promise of allocation-free behavior.
The input arrays remain unchanged because the validation only reads them.

Calling `require_sorted` with an invalid comparator would not be a repair for
that comparator. The order sampler diagnoses a different class of problem;
neither layer should be mistaken for the other.

## Throw after two output elements, before installing the result

An exception can come from validation, vector allocation, an element copy, or
the comparator. Our algorithm never publishes its partial vector. Its local
owner destroys any constructed elements and releases its storage during
unwinding. Under the stated read-only input and value-copy contracts, input
values remain unchanged. This says nothing about rolling back comparator
counters, logging, or arbitrary external side effects.

We can test a meaningful failure point without a new allocation framework.
For the standard integer fixture, calls 1 and 2 validate the two inputs. Call 3
selects the first left value; call 4 selects the first right value. Throw on
call 5, after the partial result holds `{1,2}`:

| Point | Private merge result | Installed result | Diagnostic counter |
|---|---|---|---:|
| Before invocation | Not constructed | `{90,91}` | 0 |
| After input validation | Empty | `{90,91}` | 2 |
| After two merge choices | `{1,2}` | `{90,91}` | 4 |
| Fifth call throws | Destroyed during unwinding | `{90,91}` | 5 |
| Successful retry and swap | Old installed value | `{1,2,4,4}` | Still 5 |

Append the check:

```cpp title="exercises/chapter-05/merge_demo.cpp"
bool check_comparator_failure() {
    const std::array<Number, 2> left{1, 4};
    const std::array<Number, 2> right{2, 4};
    std::vector<Number> installed{90, 91};
    const std::span<const Number> old_view{installed};
    std::size_t calls = 0;
    const auto fail_fifth = [&calls](const Number& a, const Number& b) {
        ++calls;
        if (calls == 5) {
            throw std::runtime_error("injected comparator failure");
        }
        return a < b;
    };
    bool rejected = false;
    try {
        auto candidate = ordered_merge<Number>(left, right, fail_fifth);
        installed.swap(candidate);
    } catch (const std::runtime_error&) {
        rejected = true;
    }
    if (!check(rejected && calls == 5 && installed.data() == old_view.data()
               && installed == std::vector<Number>{90, 91}
               && old_view[0] == 90 && old_view[1] == 91,
               "throw before result commit")) {
        return false;
    }
    std::cout << "throw on call " << calls << ": installed 90 91 preserved\n";

    auto candidate = ordered_merge<Number>(left, right, NumericLess{});
    installed.swap(candidate);
    // old_view now refers to candidate's old payload, not installed's new one.
    // Do not use it after that payload is released at this function's end.
    if (!check(installed == std::vector<Number>{1, 2, 4, 4}, "successful retry")) {
        return false;
    }
    std::cout << "retry: 1 2 4 4\n";
    return true;
}
```

The local candidate is initialized only after `ordered_merge` completes. On the
injected failure, that initialization never completes, `installed.swap` is not
reached, and the old vector retains both its contents and its element address.
The old span remains valid. The callback's reference to `calls` remains valid
through the exception and the catch. The counter intentionally remains five;
it is outside the protected result state.

The retry uses a nonthrowing numeric comparator and commits by swapping two
vectors with their ordinary standard allocators. This swap exchanges their
storage without copying elements. Be precise about the old view: **swap alone
does not destroy the old payload**. After this swap, `old_view` refers to the
payload now owned by `candidate`, not to `installed`'s new elements. When
`candidate` is destroyed at function exit, that old view becomes unusable.
The program does not access it afterward. This is the same resource reasoning
as Chapter 4's replacement, but the old candidate's lifetime is visible here. See the
[container swap and reference-validity requirements](https://eel.is/c++draft/container.requirements.general).

We do not claim this is a rollback facility for arbitrary user code. A comparator
that writes input elements through another alias violates the read-only
contract. A copy constructor that steals from a const source violates the
intended value-copy contract even if a declaration looks callable. An allocation
failure or a well-behaved throwing copy still releases the local result through
its vector owner; this chapter's **runnable failure fixture** specifically
injects a comparator exception, not allocator failure.

## Connect the new program to the cumulative build

Finish the executable:

```cpp title="exercises/chapter-05/merge_demo.cpp"
}  // namespace strata

int main() {
    try {
        if (!strata::check_integer_merge()
            || !strata::check_boundaries()
            || !strata::check_stability()
            || !strata::check_captured_direction()
            || !strata::check_order_laws()
            || !strata::check_unsorted_input()
            || !strata::check_comparator_failure()) {
            return 1;
        }
    } catch (const std::exception& error) {
        std::cerr << "UNEXPECTED " << error.what() << '\n';
        return 1;
    }
    std::cout << "foundation-05 passed\n";
    return 0;
}
```

Each check prints its own successful observation only after verifying it. A
false return stops the sequence; an unexpected standard exception reports its
message and fails the process. Required work does not live inside `assert`.
An unhandled nonstandard exception also cannot be a successful test; it is not
translated into an empty result.

Append the target below to the existing root CMake file. Do not replace earlier
targets and do not compile `non_callable.cpp` as part of this executable:

```cmake title="CMakeLists.txt"
add_executable(foundation-05 exercises/chapter-05/merge_demo.cpp)
target_compile_features(foundation-05 PRIVATE cxx_std_20)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(foundation-05 PRIVATE
        -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion
    )
endif()
add_test(NAME foundation-05 COMMAND foundation-05)
```

The quoted include `"ordered_merge.hpp"` resolves beside `merge_demo.cpp`; the
header supplies its own includes and complete definitions. There is one new
`main`, compiled into one new target. No separately compiled template library
or additional runtime package is required.

With the native compiler already selected, configure and run the Debug suite:

```sh title="Terminal — Debug build and cumulative tests"
cmake -S . -B build/debug -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/debug --parallel 4
ctest --test-dir build/debug --output-on-failure
./build/debug/foundation-05
```

Use a fresh build directory rather than changing compilers or generators inside
an incompatible existing CMake cache. The architecture option is for this
native Mac build, not an instruction to cross-compile from another platform.
The cumulative suite now has **six** tests: the language probe plus
`foundation-01` through `foundation-05`.

Expected output from the new executable:

```text title="Expected foundation-05 output"
merge: 1 2 4 4; comparisons=5
boundaries: empty, overlap, and checked sizes passed
stable tags: a x b c y z
captured order: 4 4 2 1
order samples: < has no witness; <=, !=, cycle, gap rejected
unsorted input: left and right rejected
throw on call 5: installed 90 91 preserved
retry: 1 2 4 4
foundation-05 passed
```

Run the same checks in the optimized configuration:

```sh title="Terminal — Release build and cumulative tests"
cmake -S . -B build/release -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_COMPILER="$CXX" \
  -DCMAKE_OSX_ARCHITECTURES=arm64
cmake --build build/release --parallel 4
ctest --test-dir build/release --output-on-failure
```

No output above is a benchmark. The comparison count describes these concrete
loops and fixtures; it does not claim a throughput, allocator implementation,
or native-machine measurement.

## Reject a non-callable argument at compilation

Create the isolated negative source:

```cpp title="exercises/chapter-05/non_callable.cpp"
#include "ordered_merge.hpp"
#include <array>
#include <cstdint>

int main() {
    const std::array<std::uint32_t, 2> left{1, 4};
    const std::array<std::uint32_t, 2> right{2, 4};
    const auto result = strata::ordered_merge<std::uint32_t>(left, right, 7);
    return result.empty() ? 1 : 0;
}
```

`T` is supplied explicitly, so failure should concern the comparator, not
array-to-span deduction. `Compare` is deduced as `int`, and the concept attempts
the expression `before(a,b)` through a const integer reference. That expression
is invalid; the constrained merge candidate is not viable.

Use a syntax-only compilation and inspect the diagnostic:

```sh title="Terminal — expected compilation rejection"
mkdir -p build/diagnostics
if "$CXX" -std=c++20 -Wall -Wextra -Wpedantic -fsyntax-only \
    exercises/chapter-05/non_callable.cpp \
    >build/diagnostics/non_callable.txt 2>&1; then
    printf '%s\n' 'UNEXPECTED: non-callable comparator compiled'
else
    cat build/diagnostics/non_callable.txt
fi
```

Expected behavior is a nonzero compiler status with a diagnostic about no
matching constrained function or unsatisfied `BooleanComparator`, eventually
pointing at the attempted call of an integer. Exact wording varies by compiler.
A missing header, an unrecognized language option, or a syntax error elsewhere
is **not** evidence that this concept worked. The same header must compile in
the normal executable, and the negative diagnostic must identify this argument.

This file stays outside CMake's normal source list, so a deliberately invalid
program cannot break every later build. Restore no hidden setting afterward;
it never changed the main target.

## Break stability without corrupting memory

Make one temporary edit in the merge's main loop. Replace only its condition:

```cpp title="Temporary diagnostic edit — right side wins equivalence"
if (!before(left[i], right[j])) {
```

Leave both branch bodies as written. The new condition takes the right side
when the left side is not strictly earlier, including on equivalence. It still
makes progress, stays in bounds, and produces the same four integers for the
first fixture. However, the tagged result becomes `a,x,y,z,b,c`, so the check
labelled `equivalent-key order` must fail.

Build and run the focused test in both configurations while the defect is
present:

```sh title="Terminal — expected stability failure"
cmake --build build/debug --target foundation-05 --parallel 4
ctest --test-dir build/debug -R '^foundation-05$' --output-on-failure
cmake --build build/release --target foundation-05 --parallel 4
ctest --test-dir build/release -R '^foundation-05$' --output-on-failure
```

The executable should exit with status 1 and print `FAIL equivalent-key order`.
CTest reports a failed test; its own nonzero status need not be the executable's
status. No memory error is required to expose this defect. A sanitizer checks
certain memory or language errors, not the ordering rule that left-side ties
precede right-side ties.

Restore the original condition exactly:

```cpp title="Restore the stable merge condition"
if (before(right[j], left[i])) {
```

Rebuild both configurations and run their complete suites. The passing source,
not the temporary tie defect, is the prerequisite for the next chapter.

## Run a focused memory diagnostic on the restored code

Use the selected compiler's supported AddressSanitizer and
UndefinedBehaviorSanitizer instrumentation on the normal program:

```sh title="Terminal — normal merge with sanitizers"
"$CXX" -std=c++20 -O1 -g -fno-omit-frame-pointer \
  -Wall -Wextra -Wpedantic -Wconversion -Wsign-conversion \
  -fsanitize=address,undefined \
  exercises/chapter-05/merge_demo.cpp \
  -o build/diagnostics/foundation-05-sanitized
./build/diagnostics/foundation-05-sanitized
```

Expected behavior is the same nine output lines and no sanitizer diagnostic.
The intentionally thrown comparator exception is caught by the program and is
not a sanitizer error. The bad ordering predicates are used only by our safe
sampler, so their mathematical failures are ordinary checked data, not an excuse
to trigger undefined behavior in a standard algorithm. Consult the official
[AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) and
[UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html)
documentation for supported toolchain/platform combinations and limitations.

A clean run does not prove all template instantiations correct or establish
allocator accounting for the whole process. It checks the concrete executable
and paths exercised here. No deliberate dangling access is needed in this
chapter.

## Diagnose the boundary that actually failed

| Observation | Likely cause | Focused correction |
|---|---|---|
| Template deduction cannot match an array to a span | `T` was not supplied and arguments do not match the parameter pattern | Supply `ordered_merge<Number>` or exact dynamic read-only spans |
| A callable fails the concept | Non-const call operator or a result type other than `bool` | Inspect the intended callable signature; do not erase the constraint blindly |
| Integers look correct, tags are reversed on ties | Wrong branch owns the equivalence case | Ask whether the **right** value is strictly before the left |
| Counter reports five when three were expected | Input validation was omitted from the cost model | Account for both sortedness scans separately from merging |
| Sortedness rejects descending data | Inputs and comparator use different directions | Establish one comparator and sort/order both inputs under it |
| Installed data changes after a comparison failure | Caller committed before preparation finished or callback violated the input contract | Keep the candidate private and locate every external write |
| Template declared in a header but unavailable at use/link time | Definition was hidden without arranging needed instantiations | Keep these generic definitions in the included header |
| A sampled comparator passes but later behaves inconsistently | Sample incompleteness or mutable ordering state | Reason about the domain and repeatability, not just the Boolean interface |

A practical review question is: what would the test detect if two equivalent
records exchanged places? For integers alone, nothing distinguishes their
origins. The tagged fixture turns the ordering promise into an observable
payload property. The runtime failure test similarly turns “exception-safe”
into a statement about one exact installed vector, its address, and its view.

## What carries forward

Keep the complete `ordered_merge.hpp`, normal `merge_demo.cpp`, isolated
`non_callable.cpp`, and appended `foundation-05` target. The template returns
an owned value from read-only borrowed inputs, uses one const-callable Boolean
comparator consistently, validates representable counts and input order, and
prefers the left input on equivalence. It does not retain inputs or comparator
state. The ordering sampler remains a small diagnostic in the executable, not
a mandatory cubic step in the merge.

The first five chapters now supply native builds, values and views, lifetime
and RAII, moves, failure-safe preparation, and constrained generic interfaces.
They do not yet supply a database. The next section of the
[roadmap](https://25d-228.github.io/cplusplus-learning/index.html) starts the
engine's record encodings and dependable file boundary. Its byte arithmetic,
resource owner, and failure reporting will build on these foundations rather
than hide those mechanisms behind a storage library.

## Technical references

These references are primary specifications and official tool documentation.
The live C++ working draft includes facilities newer than C++20; the chapter
uses only the C++20-compatible syntax and library operations shown here.

- C++ working draft: [template parameters](https://eel.is/c++draft/temp.param),
  [implicit instantiation](https://eel.is/c++draft/temp.inst),
  [name resolution and dependent types](https://eel.is/c++draft/temp.res),
  [function-call deduction](https://eel.is/c++draft/temp.deduct.call), and
  [constrained declarations](https://eel.is/c++draft/temp.constr.decl).
- C++ working draft: [requires expressions](https://eel.is/c++draft/expr.prim.req),
  [compound requirements](https://eel.is/c++draft/expr.prim.req.compound),
  [copy construction as a concept](https://eel.is/c++draft/concept.copyconstructible),
  [type transformations](https://eel.is/c++draft/meta.trans.other), and
  [inline definitions](https://eel.is/c++draft/dcl.inline).
- C++ working draft: [lambda closure types](https://eel.is/c++draft/expr.prim.lambda.closure),
  [captures](https://eel.is/c++draft/expr.prim.lambda.capture),
  [strict weak ordering](https://eel.is/c++draft/concept.strictweakorder),
  [equality preservation](https://eel.is/c++draft/concepts.equality), and
  [merge semantics](https://eel.is/c++draft/alg.merge).
- C++ working draft: [vector capacity](https://eel.is/c++draft/vector.capacity),
  [element insertion](https://eel.is/c++draft/vector.modifiers),
  [swap and container guarantees](https://eel.is/c++draft/container.requirements.general),
  [the Boolean specialization](https://eel.is/c++draft/vector.bool.pspc), and
  [return statements](https://eel.is/c++draft/stmt.return).
  Clang: [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) and
  [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html).
