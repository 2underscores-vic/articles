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
template<class Alloc>
struct std::allocator_traits
{
    static bool resize_allocated(Alloc &a, pointer p, size_type cur_size, size_type new_size);
};
```

It calls `a.resize_allocated(p, cur_size, new_size)` if that expression is
well-formed; otherwise, just returns `false`. Returned `true` means that:

1. The request was satisfied,
2. The memory block length was changed, and
3. It is at least `new_size` bytes length.

The main difference with `realloc()`'s behaivour is that the allocator doesn't
try to move any data, it is caller's responsibility, the allocator just reports
the success status.

## [P0401 - Extensions to the Allocator interface](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0401r0.html) by Jonathan Wakely (Bonus)

Jonathan Wakely proposes different extension in his P0401 paper, sort of
feedback from the allocator - allow allocator to tell the actual size of the
allocated block. It can be combined with the idea from the original proposal:

```C++
template<class Alloc>
struct std::allocator_traits
{
    static bool resize_allocated(Alloc &a, pointer p, size_type cur_size, size_type &new_size);
};
```

Now `new_size` is an input/output parameter. In case of success the allocator
can round up the requested size.

## [N3495 - inplace realloc](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3495.htm)

This paper proposes the similar idea but the function throws `std::bad_alloc()`
when resizing is not supported by allocator. I don't find it practical. From
the uses's point of view it usually worths nothing to now about the support
in principle. The main thing which matters is the result of the resize attempt:
success or not.

## Usage (code)

The sample of usage with vector-like container (including the extension from
P0401) can be found [here](https://github.com/2underscores-vic/articles/blob/master/realloc4cpp/realloc4cpp.cpp).

### Scenario 1: grow

```C++
void push_back(T v)
{
    if(next == buf.end()) // increase capacity first
    {
        if(buf.resize(new_capacity()))
        {
            // AWESOME!!! Buffer was enlarged!
            // No need to move existing elements!
        }
        else // cannot extend, move the buffer as usual
        {
            // ...
        }
    }
    construct(next, std::move(v));
    ++next;
}
```

### Scenario 2: shrink

```C++
void shrink_to_fit()
{
    if(size() == capacity()) return;
    if(buf.resize(size()))
    {
        // AWESOME!!! Buffer was narrowed!
        // No need to move existing elements!
    }
    else // cannot narrow, allocate new buffer
    {
        // ...
    }
}
```

In the both cases `buf.resize()` is a function defined as

```C++
bool resize(size_type desired_capacity)
{
    auto n = new_capacity;
    if(!alloc_traits::resize_allocated(a, begin_, capacity(), n))
        return false;
    assert(new_capacity > capacity() ? n >= new_capacity : true);
    end_ = begin_ + n;
    return true;
}
```

As it can be seen, in each scenario the code

```C++
allocate_new_buffer_and_move_data();
```

just becomes

```C++
if(!buf.resize(new_size()))
    allocate_new_buffer_and_move_data();
```

## Preferred and minimum requested size (Bonus #2)

It was suggested by Fabio Fracassi in
[std-proposals](https://groups.google.com/a/isocpp.org/d/msg/std-proposals/AeL6Q35t1j8/y869WOCRBAAJ)
list to consider adding one feature from
[N2045 - Improving STL Allocators](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2045.html)
proposal.

When we need to add `N` elements to the full vector (`size() == capacity()`),
we usually don't request additional memory for `N` elements. Instead a value
`M` >= `N` is calculated using one of known growth strategies (`new_capacity()`
function in my sample code). `M` is a "preferred size" here. It is possible
that the allocator doesn't have enough memory to expand the buffer for `M`
elements but has for `N`. So it is a reasonble strategy to request both, and
the allocator can try to satisfy at least the "minimal size" request if the
"preferred" one can't be satisfied. Of course, the allocator has the right
to adjust/align the request and allocate slightly more, as before.

The extension can have the following form (all in one):

```C++
template<class Alloc>
struct std::allocator_traits
{
    static bool resize_allocated(Alloc &a, pointer p, size_type cur_size,
        size_type &new_size);
    static bool resize_allocated(Alloc &a, pointer p, size_type cur_size,
        size_type &preferred_size, size_type min_size);
};
```

The first function calls `a.resize_allocated(p, cur_size, new_size)` if
well-formed; otherwise calls `resize_allocated(a, p, cur_size, new_size,
new_size)`.

And in my example for `push_back()` the call

```C++
buf.resize(new_capacity())
```

becomes

```C++
buf.resize(new_capacity(), 1)
```
