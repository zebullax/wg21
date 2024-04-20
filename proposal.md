---
title: Proposal for attributes introspection
date: today
audience:
  - sg7
author:
  - name: Aurelien Cassagnes
    email: <aurelien.cassagnes@gmail.com>
---
# Introduction

Attributes are used to great extent and there likely will be attributes added as the language evolve.
What is missing now is a way for generic code to look into the attributes related to an entity.
A motivating example is following

```cpp
[[nodiscard]] bool foo(int i) { return i % 2 ; }

template <class F, class... Args>
constexpr std::invoke_result_t<F, Args...> logInvoke(F&& f, Args&&... args) {
    // Do some extra work, log the call...
    // Fwd call
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}

int main() {
    foo(0); // Warning on discarded return
    logInvoke(foo, 0); // No warning on discarded return
}
```

Other examples of wrapping around callables can be found, whether by closure or explicity registering callbacks for dispatch, etc.
Other applications can easily be thought of in the context of code injection [@p2237r0] where one may want to skip over `[[deprecated]]` members

# Proposal

We put ourselves in the context of [@p2996r2] for the proposal to be more illustrative.

## std::meta::info
We propose that attributes be a supported *reflectable* property of the expression that are reflected upon. That means `std::meta::info` should be augmented to represent an attribute in addition to what it can represent.

## Metafunctions
We propose to add a function to what is discussed already in [@p2996r2]

```cpp
  template<typename E>
    consteval auto attributes_of(E entity) -> vector<info>;
```
This being applied to an entity `E` will yield a sequence of `std::meta::info` representing the attributes attached to `E`.

## Queries
We do not think it is the best option to introduce a query per attribute such as 
```cpp
  // ...
  consteval bool is_deprecated(info type);
  consteval bool is_nodiscard(info type);
  // ...
```
We would rather see a query able to flag whether a particular attribute belongs to what `attributes_of` yields.

# Discussion

Originally the idea of introducing a `declattr(Expression)` keyword seemed the most straightforward to tackle on this problem, but from feedback the concern of introspecting on expression attributes was a concern that belongs with the reflection SG. The current proposal shifted away from the original `declattr` idea to align better with the reflection toolbox.

Another item of discussion is whether vendor specific attributes should be supported or only the ones described by the standard.

---
references:
  - id: p2996r2
    citation-label: Reflection
    title: "Reflection for c++26"
    author:
      - family: Childers
        given: Wyatt
      - family: Dimov
        given: Peter
      - family: Katz
        given: Dan
      - family: Revzin
        given: Barry
      - family: Sutton
        given: Andrew
      - family: Vali
        given: Faisal
      - family: Vandevoorde
        given: Daveed
    URL: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r2.html
  - id: p2237r0
    citation-label: Metaprogramming
    title: "Metaprogramming"
    author:
      - family: Sutton
        given: Andrew
    URL: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p2237r0.pdf
---