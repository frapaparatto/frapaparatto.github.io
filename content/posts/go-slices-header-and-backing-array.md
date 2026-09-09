---
author: "Francesco Paparatto"
title: "A slice is not the data: understanding Go's slice header"
date: "2026-09-09"
draft: true
description: "Why Go slices behave the way they do, once you see them as a small header pointing at a shared backing array."
tags:
  - go
  - slices
  - arrays
---

Most Go slice bugs, the ones that make you stare at a `fmt.Println` output that makes no sense, come from a single misunderstanding: treating a slice as if it *were* the data. It isn't. A slice is a small header that points at data living somewhere else. Once that distinction is solid, the "surprising" behaviors stop being surprising and start being predictable.

## Arrays first, because slices are built on them

An array in Go is a fixed-size sequence of elements of one type, and the size is part of the type itself: `[3]int` and `[5]int` are not interchangeable, they're different types entirely. Because the size is known at compile time, an array's storage *is* the elements, laid out contiguously, nothing more.

You rarely write array types directly in Go. Their real job is to be the backing storage that slices point into.

## The slice header: three fields, 24 bytes

A slice is not the data. It's a small, fixed-size header, three fields, 24 bytes on a 64-bit machine. The actual runtime definition (`runtime/slice.go`) looks like this:

```go
type slice struct {
	array unsafe.Pointer  // pointer to the backing array
	len   int
	cap   int
}
```

Keep two things separate in your head:

- **slice** = the 24-byte header (pointer, len, cap). Small, cheap to copy.
- **backing array** = the actual element data, living elsewhere in memory, pointed at by the header.

Every "weird" slice behavior you'll ever run into falls out of this split.

## Length is the horizon, capacity is the physical extent

- **capacity** is how many elements physically exist in the backing array.
- **length** is how many of those elements this particular header is allowed to see.

Indexing and `range` are bounded by length, not capacity. Take `make([]int, 3, 8)`: the backing array physically holds 8 ints, but this header can only index `[0..2]`. Index 3 exists in memory and is allocated, but it's invisible through this header, and touching it panics. That gap between len and cap is real, allocated, unreachable memory, until an `append` extends the length into it.

That's the whole job of `append`, really: **it's the operation that turns invisible-but-allocated capacity into visible length.**

## Passing a slice to a function: the header is copied, the backing array isn't

When you pass a slice to a function, Go copies the 24-byte header. But that copied pointer field still holds the same address, so both the caller's header and the function's local header point at the *same* backing array.

Write through the copied header (`s[0] = 99`) and the caller sees it immediately, same array. But `append` inside the function reassigns the *local* header, and the caller never sees that reassignment:

```go
func f(s []int) {
	s = append(s, 99)   // writes 99 into the backing array (if cap allows)
	                     // AND sets the local header to len=4
}
// caller's header is a separate copy, still len=3
```

Physically, 99 lands in the backing array and stays there. Logically, the caller's header still says len=3, so it can't see index 3 — `caller[3]` panics, even though the data is sitting right there. This is why you always write `s = append(s, x)`: `append` returns a new header, and if you drop the return value you're left holding a header that's blind to what `append` just did.

## append is two different operations wearing one name

The line `append(s, x)` does one of two completely different things depending on capacity:

- **Within capacity** (len < cap): writes into the next slot of the *existing* backing array and returns a new header with len+1 and the *same* pointer. No allocation. Dangerous aliasing possible.
- **Exceeds capacity** (len == cap): allocates a *new* backing array, copies everything over, writes the new element, and returns a header pointing at the new, independent array. The old array becomes garbage. Growth roughly doubles capacity (with a smaller factor for very large slices).

Whether a given `append` call aliases or allocates depends entirely on a capacity value you usually can't see from the call site. Same line of code, two behaviors, and that ambiguity is exactly what makes slice bugs nasty.

## The classic aliasing trap

```go
i := make([]int, 3, 8)   // header: ptr → [8 zeros], len=3, cap=8

j := append(i, 4)        // capacity allows it: writes 4 into index 3
                          // j: len=4, same pointer as i

g := append(i, 5)        // i is STILL len=3 (append returned into j, not i)
                          // so this append again sees room at index 3
                          // and writes 5 into the SAME slot, clobbering 4
```

`j` prints `[0 0 0 5]`, not `[0 0 0 4]`, because `j` and `g` share a backing array and `g`'s append overwrote the slot `j` was pointing at. `i` prints `[0 0 0]`, blind to index 3 even though it physically holds 5 now. All three headers share one backing array.

The rule underneath it: append reuses the backing array when capacity allows, and returns a new header. Two slices derived from the same base, with spare capacity, silently corrupt each other on append. Nothing warns you.

If you've written Python or C++, this is worth unlearning explicitly: Go's `append` *returns* a value you must capture, Python's `list.append` mutates in place; Go's `append` sometimes mutates shared memory and sometimes allocates fresh depending on invisible capacity, Python has no such duality; and two Go slices can secretly share storage, two Python lists never do.

## The other trap: a tiny slice can pin a huge array

Re-slicing doesn't copy the backing array, and the garbage collector doesn't track "the part of the array this slice can see" — it tracks the allocation the pointer points into. So a slice's pointer field keeps the *entire* backing array alive, and len/cap are irrelevant to that decision. A 5-byte slice into a 500 MB file holds all 500 MB alive:

```go
func FindDigits(filename string) []byte {
    b, _ := os.ReadFile(filename)
    return digitRegexp.Find(b)   // points into the array holding the WHOLE file
}
```

The fix is to copy the interesting bytes into fresh storage:

```go
c := make([]byte, len(b))
copy(c, b)
return c
```

or the more concise idiom, appending to a nil slice, which forces an allocation because nil has cap 0:

```go
return append([]byte(nil), b...)
```

**Aliasing and retention are different problems with different fixes.** The three-index slice expression `s[low:high:max]` caps capacity so the next `append` is forced to allocate, which fixes aliasing. It does *not* fix retention, because the pointer still points into the original array. Only `copy` or `append([]T(nil), s...)` detaches, because only those allocate new storage. Strings behave the same way for retention (`s[0:5]` on a big string keeps the whole string alive), just without the aliasing-write problem, since strings are immutable.

## Removing an element, and why it's trickier than it looks

Go has no built-in "remove". The classic in-place pattern:

```go
s = append(s[:i], s[i+1:]...)
```

Given `[a b c d e]`, removing index 2: `s[:2]` views `[a b]` and `s[3:]` views `[d e]`. Since there's spare capacity, `append` writes `d` and `e` into indices 2 and 3 of the *same* backing array, an in-place shift-left, zero allocation. The result is `[a b d e]`, but the backing array is now `[a b d e e]` — index 4 still holds a stale duplicate of `e`. Harmless for plain values, but a leak if the element holds a pointer, since that stale slot keeps the removed object alive.

Prefer the standard library, which handles the stale-slot zeroing for you (Go 1.21+):

```go
s = slices.Delete(s, i, i+1)
s = slices.DeleteFunc(s, func(x T) bool {
	return x.Name == target
})
```

And never mutate a slice's length mid-`range` and keep iterating. For a single removal, mutate then immediately `return`/`break`.

## Five facts, everything else follows

1. The header is a copied value, pass-by-value copies the 24 bytes.
2. The backing array is shared through the pointer.
3. Length is the horizon, what you can see.
4. Capacity is the physical extent, what exists.
5. `append` forks on whether capacity is exceeded: reuse-and-alias, or allocate-and-copy.

Every slice surprise in Go reduces to one of these five. Once the header/backing-array split is second nature, the language stops feeling like it's hiding things from you.
