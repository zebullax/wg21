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

## Earlier work
While collecting feedback on this draft, we were redirected to [@P1887R1] as a pre existing proposal. In this paper the author discusses two topics 'user defined attributes' (also in [@P2565R0]) and reflection over said attributes. We believe the two topics need not be conflated together, both have intrinsic values on their own. We aim here to focus the discussion entirely on **standard** attributes reflection.

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
There is a long standing and confusing discussion around how ignorable are attributes allowed to be. We'll refer the reader to [@P2552R3] for an at-length discussion of this problem, and especially on what 'ignorability' really mean. This proposal agrees with the discussion carried there and in [@CWG2538]. We also feel that whether an implementation decide to semantically ignore a standard attribute should not matter.\

What matters more is self-consistency, when introspecting an entity:

- We should be able to discover appertaining attributes
- Declaring an entity with attributes discovered via introspection yield the same result as what the implementation would offer if directly declaring that entity with those attributes.

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

- `attribute [: r :]` produces a potentially empty sequence of attributes corresponding to the attributes that pertain to `r`

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
We do not think it is necessary to introduce additional query or queries at this point. Especially we would not recommend to introduce a dedicated query per attribute (eg `is_deprecated`, `is_nouniqueaddress`, etc.).\
Having said that, we feel those should be acheivable via concept for attributes inspected statically.
```cpp
template<class T>
concept IsDeprecated = std::ranges::any_of(
  attributes_of(^T),
  [] (auto meta) { meta == ^[[deprecated]]; }
);
```

## Applications
The applications here build on the adoption of earlier work, it is understood that they are not working examples *as of now*. The discussion is on the value-add of having those work.

> *There Be well fleshed out applications*

<!-- ### [[noreturn]]
Spin a thread to run that never returning function.
```cpp
void launch(Callable auto callback) {
  auto callbackAttributes = attributes_of(^callback);
  if (std::ranges::any_of(callbackAttributes, [](auto attributeMeta) {attributeMeta == ^[[noreturn]]})) {
    // Run on a dedicated thread and detache
  } else {
    callback();
  }
} -->

# Discussion

Originally the idea of introducing a `declattr(Expression)` keyword seemed the most straightforward to tackle on this problem, but from feedback the concern of introspecting on expression attributes was a concern that belongs with the reflection SG. The current proposal shifted away from the original `declattr` idea to align better with the reflection toolbox. Note also that as we advocate here for `attribute [: r :]` to be supported, we recover the ease of use that we first envisioned `declattr` to have.
