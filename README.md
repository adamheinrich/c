# C cheatsheet

## Data types

### Type sizes

The exact size of integer data types is implementation-specific. Only minimum
_ranges_ are defined (§5.2.4.2.1):

| Type           | Signed range        | Unsigned range          | Size*   |
|----------------|---------------------|-------------------------|---------|
| `char`         | ±127                | 0 to 255                | 8 bits  |
| `int`, `short` | ±32767              | 0 to 65535              | 16 bits |
| `long`         | ±2147483647         | 0 to (2<sup>32</sup>-1) | 32 bits |
| `long long`    | ±(2<sup>63</sup>-1) | 0 to (2<sup>64</sup>-1) | 64 bits |

Important notes:

- Whether a `char` is treated as `signed char` or `unsigned char` is
  up to the implementation (see "Implementation-defined behavior")
- The **size in bits** is not defined by the standard (ditto)
- There is no implicit assumption that negative numbers are represented by
  **two's complement** (which is the most common case). Consequences:
    - Range is `±(2^(N-1) - 1)` rather than `-2^(N-1)` to `2^(N-1) - 1`
    - Undefined behavior for signed overflow

The `int` type usually represents the _natural_ processor word. However, this
is not the case for 8-bit architectures or for some 64-bit systems.

To write portable code, use types defined in `<stdint.h>` (e.g. `uint32_t`)
where size matters.

### The sizeof operator

The `sizeof` operator (not a function) gives object size in "bytes" (§6.5.3.4).
However, because the standard mandates that `sizeof(char)` equals to 1, the
term "byte" is not understood as an octet (8 bits) -- simply because a `char`
can be represented by more than 8 bits. To get size in bits, multiply the result
of the `sizeof` operator by `CHAR_BITS` (defined in `<limits.h>`).

Its result is an unsigned integer of type `size_t` (defined in `<stddef.h>`).
Therefore, the `size_t` type is guaranteed to hold size of any possible object
including arrays. This makes it useful for portable array indexing.

### Integer promotion

The following code prints `c != 0xff` given that it is compiled on a platform
where `char` is treated as signed:

	char c = 0xff;
	if (c == 0xff)
		printf("c == 0xff\n");
	else
		printf("c != 0xff\n");

This is the effect of *integer promotion* (§6.3.1.1):

> If an `int` can represent all values of the original type, the value is
> converted to an `int`; otherwise, it is converted to an `unsigned int`.
> These are called the _integer promotions_. All other types are unchanged by
> the integer promotions.

The value `0xff` (255 in decimal) assigned to `c` is outside the range ±127
defined for `signed char`; given that the machine uses two's complement
representation, the value of `c` is interpreted as -1.

Because all values of `signed char` are representable in `int`, both operands
of the comparison `(c == 0xff)` are promoted to `int`. While the `0xff` literal
is represented by `0x000000ff` on a machine with 32-bit `int`, the negative
value of `c` will be represented as `0xffffffff` after the promotion.
And obviously, `0x000000ff` and `0xffffffff` do not match.

Possible fixes to print `c == 0xff`:

- Declare `c` as `unsigned char` or `uint8_t`. In this case, both operands are
  interpreted as 255 and `0x000000ff` will be compared to `0x000000ff`.
- Cast `0xff` to char in the comparison: `(chat)0xff`. In this case, both
  operands are interpreted as -1 and `0xffffffff` is be compared to
  `0xffffffff`.

### Structures

General rules:

1. The address of the structure is the address of its first member, i.e. the
   first member's offset is always 0
2. Ordering of members is preserved in memory, i.e. a member's offset is greater
   than offset of previously declared members
3. The compiler might add _padding_ between two consecutive members or at the
   end of the structure

#### Designated initializers

It is often desirable to only set a subset of structure members and zero the
others (e.g. for structures with automatic storage which are allocated on
stack or for compatibility with future versions of APIs).

A common approach is to use `memset()` from `<string.h>` to clear the structure
before assigning individual members:

	struct foo {
		int f1;
		int f2;
	} s;

	memset(&s, 0, sizeof(s));
	s.f1 = 1;

However, a better option is to use a _designated initializer_. In this case,
members not assigned by the initializer will be automatically initialized to
zero:

	struct foo s = { .f1 = 1 };

Or, when assigning a new value:

	s = (struct foo) { .f1 = 1 };

## Behavior

The following lists present the most important examples or undefined behavior,
unspecified behavior and implementation-defined behavior. For a complete list,
refer to annex J of the [C standard][c99].

### Undefined behavior

- Using uninitialized variables with automatic storage which
  [never had its address taken][ub-uninit] (§6.3.2.1). If the address has been
  taken, the value is "just" indeterminate
- Using object outside its lifetime (§6.2.4)
- Signed integer overflow
- Buffer overflow (accessing array elements outside bounds)
- Dereferencing `NULL` pointer (§6.3.2)
- Modification of string literals (§6.4.5) and `const` objects (§6.7.3)
- Left shifting past bit-width (e.g. `1UL << 32` for 32-bit int)

Moreover, **shifting value into or past the sign bit** is also undefined
behavior. More precisely, it happens the resulting value is not
[representable in result type][lshift] (because signed integer overflow is
undefined). This is the case when shifting signed positive value by 31 bits on
an architecture with 32-bit int (and two's complement representation of signed
integers):

	int foo = (1 << 31); /* Undefined behavior */

To avoid undefined behavior when left-shifting, use unsigned literals and do not
exceed result type size:

	uint32_t bar = (1UL << 31);

### Implementation-defined behavior

- The number of bits in `char`, defined in `CHAR_BITS` (§3.6)
- Whether `char` is treated as `signed char` or `unsigned char` (§6.2.5)
- Expansion of the `NULL` macro (§3.6) -- it doesn't have to be `((void*)0)`.
  However:
    - Using `if (!ptr)` to check for a null pointer is correct because an
      expression with value 0 cast to `void *` is a _null pointer constant_
      (§6.3.2.3)
    - It is safe to assume that `static char *str` will be initialized
      to a null pointer (§6.7.8)
- Representation of signed integer types (§6.2.6.2)
   - Can be either sign and magnitude, one's complement or two's complement
   - However, `intN_t` types from `<stdint.h>` have to represent integers with
     two's complement representation (§7.18.11.1).
- Right-shifting negative values (§6.5.7)
- Endianness

### Unspecified behavior

- Evaluation order of operands except for `&&`, `||`, `?:` and `,` (§6.5)
    - The exception is handy for constructions like
      `if (str != NULL && *str != '\0') {`
- Evaluation order of function arguments (§6.5.2.2)

## Function prototypes

Prototype `void foo(void)` declares a function which takes no arguments, whereas
prototype `void foo()` declares a function accepting any number of arguments.

Prototype for the main function can be either `int main(int argc, char *argv[])`
or `int main(void)` (§5.1.2.2.1). Moreover, it is not necessary to explicitly
return a value from `main` (§5.1.2.2.3):

> If the return type of the `main` function is a type compatible with `in`t, a
> return from the initial call to the `main` function is equivalent to calling
> the `exit` function with the value returned by the `main` function as its
> argument; reaching the `}` that terminates the `main` function returns a value
> of 0.

Therefore, the following construction is perfectly legal:

	int main(void)
	{
		printf("Hi there\n");
	}

## Useful GCC flags

Apart from `-Wall -pedantic`:

 - `-Wextra`
 - `-Wconversion`
 - `-Wcast-align`
 - `-Wdouble-promotion`
 - `-Wfloat-conversion`
 - [`-ftrapv`][ftrapv] traps signed integer overflow by calling `abort()`

## References

All references in the text refer to the [N1256 draft][c99] of the C99 standard
(ISO/IEC 9899:1999).

Useful links:

- https://gist.github.com/lisovy/b2e8633a53915d7e95c6
- http://www.catb.org/esr/structure-packing/
- https://spin.atomicobject.com/2014/05/19/c-undefined-behaviors/
- http://david.tribble.com/text/cdiffs.htm

[c99]: http://www.open-std.org/jtc1/sc22/WG14/www/docs/n1256.pdf
[lshift]: https://stackoverflow.com/questions/3784996/why-does-left-shift-operation-invoke-undefined-behaviour-when-the-left-side-oper
[ub-uninit]: https://stackoverflow.com/a/11965368

[ftrapv]: https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html#index-ftrapv