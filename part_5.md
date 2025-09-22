# chapter 29 - polymorphic memory resources (PMR)

skipped

---

# chapter 30 - `new` and `delete` with over-aligned data

skipped

---

# chapter 31 - `std::to_chars()` and `std::from_chars()`

> new low-level functions for converting between integral values and character
> sequences in `<charconv>`

- no run-time parsing of format strings
- no dynamic allocation inherently required by the API
- no consideration of locales
- no function pointers required
- no buffer overruns
- parsing strings: distinguishable errors from valid numbers
- parsing strings: whitespace or decorations aren't silently ignored
- guarantees floating-point round-trip support: converting floating-point
  values and back results in the same value

```cpp
if (auto [ptr, ec] = std::from_chars(str, str+10, value); ec != std::errc{}) {
  // error handling
}

if (auto [ptr, ec] = std::to_chars(str, str+10, value); ec != std::errc{}) {
  // error handling
}
```

---

# chapter 32 - `std::launder()`

skipped for now

PS: the very first sentence of this chapter in the book:

> There is a new library function called std::launder(), which, as far as I
> understand and see it, is a workaround of a core problem. That, however, does
> not really work.

---

# chapter 33/34/35

skimmed through.
