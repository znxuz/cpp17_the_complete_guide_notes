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

TODO find out why this thing even exists - what is it good for?

---

# chapter 18 - `std::byte`

- represents all the values a byte can hold with no numeric or character
  representation -> only bit-wise
- operations:
	- assignment
	- comparison
	- bit-operations
	- `std::to_integer<T>()`
	- `sizeof() == 1`

```cpp
#include <cstddef> // for std::byte

std::byte b1{0x3F};
std::byte b2{0b1111'0000};
std::byte b3[4]{b1, b2, std::byte{1}}; // byte[3] default-initialized to 0

if (b1 == b3[0])
  b1 <<= 1;

std::cout << std::to_integer<int>(b1) << '\n'; // prints 126
```

Implemented as an enumeration type based on `unsigned char`, thus no
assignment/copy initialization, implicit conversion.

```cpp
// std::byte b2(42);    // ERROR
// std::byte b3 = 42;   // ERROR
// std::byte b4 = {42}; // ERROR

// std::byte b5[] {1};  // ERROR
```

## as integral value

```cpp
std::to_integer<int>(b);
std::to_integer<bool>(b);
```

## as binary representation values

Convert to `std::bitset<std::numeric_limits<unsigned char>::digits>` to output
as a sequence of bits:

```cpp
using ByteBitset = std::bitset<std::numeric_limits<unsigned char>::digits>;

std::byte b1{42};
std::cout << ByteBitset(b1);

std::string s = ByteBitset(b1).to_string();

#include <charconv>
char str[100];
std::to_chars_result res = std::to_chars(str, str + sizeof(str) - 1,
                                         std::to_integer<int>b1, 2);
*res.ptr = '\0'; // ensure a trailing null char
```

## read into `std::byte`

```cpp
std::istream& operator>> (std::istream& strm, std::byte& b)
{
  // read into a bitset:
  std::bitset<std::numeric_limits<unsigned char>::digits> bs;
  strm >> bs;
  // without failure, convert to std::byte:
  if (!std::cin.fail())
    b = static_cast<std::byte>(bs.to_ulong()); // because no narrowing w/ enum
  return strm;
}
```

```cpp
#include <charconv>

const char* str = "101001";
int value;
std::from_char_result res = std::from_chars(str, str + std::strlen(str),
                                            value, 2);
```

---

# chapter 19 - string views

> to reiterate: it's just a dumb pointer to a allocated string, so always keep
> in mind that it has to point to a valid string / string literal.

## why `std::string_view` is better than `const std::string&`

- `std::string_view` doesn't allocate: if the passed argument is a `const
  char*`, it would allocate for a `std::string` to "convert" to it
- get substrings from `std::string_view` without copy:
	- `substr()`
	- `remove_prefix/suffix()`

## if you want your `string_view` to dangle

- take `string_view` as a ctor argument: `string_view` would then bind to a
  moved rvalue reference of a `string` passed as the ctor argument
- return a `string_view` from a getter: if invoked on a temporarily created
  object, the underlying `string` object is destroyed after the getter returns

> do not let it dangle

---

# chapter 20 - the filesystem library

## file attributes

- `auto p = std::filesystem::path{...};`
- `exists(p)`
- `is_(regular_file/directory/symlink/other/block_file/character_file/fifo/socket/empty)(p)`
	- follows symlink
- `file_size(p)`
- `hard_link_count(p)`
- `last_write_time(p)`
- `(symlink_)status(p)`: return the (symlink-followed, )cached file type and
  permissions

> due to ADL, there is no need to qualify the calls with `std::filesystem::`,
> because they take an argument that has a type of the same namespace which they
> are from.

> But its better to always qualify because not qualifying the calls might
> sometimes result in UB: `remove()` calls the C function instead of from
> `std::filesystem`

## file permissions

- `file_status.permissions()`: returns an enumeration type
- `std::filesystem::perms::`:

## create/delete files

- `create_(directory/directories)(p)`
- `create_directory(p, attr_path)`
- `create_(hard_link/symlink/directory_symlink)(to, new)`
- `copy(from, to)`
- `copy(from, to, opts)`
- `copy_file(from, to)`
- `copy_file(from, to, opts)`
- `copy_symlink(from, to)`
- `remove(p)`
- `remove_all(p)`

- for more details -> just check out cppref

## example: `switch` over the fs types

```cpp
namespace fs = std::filesystem;
switch (fs::path p{argv[1]}; status(p).type()) {
  case fs::file_type::not_found:
    std::cout << "path \"" << p.string() << "\" does not exist\n";
  break;
  case fs::file_type::regular:
    std::cout << '"' << p.string() << "\" exists with "
              << file_size(p) << " bytes\n";
  break;
  case fs::file_type::directory:
    std::cout << '"' << p.string() << "\" is a directory containing:\n";
    for (const auto& e : std::filesystem::directory_iterator{p}) {
      std::cout << " " << e.path().string() << '\n';
    }
  break;
  default:
    std::cout << '"' << p.string() << "\" is a special file\n";
  break;
}
```

## example: iterating through a directory

```cpp
std::cout << "iterating through " << p << '\n';
for (const auto& e : std::filesystem::directory_iterator{p})
  std::cout << e.path() << '\n';
```

## example: create files of different types

```cpp
namespace fs = std::filesystem;
try {
  fs::path testDir{"tmp/test"};
  create_directories(testDir); // create directories tmp/test/
  auto testFile = testDir / "data.txt"; // create file tmp/test/data.txt
  std::ofstream dataFile{testFile};
  if (!dataFile) {
    std::cerr << "OOPS, can't open \"" << testFile.string() << "\"\n";
    std::exit(EXIT_FAILURE);
  }
  dataFile << "The answer is 42\n";

  // create symbolic link from tmp/slink/ to tmp/test/:
  create_directory_symlink("test", testDir.parent_path() / "slink");
}
catch (const fs::filesystem_error& e) {
  std::cerr << "EXCEPTION: " << e.what() << '\n';
  std::cerr << "path1: \"" << e.path1().string() << "\"\n";
}

std::cout << fs::current_path().string() << ":\n"; // recursively list all files

auto iterOpts{fs::directory_options::follow_directory_symlink};
for (const auto& e : fs::recursive_directory_iterator(".", iterOpts))
  std::cout << " " << e.path().lexically_normal().string() << '\n';
```

- `/`-operator overload paired with the corresponding constructor automatically
  creates/opens a file

## error handling

```cpp
try {
} catch (cont fs::filesystem_error& e) {
  std::println(std::cerr, "{}", e.what());
}
```

## path normalization

- single directory separator
- current directory shorthand `.` not used, unless its the whole path
- file name doesn't contain `..`, unless they are at the beginning of a relative
  path
- path name only ends with a directory separator, if the trailing name is a
  directory which is other than `.` or `..`

The filesystem library provides:

- **lexical** normalization:
	- **not** taking the filesystem into account: member functions, which are
	  thus relative cheap to run, are pure lexical operations
- **filesystem-dependent** normalization
	- free-standing functions, which are filesystem-dependent, are relative
	  expensive because syscalls are involved

## path creation

| init                  | effect                   |
|-----------------------|--------------------------|
| path{charseq}         | from a char sequence     |
| path{begin, end}      | from a range             |
| u8path(u8string)      | from a UTF-8 string      |
| current_path()        | from the cwd             |
| temp_directory_path() | path for temporary files |

- a char sequence:
	- string
	- string_view
	- array of chars / input iterator(pointer) to chars ending with the null
	  terminator

## path inspection/API

- `p.empty()`
- `p.is_absolute()`
- `p.is_relative()`
- `p.begin()`
- `p.end()`
- `p.(has_)filename()`
- `p.(has_)stem()`
- `p.(has_)extension()`
- `p.(has_)root_name()`
- `p.(has_)root_directory()`
- `p.(has_)root_path()`
- `p.(has_)parent_path()`
- `p.(has_)relative_path()`

## path I/O and conversions

- `<<` and `>>` stream operator overload as/from quoted string
- `p.(w/u8/u16/u32)string(<...>)()`
- `p.lexically_normal()`: yields normalized path
- `p.lexically_(relative/proximate)()`: yields path from `p2` to p

## path modifications

- `=` assignment operator overload
- `/`, operator, `p.append(sub)`: appends a *sub(no root)* path with a separator
- `+` operator, `p.concat(str)`: concat a string to the path w/o a separator
- `p.(remove/replace)_filename((repl))`
- `p.(remove/replace)_extension((repl))`
- `p.clear()`: makes the path empty; analog to `.clear()` method from the stl
- `swap()` member and free function
- `p.make_preferred()`: TODO

## path comparison

Best practice with taking the filesystem into account and respecting symlinks:

```cpp
bool pathsAreEqual(const std::filesystem::path& p1,
                   const std::filesystem::path& p2) {
  return exists(p1) && exists(p2)
        ? equivalent(p1, p2)
        : p1.lexically_normal() == p2.lexically_normal();
}
```

- using `equivalent()` requires checking if the paths are valid
- then compare the **normalized** paths lexically
