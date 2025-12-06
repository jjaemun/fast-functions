# fast-functions!

## Overview 

`ffn` implements a non-owning function wrapper that avoids the indirection cost of
traditional type-erasure designs. This is achieved through template specialization,
which allows aggressive compiler optimizations. The immediate consequence is that 
function calls are resolved at compile time, so that codegen is identical to direct 
calls. This devirtualization implies that `ffn` preserves inlining potential suited
for hot loops or frequent callback interrupts where performance is critical.

## Introduction

We begin with a brief account of the simplest usage we have for the construct. 

```c++

#include "ffn.hpp"

int add(int a, int b) {
    return a + b;
}

int main() {

    // Automatic type deduction implicitly resolves to
    // ffn<int(*)(int, int)>.
    ffn f(&add); 

    // Devitualization invokes direct call as 
    // int result = add(2, 3);
    int result = f(2, 3);

    return 0;
}
```

## Supported Callables

`ffn` transparently wraps any standard `C++` callable:

- **Free functions pointers**
- **Member functions** (`static` or non-`static` and both `const` or non-`const`)
- **Lambdas** (captureless and capturing)
- **Functors**

## The Contract

A number of considerations suggest themselves:

### ABI stability and size

`ffn` guarantees that the binary representation of each specialization is equivalent to
its underlying pointer type. Layout is therefore trivially stable across all translation
units of a single compilation. We formally prove this in Section [**Tests**](./tests). However,
for completeness, we briefly address here the same claims it proposes and subsequent practical limitations.

A fundamental property is that two objects with equivalent representation
compare equal. It is easy to realise why this is the case among `ffn` specializations
and their respective internal pointers. Clearly, the expectation is that `ffn` will not
add any trailing padding, and directly inherit the alignment of the underlying
pointer, which is already natural to the platform. Because non-static member functions, capturing lambdas, and functors have compiler
specific ABI's, `ffn` will be **unstable across compilation boundaries** by definition. 

While `ffn` introduces no additional ABI complexity, it cannot be stable beyond
the types it contains. It is intended to be used as header-only library within a single binary.


#### Pointer sizes

It is also worth noting that callable pointer size may vary. On a typical LP64 system:

```
+--------------------------+----------+
| Callable Type            |   size   |
+--------------------------+----------+
| Free function pointers   |  8 bytes |
| Member function pointers | 16 bytes |
| Lambdas                  |  8 bytes |
| Functors                 |  8 bytes |
+--------------------------+----------+
```

`ffn` **will** preserve the native size and storage alignment of these pointers. 

### Lifetime management

The user, hereafter called **you**, is responsible for callable lifetime management. As the primary caretaker, 
you must see to it that the the callable outlives the wrapper. Undefined behaviour 
is guaranteed when making poor decisions such as returning an `ffn` object 
from a callable only defined in a temporary context. For instance,

```c++
auto create_unsafe() {
    // Define local lambda.
    auto foo = [](int a, int b) -> int { return a + b; };
    // ret unsafe.
    return ffn(&foo);
}

int main() {
    auto unsafe = create_unsafe();
    int result = unsafe(2, 3);
}
```

will compile but produce gibberish/crash at runtime. Of course, this is because the internal pointer
address is no longer valid. From the generated `asm`, we might observe 

```asm
create_unsafe():
    push    rbp
    mov     rbp, rsp
    sub     rsp, 32                  ; alloc stack frame.
    
    ; ... construct lambda on stack.
    ; ... ffn stores lambda stack address to internal pointer.
    
    mov     rax, qword ptr [rbp - 8] ; ret ffn instance, unwound stack.
    add     rsp, 32                  ; destroy stack frame.
    pop     rbp
    ret                              ; ret instruction, dangling ptr.
```

### Exception safety

As of now, `ffn` propagates the exception specification of wrapped callables. This 
has been a matter of conflict. From a design perspective, achieving a satisfactory
answer is hard. While an unconditional use of `noexcept` would default to more aggressive optimizations 
by the compiler, it would change callable observable behaviour. So there is a clear 
tradeoff between transparency and efficiency. Considering that the same
can be achieved by properly defining such `noexcept` guarantees 
on the compute side (i.e., instead of the `ffn` interface), the current 
decision to uphold consistency over efficiency is intentional but not necessarily final.

To give a concrete motivation for either choice, we point to the fact that 
performance gains from enforcing `noexcept` are only speculative while the 
semantic cost of breaking transparency is immediate. 
