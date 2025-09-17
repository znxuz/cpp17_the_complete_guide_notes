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

If only the lambda (call-operator) is `constexpr`, it can be used at compile
time, but its belonging object (`squared1`) might be initialized at run time,
which means that some problems might occur if the static initialization order
matters (e.g., causing the *static initialization order fiasco*). If the
(closure) object initialized by the lambda is `constexpr`, the object is
initialized when the program starts but the lambda might still be a lambda that
can only be used at run time (e.g., using static variables).

Therefore, you might consider declaring:

```cpp
constexpr auto squared = [](auto val) constexpr { return val*val; };
```
