# chapter 21 - type traits

> just checkout cppref page when in need of some type traits

---

# chapter 22 - parallel stl algos

1. include header `<execution>`
2. call the stl algo with an additional argument:
	- `std::execution::seq`:
		- default single thread sequential execution, effectively disables the
		  parallel execution mechanism[
		- for changing the behavior without changing the signature
		- might behave differently than the "normal" algos
	- `std::execution::par`: multiple threads sequentially execute the
	  operations for the elements
	- `std::execution::par_unseq`: executes operation for multiple elements
	  without ordering guarantee

```cpp
#include <algorithm>
#include <execution>

std::for_each(std::execution_par,
              var.begin(), var.end(),
              [](auto& val) { val.sqrt = std::sqrt(val.value); });
```

- might not always be faster: thread overhead
- **all** parallel algos call `std::terminate()` upon uncaught exception

---

# chapter 23 - new stl algos

## `_n` suffix

- `for_each_n()`
- `copy_n()`
- `fill_n()`
- `generate_n()`

> process the first `n` elements

## `std::reduce()`

- as a parallel form of `accumulate()`
- results based on parallel execution might differ compared to `accumulate()`

## `std::transform_reduce()`

- transform each value before "reducing"(accumulating) them

> *(initial_val) reduce_op transform(elem1) reduce_op transform(elem2) ...*

## `std::inclucive_scan()`

- write results to the destination range via the given `begin` iterator

> *init_val op e1, init_val op e1 op e2, init_val op e1 op e2 op e3, ...eN*

## `std::exclusive_scan()`

> *init_val, init_val op e1, init_val op e1 op e2 op e3 ...eN-1*

## `st::transform_inclusive_scan()`, `std::transform_exclusive_scan()`

- transform each value before the operation

---

# chapter 24 - substring and subsequence searchers

- string member function `.find(substr)`
- (parallel) algo `std::search()` with optional pre-computed hash of the
  needle for improving the speed in a large haystack/needle:
	- `std::(default/boyer_moore/boyer_moore_horspool)_searcher{begin(), end()}`

---

# chapter 25 - utility functions and algos

## `std::size()`

- maps to either `T.size()` or to the size of a passed raw array
- doesn't support `std::initializer_list`, `std::forward_list`

## `std::empty()`

- checks emptiness of a container, a raw array or a `std::initializer_list`
- but for raw C arrays it always returns false

## `std::data()`

## `as_const()`

- cast to `const` without explicit casting
- usages:
	- to force a `const` overload
	- capture by const reference `[&var = std::as_const(var)]`

## `std::sample()`

- extracts a random subset (sample) from a given range of values (population)

```cpp
// print 10 randomly selected values of this collection:
std::sample(coll.begin(), coll.end(),
            std::ostream_iterator<std::string>{std::cout, "\n"},
            10,
            std::default_random_engine{});
```

---

# chapter 26 - container and string extensions

## node handles for associative/unordered container

```cpp
std::map<int, std::string> m{{1, "mango"}, {2, "papaya"}, {3, "guava"}};
auto node_handle = m.extract(2);
node_handle.key() = 4;
m.insert(std::move(node_handle));
```

> instead of removing and creating a new entry for updating the value, no
> (de)allocation is used

- also supports merging, and moving between containers

---

# chapter 27 - multi-threading and concurrency

## `std::scoped_lock`

- as a `std::lock_guard` for multiple mutexes

## `std::shared_mutex`

- concurrent reads don't block each other
- only one exclusive lock for an exclusive write access

## `std::atomic<T>::is_always_lock_free`

- corresponds to macros like `ATOMIC_INT_LOCK_FREE`

---

# chapter 28 - other small library features and modifications

## `std::uncaught_exception()` and `std::uncaught_exceptions()`

```cpp
class Request {
  public:
  ...
  ~Request() {
    if (std::uncaught_exception())
      rollback();
    else
      commit();
  }
};

{
  Request r1{...};
  ...
  if (...) throw ...; // let destructor call rollback()
  ...
} // calls commit() in the good case, or rollback() on exception
```

> check if the destructor is called at the end of a normal program flow or in
> the event of an exception

A problem might arise when:

```cpp
try {
  ...
}
catch (...) {
  Request r2{...};
  ...
} // calls rollback() even if without any additional exception in the catch
```

> use `std::uncaught_exceptions()`:

```cpp
class Request {
private:
  ...
  int initialUncaught{std::uncaught_exceptions()};
public:
  ...
  ~Request() {
    if (std::uncaught_exceptions() > initialUncaught)
      rollback();
    else
      commit();
  }
};
```

- it returns how many nested exceptions are currently unhandled

> so the destructor in the catch clause won't call `rollback()` anymore

## shared pointer

### more native support for raw C arrays

`std::shared_ptr<std::string[]> p{new std::string[10]};`

instead of

`std::shared_ptr<std::string> p{new std::string[10],
std::default_delete<std::string[]>()};`

or

`std::shared_ptr<std::string> p{new std::string[10],
[](std::string* p) {
  delete[] p;
}};`
