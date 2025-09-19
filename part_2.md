# chapter 9 - class template argument deduction (CTAD)

Omit explicit definition of template arguments if the constructor is able to
deduce them:

```cpp
std::mutex mtx;
std::lock_guard lg{mtx};

std::vector v1{1, 2, 3};          // <int>
std::vector v1{"hello", "world"}; // <const char*>
std::complex c{5.1, 3.3};         // <double>
std::complex c{5, 3.3}            // ERROR: ambiguous

std::tuple t{42, 'x', nullptr};   // <int, char, std::nullptr_t>
```

Deduce non-type template:

```cpp
template<typename T, int SIZE>
class my_class {
public:
  my_class(T (&)[SIZE]) {}
};

my_class mc("hello"); // deduces T as const char and SIZE as 6
```

Copy initialization is preferred over the "raw" template type:

```cpp
std::vector v1{42};
std::vector v2{v1};     // copy init
std::vector v3{v1, v2}; // CTAD

template<typename... Args>
auto make_vector(const Args&... elems) {
  return std::vector{elems...};
}
std::vector<int> v{1, 2, 3};
auto x1 = make_vector(v, v); // vector<vector<int>>
auto x2 = make_vector(v); // implementation defined:
                          // vector<int> / vector<vector<int>>
```

## deduction guides

```cpp
template<typename T1, typename T2>
struct Pair {
  T1 first;
  T2 second;
  Pair(const T1& x, const T2& y) : first{x}, second{y} { }
};
```

Without deduction guides:

```cpp
Pair p{"hi", "world"}; // non-const char[]
```

> `T1` and `T2` are deduced as plain `char []` because of the `const` qualifier
> and the reference in the constructor: arguments passed by value *decay*, while
> they won't when passed by reference

With deduction guide:

```cpp
template<typename T1, typename T2>
Pair(T1, T2) -> Pair<T1, T2>;

Pair p{"hi", "world"}; // const char*
```

> the deduction guide takes the parameters by value, so that `T1` and `T2` are
> deduced as `const char*`

- on the left side of the `->`: declare what is needed to be deduced
- on the right side of the `->`: define the resulting deduction

Deduction guide thus creates a "mapping" for types to be deduced as desired:

```cpp
template<typename T>
struct S {
  T val;
};

S(const char*) -> S<std::string>; // map S<> for string literals to S<std::string>
```

> thus it competes with the existing constructors and is preferred because it
> has the **highest** priority according to *overload resolution*

---

# chapter 10 - compile-time `if`

```cpp
if constexpr () {}
```

> compile-time branch evaluation

## caveats

### return type might be int or void

```cpp
auto foo() {
  if constexpr (sizeof(int) > 4)
    return 42;
}
```

### without `else` for the second/default branch is UB

```cpp
auto foo() {
  if constexpr (sizeof(int) > 4) 
    return 42;
  return 42u;
}
```

### no "short-circuiting": expression must be valid as a whole

```cpp
template<typename T>
constexpr auto bar(const T& val)
{
  if constexpr (std::is_integral<T>::value && T{} < 10)
    return val * 2;
  return val;
}
```

> the `constexpr if` fails and results in an compile-time error when passing a
> type that's not an integral and less than `10`

---

# chapter 11 - fold expressions

> fold expressions compute result of using a binary operator over all the
> arguments of a parameter pack (ellipsis)

```cpp
template<typename... T>
auto unary_left_fold(T... args) {
  return (... + args); // ((arg1 + arg2) + arg3) ...
}

auto unary_right_fold(T... args) {
  return (args + ...); // (arg1 + (arg2 + arg3)) ...
}

left_fold(std::string("hello"), "world", "!");  // OK
left_fold("hello", "world", std::string("!"));  // ERROR
right_fold(std::string("hello"), "world", "!"); // ERROR
```

TODO binary fold syntax p. 110

- used as an iterative recursion
- when used with an empty parameter pack:
	- with `&&`: the result is true
	- with `||`: the result is false
	- with `,`-operator: the result is `void()`
	- with all other operators: ill-formed

---

# chapter 12 - string literals as template parameters

wtf

---

# chapter 13 - placeholders like `auto` as non-type template parameter types

```cpp
template<auto N> class S {};

S<42> s1;
S<'a'> s2;

// partial specialization
template<int N> class S<N> {};

// example:
template<typename auto v>
struct constant {
  static constexpr auto value = v;
};
// instead of
template<typename T, T v>
struct constant {
  static constexpr T value = v;
};
```

- support CTAD, cv-qualifiers, variadic templates

## `decltype(auto)`

- `decltype(expression)` yields:
	- `type` for a prvalue (e.g., temporaries)
	- `type&` for an lvalue (e.g., objects with names)
	- `type&&` for an xvalue (e.g., objects marked with `std::move()`)

> so the deduced type via `decltype(auto)` might be one of those types with
> additional side effects

### example

```cpp
#include <iostream>

template<decltype(auto) N>
struct S {
  void printN() const { std::cout << "N: " << N << '\n'; }
};

static const int c = 42;
static int v = 42;

int main()
{
  S<c> s1;     // deduces N as const int 42
  S<(c)> s2;   // deduces N as const int& referring to c!
  S<(v)> s3;   // deduces N as int& referring to v!
  v = 77;
  s3.printN(); // prints: N: 77
}
```

---

# chapter 14 - extended `using` declarations

```cpp
using std::cin, std::cos, std::tan; // instaed of three different ones
```

## variadic using declarations

i wonder when this will be useful...
