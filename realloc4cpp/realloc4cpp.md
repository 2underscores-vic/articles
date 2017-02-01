# `realloc()` for C++

## Abstract

It is a proposal to add a part of `realloc()` function behaivour to C++.

## Introduction

C language defines following subroutines to allocate/deallocate heap memory:

1. `malloc()`/`calloc()`
2. `free()`
3. `realloc()`

C++ provides some counterparts:

1. `malloc()`/`calloc()` - `operator new`/`operator new[]`
2. `free()` - `operator delete/operator delete[]`
3. `realloc()` - ?

As we can see, there is no counterpart for `realloc()`. And there is a solid
reason for that: `realloc()` moves the memory block when it cannot be just
expanded or shortened. It is unacceptable behaviour because objects in C++
are not trivially copyable in general. Therefore `realloc()` cannot be used in
generic code. But! The ability to resize already allocated memory block can
be very useful. We could potentially get both performance and safety gain:
no need to move the existing data (performance) and consequently no risk
to cause an exception by throwing move-operation (safety). Yes, it cannot be
guaranteed that every resize-request will be satisfied (it won't usually in
fact). And what do we do in such case? All we do today! Just fall back to the
current technics.

## Proposal

I propose to extend `std::allocator_traits` with additional function:

```C++
// Extended allocator_traits interface
template<class Alloc>
struct realloc_allocator_traits : public std::allocator_traits<Alloc>
{
    static bool resize_allocated(Alloc &a, pointer p, size_type new_size);
};
```

It calls `a.resize_allocated(p, new_size)` if that expression is well-formed;
otherwise, just returns `false`. Returned `true` means that:

# The request was satisfied,
# The memory block length was changed, and
# It is at least `new_size` bytes length.

The main difference with `realloc()`'s behaivour is that the allocator doesn't
try to move any data, it is caller's responsibility, the allocator just reports
the success status.

## [P0401 - Extensions to the Allocator interface](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0401r0.html) by Jonathan Wakely (Bonus)

Jonathan Wakely proposes different extension in his P0401 paper, sort of
feedback from the allocator - allow allocator to tell the actual size of the
allocated block. It can be combined with the idea from the original proposal:

```C++
template<class Alloc>
struct realloc_allocator_traits : public std::allocator_traits<Alloc>
{
    static bool resize_allocated(Alloc &a, pointer p, size_type &new_size);
};
```

Now `new_size` is an input/output parameter. In case of success the allocator
can round up the requested size.

## TODO

- Do we need to pass also the current block size as for `allocator::deallocate()`?

## Usage (code)

The sample of usage with vector-like container (including the extension from
P0401) can be found [here](https://github.com/2underscores-vic/articles/blob/master/realloc4cpp/realloc4cpp.cpp).
