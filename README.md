<div align="center">
  checkpipe
  <p>To bring functional programming data pipelines with robust validation to python and mypy</p>
</div>

<p align="center">
  <a href="https://opensource.org/licenses/MIT">
    <img src="https://img.shields.io/badge/License-MIT-brightgreen.svg"
      alt="License: MIT" />
  </a>
  <a href="https://buymeacoffee.com/lan22h">
    <img src="https://img.shields.io/static/v1?label=Buy me a coffee&message=%E2%9D%A4&logo=BuyMeACoffee&link=&color=greygreen"
      alt="Buy me a Coffee" />
  </a>
</p>

<!-- <p align="center">

  <a href="https://github.com/sponsors/jeffreytse">
    <img src="https://img.shields.io/static/v1?label=sponsor&message=%E2%9D%A4&logo=GitHub&link=&color=greygreen"
      alt="Donate (GitHub Sponsor)" />
  </a>

  <a href="https://github.com/jeffreytse/zsh-vi-mode/releases">
    <img src="https://img.shields.io/github/v/release/jeffreytse/zsh-vi-mode?color=brightgreen"
      alt="Release Version" />
  </a>

  <a href="https://liberapay.com/jeffreytse">
    <img src="http://img.shields.io/liberapay/goal/jeffreytse.svg?logo=liberapay"
      alt="Donate (Liberapay)" />
  </a>

  <a href="https://patreon.com/jeffreytse">
    <img src="https://img.shields.io/badge/support-patreon-F96854.svg?style=flat-square"
      alt="Donate (Patreon)" />
  </a>

  <a href="https://ko-fi.com/jeffreytse">
    <img height="20" src="https://www.ko-fi.com/img/githubbutton_sm.svg"
      alt="Donate (Ko-fi)" />
  </a>

</p> -->

<div align="center">
  <h4>
    <a href="#-whycheckpipe">Why checkpipe?</a> |
    <a href="#-usecases">Use Cases</a> |
    <a href="#install">Install</a> |
    <a href="#-todo">Todo</a> |
    <a href="#-sponsorship">Sponsorship</a> |
    <a href="#-credits">Credits</a> |
    <a href="#-contributing">Contributing</a> |
    <a href="#-license">License</a>
  </h4>
</div>

<div align="center">
  <sub>Built with ❤︎ by Mohammed Alzakariya
  <!-- <a href="https://jeffreytse.net">jeffreytse</a> and
  <a href="https://github.com/jeffreytse/zsh-vi-mode/graphs/contributors">contributors </a> -->
</div>
<br>

<!-- <img alt="TTM Demo" src="https://user-images.githubusercontent.com/9413602/105746868-f3734a00-5f7a-11eb-8db5-22fcf50a171b.gif" /> TODO -->

- [Why checkpipe?](#why-checkpipe)
- [Install](#install)
- [Use Cases](#use-cases)
  - [Case 1: Basic filtering and mapping](#case-1-basic-filtering-and-mapping)
  - [Case 2: Direct transformations outside iterators](#case-2-direct-transformations-outside-iterators)
  - [Case 3: Basic validation in dataflows](#case-3-basic-validation-in-dataflows)
  - [Case 4: Flattening of Errors](#case-4-flattening-of-errors)
  - [Case 5: Flattening of Options](#case-5-flattening-of-options)
  - [Case 6: Unpacking tuples](#case-6-unpacking-tuples)
  - [Case 7: Enumeration](#case-7-enumeration)
  - [Case 8: Creating a new Pipe function](#case-8-creating-a-new-pipe-function)
- [Todo](#todo)
- [Sponsorship](#sponsorship)
- [🎉 Credits](#-credits)
- [Contributing](#contributing)
- [License](#license)


## Why checkpipe?

One problem is trying to express python functions in terms of dataflows. Think of a function that progresses in stages like the following:
```
source_input -> filter -> transform -> filter -> sum
```

Dataflows can be more naturally represented with infix notation, with the
preceding stage leading to the following stage through chaining. But in python
we would find ourselves writing
```
sum(filter(transform(filter(source_input))))
```
which is not very handy. Another approach would be creating placeholder variables
to store each stage, but this also introduces unnecessary state. If this state is mutable, it goes against the principle of purity in functional programming and the
function does not clearly denote a mathematical function.

In data analysis and ETL contexts, we may have to build large dataflows so a better approach is necessary.

a major inspiration for this project and a project which solves the above problem is
[pipe by Julien Palard](https://github.com/JulienPalard/Pipe). It allows for infix notation and gives a simple @Pipe decorator to extend this to any functionality the user needs.

This project aims to build on Julien Palard's project, but with new design considerations: 
* Intellisense and mypy friendly: No untyped lambdas, get full autocomplete ability and type checking by mypy.
* Extended built-in support for error-checking which is integrated into the dataflow. This integrates with the [rustedpy/Result](https://github.com/rustedpy/result) which brings Rust-like Result[T, E] into python.

The project aims to make it easier to write pure python functions with robust error-checking and all the benefits of static analysis tools like mypy.

## Install

```
pip install checkpipe
```

## Use Cases

### Case 1: Basic filtering and mapping
```py
import checkpipe as pipe

print(
    [1, 2, 3]
        | pipe.OfIter[int]
        .map(lambda n: n * 2)
        | pipe.OfIter[int]
        .filter(lambda n: n != 4)
        | pipe.OfIter[int]
        .to_list()
)
```
```
[2, 6]
```

The above example takes a source input `[1, 2, 3]` and transforms it by multiplying each value by 2 into, then keeping only results that aren't 4 and finally consuming this lazy iterator chain into a list result.

When using checkpipe, we are relying on specifying the type of the source
in order for our lambdas to be typed. `[1, 2, 3]` is a List[int] and also can be iterated through as an Iterable[int]. Working with this type of source, we
use `pipe.OfIter[int]`. This makes use of generics to give us expectations on
the signature of the higher order functions passed to functions like `.map` and `.filter`. These expectations can be automatically checked by mypy. And vscode is able to know that `n` is an integer in the lambdas.

### Case 2: Direct transformations outside iterators
```py
import checkpipe as pipe

print(
    3
        | pipe.Of[int]
        .to(lambda n: n+1)
)
```
```
4
```

checkpipe does not only work with iterators. It works directly with types and
allows transformations to the source object as well. In this case, no consumption
of an iterator is jnecessary. `.to(...)` will return the transformed source
directly.

### Case 3: Basic validation in dataflows
```py
import checkpipe as pipe
from result import Result

print(
    [1, 2, 3]
        | pipe.OfIter[int]
        .map(lambda n: n * 2)
        | pipe.OfIter[int]
        .check(lambda n: n != 4)
        | pipe.OfIter[Result[int, int]]
        .to_list()
)
```
```
[Ok(2), Err(4), Ok(6)]
```

Here, we are able to use `.OfIter[int].check` to apply a tag on all values in the source. `Ok[int]` when they pass the check `n != 4` otherwise `Err[int]`. This allows us to propogate errors and handle errors in the pipeline itself. Note that when we're consuming the iterator pipeline with `.to_list()`, we are referring to a new source `Iterator[Result[int, int]]` to reflect the Ok/Err tagging.

We can now proceed to perform more computations on the `Ok[int]` results only:

```py
import checkpipe as pipe
from result import Result

print(
    [1, 2, 3]
        | pipe.OfIter[int]
        .map(lambda n: n * 2)
        | pipe.OfIter[int]
        .check(lambda n: n != 4)
        | pipe.OfResultIter[int, int]
        .on_ok(lambda n: n + 1)
        | pipe.OfIter[Result[int, int]]
        .to_list()
)
```
```
[Ok(3), Err(4), Ok(7)]
```

Here, `.OfResultIter[int, int]` works with an iterable of Results as a source, and only when it detects an Ok, it performs the computation n+1. So we can see that `Ok(2)` became `Ok(3)` and `Ok(6)` became `Ok(7)`, but `Err(4)` remains untouched.

We can also use a different type for the error:

```py
import checkpipe as pipe
from result import Result

print(
    [1, 2, 3, 4]
        | pipe.OfIter[int]
        .map(lambda n: n + 2)
        | pipe.OfResultIter[int, str]
        .check(
            lambda n: n % 2 != 0,
            lambda n: f'Evens like {n} are not allowd!')
        | pipe.OfIter[Result[int, str]]
        .to_list()
)
```
```
[Ok(3), Err('Evens like 4 are not allowd!'), Ok(5), Err('Evens like 6 are not allowd!')]
```

Here `OfResultIter[int, str]` specifies that errors will be in type str and Ok is in type int. It takes two functions, a predicate to check if the int is okay, and a function that maps from that int to some error message. We can then continue processing on just the `Ok[int]` results with `.on_ok(...)` just like before:

```py
import checkpipe as pipe
from result import Result

print(
    [1, 2, 3, 4]
        | pipe.OfIter[int]
        .map(lambda n: n + 2)
        | pipe.OfResultIter[int, str]
        .check(
            lambda n: n % 2 != 0,
            lambda n: f'Evens like {n} are not allowd!')
        | pipe.OfResultIter[int, str]
        .on_ok(lambda n: n * 10)
        | pipe.OfIter[Result[int, str]]
        .to_list()
)
```
```
[Ok(30), Err('Evens like 4 are not allowd!'), Ok(50), Err('Evens like 6 are not allowd!')]
```

We can also chain multiple checks in a row, keeping in mind that checks on `Result[T, E]` use the `then_check` variants while checks on `T` use `check`.

```py
import checkpipe as pipe
from result import Result

print(
    [1, 2, 3, 4]
        | pipe.OfIter[int]
        .map(lambda n: n + 2)
        | pipe.OfResultIter[int, str]
        .check(
            lambda n: n % 2 != 0,
            lambda n: f'Evens like {n} are not allowd!')
        | pipe.OfResultIter[int, str]
        .then_check(
            lambda n: n != 3,
            lambda _: 'The number 3 is specifically not welcome!')
        | pipe.OfResultIter[int, str]
        .on_ok(lambda n: n * 10)
        | pipe.OfIter[Result[int, str]]
        .to_list()
)
```
```
[Err('The number 3 is specifically not welcome!'), Err('Evens like 4 are not allowd!'), Ok(50), Err('Evens like 6 are not allowd!')]
```

Sometimes doing a check requires finding a problematic aspect of the source object. For this, we
use the `check_using` functions, which take a finder callback which returns None if it finds
nothing problematic, it just tags the source Ok. But if it does find something problematic, it uses
the problematic object to create an Err object.

```py
import checkpipe as pipe
from result import Result

def find_capitalized_word(s: str) -> Optional[str]:
    words = s.split(' ')

    for word in words:
        if str.isupper(word):
            return word
    
    return None

print(
    [ 
        'this string contains no CAPITALIZED words!',
        'this one is all good!'
    ]
    | pipe.OfResultIter[str, str]
    .check_using(
        find_capitalized_word,
        lambda cap_word: f'Bad! You used a capitalized word: {cap_word}')
    | pipe.OfIter[Result[str, str]]
    .to_list()
)
```
```
[Err('Bad! You used a capitalized word: CAPITALIZED'), Ok('this one is all good!')]
```

### Case 4: Flattening of Errors

Often we might have an error occur during mapping so when we consume we end up with a type like `List[Result[T, E]]`. We can flatten the results by shortcircuiting on the first error, turning it into a `Result[List[T], E]`. Let's say we're interested in checking that a tuple of `(n, m, sub_eq)` satisfy the property that `n - m = sub_eq`:

```py
from result import Result
import checkpipe as pipe
from typing import Tuple

print(
    [(4, 1, 3), (3, 2, 1), (10, 5, 5), (1, 3, 0)]
        | pipe.OfIter[Tuple[int, int, int]]
        .check(pipe.tup3_unpack(lambda n, m, sub_eq:
            n - m == sub_eq
        ))
        | pipe.OfIter[Result[Tuple[int, int, int], Tuple[int, int, int]]]
        .to_list()
)
```
```
[Ok((4, 1, 3)), Ok((3, 2, 1)), Ok((10, 5, 5)), Err((1, 3, 0))]
```

We are able to compute what elements of our iterable satisfy the property, but what if we expect that they all must satisfy it? Then we can flatten the error:

```py
import checkpipe as pipe
from typing import Tuple

print(
    [(4, 1, 3), (3, 2, 1), (10, 5, 5), (1, 3, 0)]
        | pipe.OfIter[Tuple[int, int, int]]
        .check(pipe.tup3_unpack(lambda n, m, sub_eq:
            n - m == sub_eq
        ))
        | pipe.OfResultIter[Tuple[int, int, int], Tuple[int, int, int]]
        .flatten()
)
```
```
Err((1, 3, 0))
```

As expected, we get the error variant, communicating that the entire list did not satisfy the property for all of its elements. Here's what we get if they do satisfy:

```py
import checkpipe as pipe
from typing import Tuple

print(
    [(4, 1, 3), (3, 2, 1), (10, 5, 5), (3, 1, 2)]
        | pipe.OfIter[Tuple[int, int, int]]
        .check(pipe.tup3_unpack(lambda n, m, sub_eq:
            n - m == sub_eq
        ))
        | pipe.OfResultIter[Tuple[int, int, int], Tuple[int, int, int]]
        .flatten()
)
```
```
Ok([(4, 1, 3), (3, 2, 1), (10, 5, 5), (3, 1, 2)])
```

### Case 5: Flattening of Options

Similarly to flattening of Results, sometimes we may map an element in an iterator to 
None, and we want to guarantee our final list consumed has no Nones in it or it is 
entirely None. Here is the example:

```py
import checkpipe as pipe
from typing import Tuple

print(
        [(4, 1, 3), (3, 2, 1), (10, 5, 5), (1, 3, 2)]
            | pipe.OfIter[Tuple[int, int, int]]
            .map(pipe.tup3_unpack(lambda n, m, sub_eq:
                (n, m, sub_eq) if n - m == sub_eq else None
            ))
            | pipe.FlattenOptionals[Tuple[int, int, int]]
            .flatten()
)
```
```
None
```

And the valid example:
```py
import checkpipe as pipe
from typing import Tuple

print(
        [(4, 1, 3), (3, 2, 1), (10, 5, 5), (3, 1, 2)]
            | pipe.OfIter[Tuple[int, int, int]]
            .map(pipe.tup3_unpack(lambda n, m, sub_eq:
                (n, m, sub_eq) if n - m == sub_eq else None
            ))
            | pipe.FlattenOptionals[Tuple[int, int, int]]
            .flatten()
)
```
```
[(4, 1, 3), (3, 2, 1), (10, 5, 5), (3, 1, 2)]
```


### Case 6: Unpacking tuples

checkpipe comes with support for unpacking tuples of limited size while specifying
the types of each element:

```py
import checkpipe as pipe
from typing import Tuple

print(
    (4, 2, 'Hello ')
        | pipe.Of[Tuple[int, int, str]]
        .to(pipe.tup3_unpack(lambda num_underscores, repeat, text: 
            '"' + ('_' * num_underscores) + (repeat * text) + '"'
        ))
)
```
```
"____Hello Hello "
```

You can also use the pipe.tupN_unpack functions within a `pipe.OfIter[T].map` for instance:

```py
import checkpipe as pipe
from typing import Tuple

print(
        [(4, 1, 3), (3, 2, 1), (10, 5, 5), (1, 3, 0)]
            | pipe.OfIter[Tuple[int, int, int]]
            .map(pipe.tup3_unpack(lambda n, m, sub_eq:
                n - m == sub_eq
            ))
            | pipe.OfIter[bool]
            .to_list()
)

```
```
[True, True, True, False]
```

### Case 7: Enumeration

We often want to tag an index alongside our data as we iterate. Here is an example:

```py
import checkpipe as pipe
from typing import Tuple

print(
    ['a', 'b', 'c']
    | pipe.OfIter[str]
    .enumerate()
    | pipe.OfIter[Tuple[int, str]]
    .map(pipe.tup2_unpack(lambda i, c: 
        'X' if i == 1 else c
    ))
    | pipe.OfIter[str]
    .to_list()
)
```
```
['a', 'X', 'c']
```


### Case 8: Creating a new Pipe function

```py
import checkpipe as pipe
from checkpipe import Pipe
from typing import Callable, Iterable

@Pipe
def multiply_by_num(num: int) -> Callable[[Iterable[int]], Iterable[int]]:
    def inner(source: Iterable[int]) -> Iterable[int]:
        return map(lambda n: n * num, source)
    return inner

print(
    [1, 2, 3]
        | multiply_by_num(3)
        | pipe.OfIter[int]
        .to_list()
)
```
```
[3, 6, 9]
```

Here we create a new function that could utilize the pipe operator `|`, `multiply_by_num`. It defines an inner function which takes a source, `Iterable[int]`, and it maps it to another `Iterable[int]` via the builtin map function.

If we want to utilize generics to create a more type-general pipe function, we could use typevars to infer types from the arguments passed into the function. If we want to inform the function about a more generic source type, we can wrap it in a class then inform of it the expected source type through the class like this:

```py
import checkpipe as pipe
from checkpipe import Pipe
from typing import Generic, TypeVar, Callable, Iterable

T = TypeVar('T')

class Repeat(Generic[T]):
    @Pipe
    @staticmethod
    def repeat(n: int) -> Callable[[Iterable[T]], Iterable[T]]:
        def inner(source: Iterable[T]) -> Iterable[T]:
            for item in source:
                for _ in range(n):
                    yield item
        return inner

print(
    ['a', 'b', 'c']
        | Repeat[str]
        .repeat(3)
        | pipe.OfIter[str]
        .to_list()
)
```
```
['a', 'a', 'a', 'b', 'b', 'b', 'c', 'c', 'c']
```

The pipes are type-safe and they can be checked by mypy. checkpipe cannot
automatically infer the source type from the left of the `|`. By specifiying `Repeat[str]`, mypy knows
that when the source `['a', 'b', 'c']` is piped to Repeat, that it must comply to being an `Iterable[str]` or mypy will error.

## Todo
- Implement similar default pipes to Julien Palard's project to facilitate
  transition
- Implement unit testing for all functions of this module

## Sponsorship

If this project brings value to you, please consider supporting me with a monthly sponsorship or [buying me a coffee](https://buymeacoffee.com/lan22h)

## 🎉 Credits

- Thanks to [Julien Palard](https://github.com/JulienPalard/) for the pipe library which was a major inspiration for this project.
- Thanks to [jeffreystse](https://github.com/jeffreytse) for the README style.


## Contributing

All contributions are welcome! I would appreciate feedback on improving the library and optimizing for use cases I haven't thought of yet! Please feel free to contact me by opening an issue ticket or emailing lanhikarixx@gmail.com if you want to chat.

## License

This theme is licensed under the [MIT license](https://opensource.org/licenses/mit-license.php) © Mohammed Alzakariya.