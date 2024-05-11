---
title: Attributes introspection
date: today
audience:
  - sg7
author:
  - name: Aurelien Cassagnes
    email: <aurelien.cassagnes@gmail.com>
---
# Introduction
The current draft aims at getting the conversation and work started on supporting introspection of attributes. There is ongoing work to refine that draft, especially when it comes to motivating examples.

# Motivation

Attributes are used to great extent and there likely will be attributes added as the language evolve.
What is missing now is a way for generic code to look into the attributes appertaining to an entity.
A motivating toy example is the following

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
Ideally we would want a mechanism to recover the `[[nodiscard]]` attribute that originally appertained to `foo` declaration.
Other examples of wrapping around callables can be found, whether by closure or explicity registering callbacks for dispatch, etc.
The current example deals with code as it is in a c++23 world, but other applications can easily be thought of in the context of code injection [@P2237R0] where one may want to skip over `[[deprecated]]` members for example.

# Optionality rule
Currently the standard notes in ([dcl.attr.grammar]) that (broadly speaking) attributes are ignorable. Let us note

- [@CWG2538] discusses changing this to restrict ignorability to `non standard`, and make program ill-formed when attribute semantic is violated
- [@P2552R3] discusses at length how underspecified this wording is, especially on what 'ignorability' really mean
This proposal agrees with the change proposed in [@CWG2538]. We also feel that whether an implementation decide to semantically ignore a standard  attribute should not matter.\

What matters more is self-consistency, when introspecting an entity:

- We should be able to discover appertaining attributes (syntactically mandatory)
- Declaring an entity with those discovered attributes yield the same result as what the implementation offer to natively (semantically optional)

# Proposal
We put ourselves in the context of [@P2996R2] for the current proposal to be more illustrative in terms of what is being proposed.

## Scope
Attributes are split into standard and non standard. This proposal wishes to limit itself to standard attributes ([dcl.attr]). We feel that since it is up to implementation to define how they handle non standard attributes, it would lead to obscure situations that we don't claim to tackle here.\
A fairly (admittedly ridiculous) example can be built as such: Given an implementation supporting a non standard `[[no_introspect]]` attributes that suppress all reflection information appertaining to an entity, we would have a hard time coming up with a self-consistent system of rules to start with.

## Reflection operator
If our understanding is correct, the proposition for `^` grammar does not cover attributes , as in `^[[deprecated]]` is meaningless. We think this will limit the potential use of attributes introspection. The current proposal advocates for 
```
^ attribute
```
to be well formed.

## Splicers
We propose that the form
```cpp
attribute [: r :]
```
be supported. This implicitly means that `std::meta::info` definition must be extended, this will be discussed thereafter. 

- `attribute [: r :]` produces a potentially empty sequence of attributes corresponding to the attributes that are attached to `r`

## std::meta::info
We propose that attributes be a supported *reflectable* property of the expression that are reflected upon. That means value of type `std::meta::info` should be able to represent an attribute in addition to the current supported set.

## Metafunctions
We propose to add a metafunction to what is discussed already in [@P2996R2]

```cpp
  template<typename E>
    consteval auto attributes_of(E entity) -> vector<info>;
```
This being applied to an entity `E` will yield a sequence of `std::meta::info` representing the attributes attached to `E`. In particular we think this addresses the case where `attribute-list` is of the form `[[ attribute... ]]`.

## Queries
We do not think it is necessary to introduce query or queries at this point. Especially we would not recommend to introduce a dedicated query per attribute (eg `is_nodiscard`, `is_nouniqueaddress`, etc.)

# Discussion

Originally the idea of introducing a `declattr(Expression)` keyword seemed the most straightforward to tackle on this problem, but from feedback the concern of introspecting on expression attributes was a concern that belongs with the reflection SG. The current proposal shifted away from the original `declattr` idea to align better with the reflection toolbox. Note also that as we advocate here for `attribute [: r :]` to be supported, we recover the ease of use that we first envisioned `declattr` to have.
<!-- 
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
    -id: p2552r3
    citation-label: Optionality rule
    title: "On the ignorability of standard attributes"
    author:
      - family: Doumler
        given: Timur
    URL: https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2552r3.pdf
    -id: CWG2358
    citation-label: CWG2358
    title: "Can standard attributes be syntactically ignored?"
    author:
      - family: Maurer
        given: Jens
    URL: https://cplusplus.github.io/CWG/issues/2538.html
--- -->