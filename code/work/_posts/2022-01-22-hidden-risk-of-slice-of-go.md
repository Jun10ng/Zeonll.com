---
layout: post
title: Don't use slice as parameters in GO
description: >
  Golang compiler did some optimize for Slice Structure. So, there are some hidden risks of using Slice, especially use Slice as function parameter. 
image: 
  path: /assets/img/blog/example-content-iii.jpg
  srcset:
    1060w: /assets/img/blog/example-content-iii.jpg
    530w:  /assets/img/blog/example-content-iii@0,5x.jpg
    265w:  /assets/img/blog/example-content-iii@0,25x.jpg
sitemap: true
---

## Slice length was reset after appending.

```go
package main

import (
	"fmt"
	"unsafe"
)

func sub(ss []string)  {
	ss[0] = "100"
	ss = append(ss,"3")
    fmt.Println(ss) // highlight-line  // [100 2 3] 
}
func main()  {
	ss := make([]string,0,3)
	ss = append(ss,"1","2")

	sub(ss)
	fmt.Println(ss) // highlight-line  // [100 2] 
}
```

In the above example code, we appended a number into Slice `ss` on `sub` function, it was `[100 2 3]` 
that can be verified at the first highlight line. But after function processes, the `ss` Slice‘s length 
became 2, and its third element was cut off.

The reason is simple. Because when Go retrieves Slice's data, it will get its length at the first time, then cut its underlying array depending on Slice's length. instead of the underlying array. The Slice's length was record at the [SliceHeader](https://pkg.go.dev/reflect#SliceHeader)'s `Len` variable, which is the second member variable of SliceHeader.

```go 
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

In Go Assembly, Slice was split into three variables when passing it into a function: A `uintptr`, and two `int`.
Because `Data` is a reference type variable, any changes about `Data` will take effect(that's why `ss[0]` became `100`),
however, `len` and `cap` are both value type variables, so the changes about them are noneffective.
``` 
"".sub STEXT size=311 args=0x18 locals=0x50 funcid=0x0
             .......
        0x0021 00033 (main.go:9)        MOVQ    "".ss+88(SP), DI. // slice's Data
        0x0026 00038 (main.go:9)        MOVQ    "".ss+96(SP), CX. // slice's Len
```

### Using `unsafe` change its `Len` variables.

In the following example code, we added two lines codes, using some tricks to change the `Len` of SliceHeader.
Then we got an output of `[100 2 3]`, which represents the underlying array we modified in `sub` function is the same as 
we used in `main` function. But `Len` is different.

```go
func main()  {
	ss := make([]string,0,3)
	ss = append(ss,"1","2")

	sub(ss)
	x := uintptr(unsafe.Pointer(&ss)) + uintptr(8) // highlight-line // x stand for pointer of Len of ss
	*(*int)(unsafe.Pointer(x)) +=1 // highlight-line

	fmt.Println(ss) // [100 2 3]
}
```

## Slice's underlying array was changed
If we append a bunch of elements to Slice, Go runtime will build a new bigger underlying array and re-assign
to Slice, in the way, the hidden risk emerges. Now, we have two different underlying arrays in memory. After
`sub` function is finished, the new one would be collected by GC. Especially, any operations about Slice is **NOT**
effective.


## Why Go designed that?
To solve the problem of data over-duplicated problem.

SliceHeader data structure makes variables refer to the same original data in memory.



## How to avoid?

I hope we can have consistent code habits. **Try to avoid using Slice as a parameter**, if we can't avoid it,
remember: **function MUST return the processed Slice, and re-assign to the Slice variable**.
```go
func sub(ss []string) []string {
	ss[0] = "100"
	ss = append(ss,"3")
    fmt.Println(ss) // [100 2 3] 
	return ss // highlight-line 
}
func main()  {
	ss := make([]string,0,3)
	ss = append(ss,"1","2")

	ss = sub(ss) // highlight-line 
	fmt.Println(ss) // [100 2 3]  
}
```



 