# str: yet another string library for C language.

[![License: BSD 3 Clause](https://img.shields.io/badge/License-BSD_3--Clause-yellow.svg)](https://opensource.org/licenses/BSD-3-Clause)

## Motivation

Bored with developing the same functionality over and over again, unsatisfied
with existing libraries, so decided to make the right one, once and forever. 🙂

## Installation
Just clone the project and copy (or symlink) the files `str.h` and `str.c` into your project,
but please respect the [license](LICENSE).

## Code Example
```C
str s = str_null;

str_join(&s, str_lit(", "), str_lit("Here"), str_lit("there"), str_lit("and everywhere"));
str_cat(&s, s, str_lit("..."));

assert(str_eq(s, str_lit("Here, there, and everywhere...")));
str_free(s);
```

## User Guide

_**Disclaimer:** This is the good old C language, not Rust, so nothing can be enforced
on the language level, and certain programming discipline is required to make sure
there is no corrupt or leaked memory resulting from using this library._

A string is represented by the type `str` that maintains a pointer to some memory containing
the actual string. Objects of type `str` are small enough (a struct of a `const char*` and a `size_t`)
to be cheap to pass by value. The strings are assumed to be immutable, like those in Java or Go.

All string objects must be initialised. Uninitialised objects are likely to cause
undefined behaviour. Use `str_null` for empty strings.

There are two kinds of `str` objects: those actually owning the memory they point to, and
non-owning references. This property can be queried using `str_is_owner` and `str_is_ref`
functions, otherwise such objects are indistinguishable.

String references are safe to copy and assign to each other, as long as the memory
they refer to is valid. They do not need to be free'd. `str_free` is a no-op for reference
objects. A reference object can be cheaply created from almost anything that looks like
a C string, including string literals.

Owning objects require special care, in particular:
* It is a good idea to have only one owning object per each allocated string, but such a string
can have many references, as long as those references do not outlive the owning object.
Sometimes this rule may be relaxed for code clarity, like in the above example where
the owning object is passed directly to a function, but only if the function does not
store or release the object. When in doubt pass such an object via `str_ref`.
* Direct assignments (like `s2 = s1;`) to owning objects will certainly leak memory, use
`str_assign` function instead. In fact, this function can assign to any string object,
owning or not, so it can be used everywhere, just to avoid any doubt.
* There is no automatic memory management in C, so every owning object must be released at
some point, either directly by using `str_free` function, or indirectly by assignment from
`str_assign` or a similar function.
* An owning object can be passed over to another location by using `str_move` function. The
function resets its source object to an empty string.

### Safety

As already mentioned, C language offers no help in preventing mistakes like direct assignment
to an owning object, but at least some types of mistakes can be eliminated via careful API design.
For example, in this library all functions producing a new owning string do not simply
return the new string, but instead they assign the result through a pointer to the destination
object. Simply returning a new string would provoke people to assign the result directly,
like `s = str_cat(...);`, potentially leaking memory, or to pass the result to another function,
like `str_cat(str_cat(...), ...)`, where the memory allocated by the inner `str_cat` invocation
will certainly be lost. Assigning via a pointer makes these errors less likely.

In this library all strings are assumed to be immutable, but only by means of `const char*`
pointers, so it is actually possible to write to such a string, although the required type
cast to `char*` offers at least some (mostly psychological) protection from modifying a string
by mistake.

Another issue is that it is technically possible to create a reference to a string that is not
null-terminated. The library accepts strings without null-terminators, but every new string
allocated by the library is guaranteed to be null-terminated.

Just to make things more clear, here is the same code as in the example above, but with comments:
```C
// declare a variable and initialise it with an empty string
str s = str_null;

// join the given string literals around the separator (second parameter),
// storing the result in "s" (first parameter)
str_join(&s, str_lit(", "), str_lit("Here"), str_lit("there"), str_lit("and everywhere"));

// create a new string concatenating "s" and a literal; the function does not modify its
// destination object "s" before the result is computed, also freeing the destination
// before the assignment, so it is safe to use "s" as both a parameter and a destination.
// note: we pass a copy of the owning object "s" as the second parameter, and here it is
// safe to do so because this particular function does not store or release its arguments.
str_cat(&s, s, str_lit("..."));

// just check that we have got the expected result
assert(str_eq(s, str_lit("Here, there, and everywhere...")));

// finally, free the memory allocated for the string
str_free(s);
```

## API brief

`typedef struct { ... } str;`<br>
The string object.

`str_null`<br>
Empty string constant.

`size_t str_len(const str s)`<br>
Returns the number of bytes in the string associated with the object.

`const char* str_ptr(const str s)`<br>
Returns a pointer to the first byte of the string associated with the object. The pointer is never NULL.

`const char* str_end(const str s)`<br>
Returns a pointer to the next byte past the end of the string associated with the object.
The pointer is never NULL, but it is not guaranteed to point to any valid byte or location.
For C strings it points to the terminating null character. Useful in places like `for` loops.

`bool str_is_empty(const str s)`<br>
Returns "true" for empty strings.

`bool str_is_owner(const str s)`<br>
Returns "true" if the string object is the owner of the memory it references.

`bool str_is_ref(const str s)`<br>
Returns "true" if the string object does not own the memory it references.

`void str_free(const str s)`<br>
Release the memory referenced by the owning object; no-op for references.

`void str_assign(str* const ps, const str s)`<br>
Assigns the object "s" to the object pointed to by "ps". Any memory owned by the target
object is freed before the assignment.

`str str_move(str* const ps)`<br>
Saves the given object to a temporary, resets the source object to `str_null`, and then
returns the saved object.

`void str_clear(str* const ps)`<br>
Sets the target object to `str_null` after freeing any memory associated with the object.

`int str_cmp(const str s1, const str s2)`<br>
Lexicographically compares the two string objects, with usual semantics.

`bool str_eq(const str s1, const str s2)`<br>
Returns "true" if the two strings match exactly.

`void str_cat_range(str* const dest, const str* const src, const size_t n)`<br>
Concatenates `n` strings from the array starting at address `src`, and assigns the newly
allocated string to `dest`, with `str_assign` semantics.

`str_cat(dest, ...)`<br>
Concatenates a variable list of `str` arguments into `dest`, with `str_assign` semantics.
Implemented as a macro.

`void str_join_range(str* const dest, const str sep, const str* const src, const size_t n)`<br>
Joins around `sep` the `n` strings from the array starting at address `src`, and assigns
the newly allocated string to `dest`, with `str_assign` semantics.

`str_join(dest, sep, ...)`<br>
Joins around `sep` the variable list of `str` arguments, and assigns
the newly allocated string to `dest`, with `str_assign` semantics.
Implemented as a macro.

`void str_join_range_ignore_empty(str* const dest, const str sep, const str* const src, const size_t n)`<br>
Same as `str_join_range`, but ignores empty strings.

`str_join_ignore_empty(dest, sep, ...)`<br>
Same as `str_join`, but ignores empty arguments. Implemented as a macro.

`str_lit(s)`<br>
Constructs `str` reference object from a string literal. Implemented as a macro.

`str_ref(s)`<br>
Constructs string reference object from either a null-terminated C string, or another `str` object.
Implemented as a macro.

`str str_ref_chars(const char* const s, const size_t n)`<br>
Returns a non-owning object referencing the given range of bytes.

`void str_acquire_chars(str* const dest, const char* const s, size_t n)`<br>
Creates an owning object for the specified range of bytes. The range should be safe to pass to
`free(3)` function. Destination is assigned using `str_assign` semantics.

`void str_acquire(str* const dest, const char* const s)`<br>
Creates an owning object from the given C string. The string should be safe to pass to
`free(3)` function. Destination is assigned using `str_assign` semantics.

### Status
The library requires at least a C11 compiler. So far has been tested on Linux Mint 19.3
with `gcc` version 7.5.0.
