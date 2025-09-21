# chapter 15 - `std::optional`

- memory layout:
	- the underlying type
	- one byte for the additional `bool` flag
	- possible alignment overhead
- *value semantics* -> deep copying
- dedicated `std::nullopt` as a none/empty value
- implicitly convertable to `bool`, or use `.has_value()`
- construct with multiple arguments:
	- `std::in_place` in the ctor, or
	- `std::make_optional`
- access the data:
	- via `*`, UB if none
	- `.value()` with `std::bad_optional_access`
- has [deduction guides](part_2.md#deduction guides): -> CTAD support
- `value_or()` might (move-)construct the given (rvalue) object (support
  move-only types), and allocate memory upon return, while `value()` alone never
  does

## move semantics

```cpp
std::optional<std::string> func();

std::string s1 = func().value();
std::string s2 = *func();
```
> guarantees that the non-nullopt object is moved into

## comparison

- via operators:
	- both operands have a value: comparison passed onto the underlying object
	- both are none: they are treated as equal
	- otherwise the none value is considered "less" than, so

### some nice foot guns

```cpp
std::optional<bool> ob{false};
if (!ob)                        // false: contains a value, equals .has_value()
if (ob == false)                // true: compares against the underlying value

std::optional<int*> op{nullptr};
if (!op)                        // false: contains a value

`std::optional<unsigned>{} < 0` // `true`: none is considered less than values

std::optional<std::optional<std::string>> oos; // optional inception
```

---

# chapter 16 - `std::variant<>`

- same memory layout as `union`
- value semantics
- no empty state: calls the default ctor of the first alternative by default
	- `std::variant<std::string, int> var; // .index() == 0, value == ""`
- use `std::monostate` as placeholder as an 'empty' alternative
- use `std::in_place_type<>` or `std::in_place_index<>` for initializing with
  multiple arguments like `std::variant<std::complex<double>>`:
	- `v{std::in_place_type<std::complex<double>>, 3.0, 4.0};`
	- `v{std::in_place_index<0>, 3.0, 4.0};`
- use `std::in_place_index<>` to solve ambiguity error:
	- `std::variant<int, int> v{std;:in_place_index<1>, 77}; // init 2. int`
- access the value:
	- `std::get<>` supplied with either an index or a (non-duplicated) type
	- `std::get_if<>()` to get whether it exists
- change the value:
	- `=` operator overload: `var = "hello"`
	- `.emplace<>()`: `var.emplace<1>(42);`
	- `std::get<0>(var) = 99;`
	- use the pointer returned by `std::get_if`

## comparison

> only valid for variants of the same declaration type

1. an "earlier" alternative is *less* than a *later* alternative
2. compare against the underlying type upon same alternative

## `valueless_by_exception()`

what is this

## visitors

> an object which provides a function call operator for every possible type

> the best matching function gets called for the actual value of the variant

```cpp
struct MyVisitor
{
  void operator() (int i) const { std::cout << "int: " << i << '\n'; }
  void operator() (string s) const { std::cout << "string: " << s << '\n'; }
  void operator() (double d) const { std::cout << "double: " << d << '\n'; }
};

int main()
{
  std::variant<int, std::string, double> var(42);
  std::visit(MyVisitor(), var); // calls operator() for int
  var = "hello";
  std::visit(MyVisitor(), var); // calls operator() for string
  var = 42.7;
  std::visit(MyVisitor(), var); // calls operator() for long double
}
```

## Modify the underlying value

> via rvalue reference as argument

```cpp
struct Twice
{
  void operator()(double& d) const { d *= 2; }
  void operator()(int& i) const { i *= 2; }
  void operator()(std::string& s) const { s = s + s; }
};
```

## as a lambda

```cpp
std::visit([](auto& val) { val = val + val; }, var);
```

## return values from `std::visit`

```cpp
std::vector<std::variant<int, double>> coll{42, 7.7, 0, -0.7};

double sum{0};
for (const auto& elem : coll) {
  sum += std::visit([] (const auto& val) -> double { return val; }, elem);
}
```

## static polymorphism via `std::visit`

```cpp
using GeoObj = std::variant<Line, Circle, Rectangle>; // common type

std::vector<GeoObj> createFigure()
{
  std::vector<GeoObj> f;
  f.push_back(Line{Coord{1,2},Coord{3,4}});
  f.push_back(Circle{Coord{5,5},2});
  f.push_back(Rectangle{Coord{3,3},Coord{6,4}});
  return f;
}

int main()
{
  std::vector<GeoObj> figure = createFigure();
  for (const GeoObj& geoobj : figure) {
    std::visit([] (const auto& obj) { obj.draw(); }, geoobj);
  }
}
```

### benefits

- static
- no need for a common base type/class
- no pointer handling
- no `virtual`
- value semantics: memories are packed together (in a container for example)

### constraints

- all alternatives have to be known at compile-time
- all have the size of the largest element type
- value semantics: expensive copying

## foot gun: `std::variant<bool, std::string>`

> has been fixed in P0608R3

---

# chapter 17 - `std::any`
