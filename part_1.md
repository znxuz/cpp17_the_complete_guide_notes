# chapter 1 - structured bindings

Decompose the objects passed for initialization:

```cpp
struct my_struct {
    int i = 0;
    std::string s{};
};
auto [u, v] = ms;
```

Behind the curtain:

```cpp
auto [u, v] = ms;
// equals
auto e = ms;
aliasname u = e.i;
aliasname v = e.s;
```

- qualifiers `const` and `&` apply only to the anonymous object `e`:
  `decltype(u)` or `decltype(v)` don't aren't reference types
- same applies to `alignas(n)`: the whole object is aligned, not the individual
  type


## as rvalue reference

```cpp
MyStruct ms = { 42, "Jim" };

// an anonymous entity being an rvalue reference to ms
auto&& [v,n] = std::move(ms);

// while ms still holds its value:
std::cout << "ms.s: " << ms.s << '\n'; // prints "Jim"

std::string s = std::move(n);
std::cout << "ms.s: " << ms.s << '\n'; // UB
std::cout << "n: " << n << '\n';       // UB
std::cout << "s: " << s << '\n';       // prints "Jim"
```

- can be used:
	- for structures and classes
	- on each element of raw arrays; by copy
	- for any type with a tuple-like API with components:
		- `std::tuple_size<type>::value`: is the number of elements
		- `std::tuple_element<idx, type>::type`: is the type of the `idx`th
		  element
		- `get<idx>()`: yields the value of the `idx`th element
- bind multiple times to `_` not supported; *damn rust you win this time*

## assigning new values to structured bindings for `pair` and `tuple`

```cpp
auto [a, b, c] = getTuple();
// ...
std::tie(a, b, c) = getTuple();
```

## providing API for structured bindings

```cpp
#include <string>
#include <utility>

class Customer {
private:
  std::string first;
  std::string last;
  long val;
public:
  Customer (std::string f, std::string l, long v)
  : first{std::move(f)}, last{std::move(l)}, val{v} { }
  std::string getFirst() const { return first; }
  std::string getLast() const { return last; }
  long getValue() const { return val; }
};

// provide a tuple-like API for class Customer for structured bindings:

template<>
struct std::tuple_size<Customer> {
  static constexpr int value = 3; // we have 3 attributes
};

template<>
template<std::size_t Idx>
struct std::tuple_element<Idx, Customer> {
  using type = std::string; // the other attributes are strings
};

struct std::tuple_element<2, Customer> {
  using type = long; // the 2.(last) attribute is a long
};

// define specific getters:
template<std::size_t> auto get(const Customer& c);
template<> auto get<0>(const Customer& c) { return c.getFirst(); }
template<> auto get<1>(const Customer& c) { return c.getLast(); }
template<> auto get<2>(const Customer& c) { return c.getValue(); }
// or use `constexpr if` to aggregate the specializations into a single function

// template<> long get<2>(const Customer& c) { return c.getValue(); }
// compile error: the specialization has to adhere to the template signature
// it derives from
```

### For write-only

Overload `get<>` for `const`, non-`const` and forwarding(`&&`) reference:

```cpp
template<std::size_t> decltype(auto) get(const Customer& c);
template<std::size_t> decltype(auto) get(Customer& c);
template<std::size_t> decltype(auto) get(Customer&& c);
```

---

# chapter 2 - if/switch initialization

## `if`

```cpp
if (std::lock_guard<std::mutex> _{collMutex}; !coll.empty()) {}
if (auto x{1}, y{2}; x != y)
if (auto [pos, ok] = map.insert({"new", 42}); !ok)
```

## `switch`

```cpp
switch (std::filesystem::path p{name}; status(p).type()) {
  case std::filesystem::file_type::not_found:
  ...
}
```

---

# chapter 3 - inline variables

- declare **and** define a variable/object in a header file by using `inline`
- in general: `inline` makes the linker to find all references and then refer to
  a single object
- `constexpr` implies `inline` for static members
- paired with `thread_local` to have a copy for each thread

---

# chapter 4 - aggregate extensions

```cpp
struct Data {
  const char* name;
  double value;
};
struct CppData : Data {
  bool critical;
}
CppData x1{};         // zero-initialize all elements
CppData x2{{"msg"}};  // same as {{"msg",0.0},false}
CppData x3{{}, true}; // same as {{nullptr,0.0},true}
CppData x4;           // values of fundamental types are unspecified
```

- must adhere to the definition of **aggregates** for leveraging these
  extensions

---

# chapter 5 - copy elision

Too complex to summarize.

---

# chapter 6 - lambdas

Implicitly `constexpr` since c++17

> meaning that the call-operator of the lambda/closure object is `constexpr`

## `constexpr` lambda variable vs. call-operator

```cpp
auto squared1 = [](auto val) constexpr { return val*val; };
constexpr auto squared2 = [](auto val) { return val*val; };
```

> If only the lambda(call-operator) is `constexpr`, it can be used at compile
> time, but its belonging object(`squared1`) might be initialized at run time,
> which means that some problems might occur if the static initialization order
> matters (e.g., causing the *static initialization order fiasco*). If the
> (closure) object initialized by the lambda is `constexpr`, the object is
> initialized when the program starts but the lambda might still be a lambda
> that can only be used at run time (e.g., using static variables).

Therefore, you might consider declaring:

```cpp
constexpr auto squared = [](auto val) constexpr { return val*val; };
```

## `this` capture

- `this` syntax to reference the object (potential lifetime issue)
- `*this` syntax to copy the object into the lambda:

```cpp
class Data {
private:
  std::string name;
public:
  Data(const std::string& s) : name(s) { }

  auto startThreadWithCopyOfThis() const {
    using namespace std::literals;
    std::thread t([*this] {
      std::this_thread::sleep_for(3s);
      std::cout << name << '\n';
    });
    return t;
  }
};
```

- capture by `const` reference: `[&var = std::as_const(ptr)]`

---

# chapter 7 - attributes

- `[nodiscard]`:
	- warning for discarded return values
	- `(void)` to disable
- `[maybe_unused]`: self-explanatory
- `[fallthrough]`: for switch cases which intent to fall through duh
- `[deprecated]`: self-explanatory

---

# chapter 8 - other features

- nested namespaces: `namespace A::B::C`; no `inline` support yet

## defined expression eval order

Previously UBs:

```cpp
s.replace(0,8,"")
	.replace(s.find("even"),4,"sometimes")
	.replace(s.find("you don't"),9,"I");
```

> `find()` might be executed at **any time** while the whole statement is being
> processed

```cpp
std::cout << f() << g() << h();
```

> `f()`, `g()` and `h()` might be called in **any order**

```cpp
i = 0;
std::cout << ++i << ' ' << --i << '\n';
```

> evaluated at **any order**

Now the evaluation order is guarantees for *some* operators:

- left to right:
	- `e1 [ e2 ]`
	- `e1 . e2`
	- `e1 .* e2`
	- `e1 ->* e2`
	- `e1 << e2`
	- `e1 >> e2`
- right to left:
	- `e2 = e1`
	- `e2 += e1`
	- `e2 *= e1`

## relaxed enum init

```cpp
enum MyInt : char {};
MyInt i1{42};    // OK since C++17 (ERROR before C++17)
MyInt i2 = 42;   // still ERROR
MyInt i3(42);    // still ERROR
MyInt i4 = {42}; // still ERROR

// enum class Weekday : char/int { mon, tue, wed, thu, fri, sat, sun };
enum class Weekday { mon, tue, wed, thu, fri, sat, sun };
Weekday s1{0};    // OK since C++17 (ERROR before C++17)
Weekday s2 = 0;   // still ERROR
Weekday s3(0);    // still ERROR
Weekday s4 = {0}; // still ERROR
```

## fixed **direct** list init with `auto`

- `auto a{42}`:
	- previously: is an `std::initializer_list`
	- now: an `int`
- `auto a{1, 2, 3}`:
	- previously: is an `std::initializer_list`
	- now: ERROR

### note:

- direct initialization: `auto a{...};`
- copy initialization: `auto a = {...};`

## hexadecimal floats

Not interested for now

## UTF-8 character literals

- `u8` prefix also supported for char literals now: `auto c = u8'6';`
- `char c = u8'ö';` not allowed: `ö` is two bytes in size
- `u8` for single-byte US-ASCII and UTF-8
- `u` for two-byte UTF-16
- `U` for four-byte UTF-32
- `L` for wide chars without specific encoding (either 2 or 4 bytes)

## exception specification as part of the type

- `noexcept` functions are now a different type but it still can't be used for
  overloading:
	- `noexcept` function pointer can't point to "throwable" functions
	- must be represented using a different template parameter
- conditional `noexcept`:
	- `void f() noexcept(sizeof(int) < 4);`: either `noexcept` or not depending
	  on the predicate
- `throw()` specifier is now deprecated

## `static_assert`

> Jason Turner's fav

## preprocessor condition `__has_include`

```cpp
#if __has_include(<filesystem>)
# include <filesystem>
# define HAS_FILESYSTEM 1
#elif __has_include(<experimental/filesystem>)
# include <experimental/filesystem>
# define HAS_FILESYSTEM 1
# define FILESYSTEM_IS_EXPERIMENTAL 1
#elif __has_include("filesystem.hpp")
# include "filesystem.hpp"
# define HAS_FILESYSTEM 1
# define FILESYSTEM_IS_EXPERIMENTAL 1
#else
# define HAS_FILESYSTEM 0
#endif
```
