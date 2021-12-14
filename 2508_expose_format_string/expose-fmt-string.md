---
title: "Expose `std::$basic-format-string$<charT, Args...>`"
document: D2508R0
date: today
audience: LEWG
author:
    - name: Barry Revzin
      email: <barry.revzin@gmail.com>
toc: true
---

# Introduction

[@P2216R3], in order to improve type safety around the `std::format` API, introduced the type `std::$basic-format-string$<charT, Args...>` as a way to check to make sure that the format string actually matches the arguments provided. The example used in that paper is:

::: bq
```cpp
std::string s = std::format("{:d}", "I am not a number");
```
:::

Originally, this would throw an exception at runtime. With the change, this now is a compile error. Which is a very nice improvement - catching bugs at compile time is awesome.

Unfortunately, the ability to validate format strings at compile time is currently limited to specifically those functions provided by the standard library. In particular, `std::format`, `std::format_to`, `std::format_to_n`, and `std::formatted_size`. But users cannot take advantage of this in their own code.

As Joseph Thompson pointed out in [@std-discussion]:

::: bq
However, consider the common use case:

```cpp
template <typename... Args>
void log(std::format_string<Args...> s, Args&&... args) {
    if (logging_enabled) {
        log_raw(std::format(s, std::forward<Args>(args)...));
    }
}
```

Without a specified `std::format_string` type, is it possible to forward
arguments to `std::format` with compile-time format string checks? I'm
thinking about this as an average user of the standard library, so I'm
not looking for workarounds. Is this a defect?
:::

Today, if users want to statically verify their format strings... what can they do? Originally, I thought that you could write a macro to wrap the call to `log`, so that you can attempt to pass the expressions as they are to try to statically verify them. But this compiles just fine:

```cpp
using T = decltype(std::format("{:d}", "I am not a number"));          // ok
static_assert(requires { std::format("{:d}", "I am not a number"); }); // ok
```

Because... of course it compiles. The way that this check works in format today is that `$basic-format-string$<charT, Args...>` has a `consteval` constructor that just isn't a constant-expression if the value of the format string doesn't match the arguments.

 But constant-expression-ness (or, in general, values) don't affect overload resolution, so there's no way to check this externally (there were some papers that considered value-based constraints which would've let this work [@P1733R0] [@P2049R0], which aren't without potential issue [@P2089R0], but we don't have something like that today).

The only way for Joseph (or you or me or ...) to get this behavior in `log` is to basically manually re-implement `$basic-format-string$`. That seems like an unreasonable burden to place for such a useful facility. The standard library already has to provide it, why not provide it under a guaranteed name that we can use?

## Why exposition-only?

There are two reasons why `$basic-format-string$<charT, Args...>` (and its more specialized alias templates, `$format-string$<Args...>` and `$wformat-string$<Args...>`) are exposition-only.

The first is that, as a late DR against C++20, [@P2216R3] was trying to limit its scope. Totally understandable.

The second is that it is possible with future language features to be able to do this better. For example, if we had `constexpr` function parameters [@P1045R1] or some kind of hygienic macro facility [@P1221R1], then the whole shape of this API would be different. For `constexpr` function parameters, for instance, the first argument to `std::format` would probably be a `constexpr std::string` (or some such) and then there would just be a `consteval` function you could call like `validate_format_string<Args...>(str)`. In which case, we could expose `validate_format_string` and the implementation of `log` could call that directly too. And that would be pretty nice!

But we won't have `constexpr` function parameters or a hygienic macros in C++23, so the solution that we have to this problem today is `$basic-format-string$<charT, Args...>`. Since we have this solution, and it works, we shouldn't just restrict it to be used by the standard library internally. If eventually we adopt a better way of doing this checking, we can expose that too. Better give users a useful facility today that's possibly worse than a facility we might be able to provide in C++26, rather than having them have to manually write their own version (which is certainly doable, though tedious and error-prone).

## Yet another DR against C++20, are you serious?

This doesn't strictly have to be a DR, and could certainly just be a C++23 feature. Although it would be nice to have sooner rather than later. At least it obviously has no ABI impact (`std::basic_format_string<char, Args...>` can just be an alias template for `std::_Ugly::_Basic_Format__Demand__More__Underscores<char, Args...>`, the original need not be renamed. Although I don't think anybody implements this yet) so I leave it up to the discretion of whether or not we ever actually want to declare C++20 complete.

# Wording

In [format.syn]{.sref}, replace the exposition-only names `$basic-format-string$`, `$format-string$`, and `$wformat-string$` with the non-exposition-only names `basic_format_string`, `format_string`, and `wformat_string`.

Do the same for their uses in [format.fmt.string]{.sref} (renaming the clause to "Class template `basic_format_string`) and [format.functions]{.sref}.

In [format.fmt.string]{.sref}, the member should still be exposition only. The full subclause should now read (the only change is the name of the class template, which no longer is exposition only):

::: bq
```cpp
template<class charT, class... Args>
struct basic_format_string {
private:
  basic_string_view<charT> $str$;         // exposition only

public:
  template<class T> consteval basic_format_string(const T& s);
};
```

```cpp
template<class T> consteval basic_format_string(const T& s);
```
[#]{.pnum} *Constraints*: `const T&` models `convertible_to<basic_string_view<charT>>`

[#]{.pnum} *Effects*: Direct-non-list-initializes `$str$` with `s`.

[#]{.pnum} *Remarks*: A call to this function is not a core constant expression ([expr.const]) unless there exist `args` of types `Args` such that `$str$` is a format string for `args`.
:::

## Feature-test macro

It is definitely important to provide a feature-test macro for this, which will allow `log` above to conditionally opt in to this feature if possible:

:::bq
```cpp
#if __cpp_lib_format >= whatever
    template <typename... Args>
    using my_format_string = std::format_string<std::type_identity_t<Args>...>;
#else
    template <typename... Args>
    using my_format_string = std::string_view;
#endif

template <typename... Args>
void log(my_format_string<Args...> s, Args&&... args);
```
:::

Bump the `format` feature-test macro in [version.syn]{.sref}:

::: bq
```diff
- #define __cpp_­lib_­format    @[202110L]{.diffdel}@ // also in <format>
+ #define __cpp_­lib_­format    @[2022XXL]{.diffins}@ // also in <format>
```
:::


---
references:
  - id: std-discussion
    citation-label: std-discussion
    title: "Should a `std::basic_format_string` be specified?"
    author:
      - family: Joseph Thomson
    issued:
      year: 2021
    URL: https://lists.isocpp.org/std-discussion/2021/12/1526.php
---