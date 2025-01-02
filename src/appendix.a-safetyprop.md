# Appendix A: Privimitive Safety Properties for Rust Contract Design (Draft)

This document presents a draft outlining the fundamental safety properties essential for contract definition. The current documentation on API safety descriptions in the standard library remains ad hoc. For example, the term 'valid pointer' is frequently used, but the validity of a pointer depends on the context. In practice, a valid pointer may need to satisfy one or more fundamental conditions, such as being non-null, non-dangling, and pointing to an object of type T. It is worth noting that the Rust community is making progress toward standardizing contract design, as highlighted in the links below. We believe this proposal will contribute significantly to the development of contract specifications.

[Rust Contracts RFC (draft)](https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2022-11-25-contracts.md)  
[MCP759](https://github.com/rust-lang/compiler-team/issues/759)  
[std-contracts-2025h1](https://rust-lang.github.io/rust-project-goals/2025h1/std-contracts.html)  

## Overall Idea
In contract design, safety properties can be categorized into two types:

**Precondition**: Safety requirements that must be satisfied before invoking an unsafe API. These represent the fundamental conditions for safely using the API.

**Postcondition**: Traditionally, this refers to properties the system must satisfy after the API call. In Rust, it implies that the program must not violate Rust's safety requirements, such as exclusive mutability, after the API is executed. If an API has postconditions, it indicates that satisfying the preconditions alone does not guarantee program safety. Developers must also carefully analyze the API's implementation and its usage context.

While preconditions and postconditions are foundational to safety reasoning, they may not always suffice in Rust. For instance, certain scenarios—such as implementing a doubly linked list or the internals of [Rc](https://doc.rust-lang.org/std/rc/struct.Rc.html) and [RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html) require temporarily violating postconditions. In such cases, it is crucial to document how the program state deviates from Rust's safety principles and whether these vulnerabilities are eventually resolved.

**Hazard (new)**: In some cases, invoking an unsafe API may temporarily leave the program in a vulnerable or inconsistent state. This new category highlights such cases, making it essential to document these hazards clearly.

In practice, a safety property may correspond to a precondition, postcondition, or hazard. To address the ambiguity of certain high-level or ad hoc safety property descriptions, we propose breaking them down into primitive safety requirements. By collecting and analyzing commonly used safety descriptions, we aim to provide a clearer framework for understanding and documenting these properties. The following sections will elaborate on these details.

## Safety Properties
### I. Layout
Refer to the document of [type-layout](https://doc.rust-lang.org/reference/type-layout.html), there are three components related to layout: alignment, size, and padding.

#### Alignment
Alignment is measured in bytes. It must be at least 1 and is always a power of 2. This can be expressed as \\(2^x, s.t. x\ge 0\\). A memory address of type `T` is considered aligned if the address is a multiple of alignment(T). The alignment requirement can be formalized as:

$$ \text{addressof}(\text{instance}(T)) \\% \text{alignment}(T) = 0 $$

In practice, we generally require a pointer `p` of type `T∗` to be aligned. This property can be formalized as:

**psp-1: Aligned(p, T)**:  $$p \\% \text{alignment}(T) = 0$$

Example APIs: [ptr::read()](https://doc.rust-lang.org/nightly/std/ptr/fn.read.html), [ptr::write()](https://doc.rust-lang.org/std/ptr/fn.write.html), [Vec::from_raw_parts()](https://doc.rust-lang.org/beta/std/vec/struct.Vec.html#method.from_raw_parts)

#### b) Size 
The size of a value is the offset in bytes between successive elements in an array with that item type including alignment padding. It is always a multiple of its alignment (including 0), i.e., $\text{sizeof}(T) \\% \text{alignment}(T)=0$. 

A safety property may require the size of a type `T` to be not ZST. We can formulate the requirement as 

**psp-2: NonZST(T)**: $$\text{sizeof}(T) > 0$$

Example API: [NonNull.offset_from](https://doc.rust-lang.org/core/ptr/struct.NonNull.html#method.offset_from), [pointer.sub_ptr](https://doc.rust-lang.org/beta/std/primitive.pointer.html#method.sub_ptr)

#### c) Padding 
Padding refers to the unused space inserted between successive elements in an array to ensure proper alignment. Padding is taken into account when calculating the size of each element. For example, the following data structure includes 1 byte of padding, resulting in a total size of 4 bytes.
```rust
struct MyStruct { a: u16,  b: u8 } // alignment: 2; padding 1
mem::size_of::<MyStruct>(); // size: 4
```

A safety property may require the type `T` has no padding. We can formulate the requirement as 

**psp-3: Padding(T)**: $$\text{padding}(T)=0$$

Example API: intrinsic [raw_eq()](https://doc.rust-lang.org/std/intrinsics/fn.raw_eq.html)

### Pointer Validity

Referring to the [pointer validity](https://doc.rust-lang.org/std/ptr/index.html#safety) documentation, whether a pointer is valid depends on the context of its usage, and the criteria vary across different APIs. To better describe pointer validity and reduce ambiguity, we break down the concept into several primitive components.

#### d) Address
The memory address that the pointer refers to is critical. A safety property may require the pointer `p` to be non-null, as the behavior of a null pointer is undefined. This property can be formalized as:

**psp-4: NonNull(p)**: $$p != null$$

Example APIs: [NonNull::new_unchecked()](https://doc.rust-lang.org/std/ptr/struct.NonNull.html#method.new_unchecked), [Box::from_non_null()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_non_null)

#### e) Allocation
To determine whether the memory address referenced by a pointer is available for use or has been allocated by the system (either on the heap or the stack), we consider the related safety requirement: non-dangling. This means the pointer must refer to a valid memory address that has not been deallocated on the heap or remains valid on the stack.

In practice, an API may enforce that a pointer `p` to a type `T` must satisfy the non-dangling property.

**psp-5: NonDangling(p, T)**: 

$$\text{allocator}(p) = x, s.t., \quad x \in \{ \text{GlobalAllocator}, \text{OtherAllocator}, \text{stack} \} || \text{sizeof}(T) = 0 $$ 

**Proposition 1** (NOT SURE): NonDangling(p, T) implies NonNull(p).

Example API: [ptr::offset()](https://doc.rust-lang.org/beta/std/primitive.pointer.html#method.offset), [Box::from_raw()](https://doc.rust-lang.org/beta/std/boxed/struct.Box.html#method.from_raw)

Besides, some properties may require the allocator to be consistent, i.e., the memory address pointed by the pointer `p` should be allocated by a specific allocator `A`.

**psp-6: AllocaConsistency(p, A)**: $$\text{allocator}(p) = A $$

Example APIs: [Arc::from_raw_in()](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.from_raw_in), [Box::from_raw_in()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw_in)

If the allocator `A` is unspecified, it typically defaults to the global allocator.

**psp-6.1: AllocaConsistency(p)**: $$\text{allocator}(p) = GlobalAllocator $$

Example APIs: [Arc::from_raw()](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.from_raw),[Box::from_raw()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw)

#### f) Pointto

A safety property may require that a pointer `p` refers to a value of a specific type `T`. This property can be formalized as:

**psp-7: Pointto(p, T)**: $$\text{typeof}(*p) = T $$

**Proposition 2** (NOT SURE): Pointto(p, T) implies NonDangling(p, T) and  NonNull(p).

#### Derived Safety Properties
There are two useful derived safety properties based on the previous components.

The first one is bounded access, which requires that the pointer access with respet to an offset stays within the bound. This ensures that dereferencing the pointer results in a value of the expected type T.

**psp-8: Bounded(p, T, offset)**: $$\text{typeof}(*(p + \text{sizeof}(T) * offset))  = T $$

Example APIs: [ptr::offset()](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset), [ptr::copy()](https://doc.rust-lang.org/std/ptr/fn.copy.html) 

A safety property may require the two pointers do not overlap with respect to `T`: 

**psp-9: NonOverlap(\\(p_{dst}\\), \\(p_{src}\\), T)**: $$|p_{dst} - p_{src}| > \text{sizeof}(T)$$

Example APIs: [ptr::copy_from()](https://doc.rust-lang.org/std/ptr/fn.copy.html), [ptr.copy()](https://doc.rust-lang.org/std/ptr/fn.copy_from.html) 

It may also require the two pointers do not overlap with respect to $T\times count$: 

**psp-9.1: NonOverlap(\\(p_{dst}\\), \\(p_{src}\\), T, count)**: $$|p_{dst} - p_{src}| > \text{sizeof}(T) * count $$

Example APIs: [ptr::copy_nonoverlapping()](https://doc.rust-lang.org/std/ptr/fn.copy_nonoverlapping.html), [ptr.copy_from_nonoverlapping](https://doc.rust-lang.org/core/primitive.pointer.html#method.copy_from_nonoverlapping)
 
### Content

#### g) Initialization
A memory of type T pointed by a pointer is either initialized or not. This is a binary primitive property.

$$\text{init}(*p)\in \lbrace true, false \rbrace $$

Example APIs: [MaybeUninit.assume_init()](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#method.assume_init), [Box::assume_init()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.assume_init)

#### h) Integer
Interger arithmatic should not overflow the max or the main value.
**Single Value**
$$ u8::MIN \leq u8(x) \geq u8::MAX $$

Example API: [f32.to_int_unchecked()](https://doc.rust-lang.org/std/primitive.f32.html#method.to_int_unchecked)

Not zero
[NonZero::from_mut_unchecked()](https://doc.rust-lang.org/beta/std/num/struct.NonZero.html#tymethod.from_mut_unchecked)
**Integer Arithmatic**
$$ isize::MAX \leq isize(\text{binop} (x_1, x_2)) \geq isize::MIN $$

$$ usize::MAX \leq usize(\text{binop} (x_1, x_2)) \geq usize::MIN $$

Example APIs: [isize.add()](https://doc.rust-lang.org/std/primitive.isize.html#method.unchecked_add), [usize.add()](https://doc.rust-lang.org/std/primitive.usize.html#method.unchecked_add), [pointer.add(usize.add())](https://doc.rust-lang.org/std/primitive.pointer.html#method.add)

#### i) String
The content must be a valid string. There are two types of string in Rust, [String](https://doc.rust-lang.org/std/string/struct.String.htm) which requires valid utf-8 format, and [CStr](https://doc.rust-lang.org/std/ffi/struct.CStr.html) for interacting with foreign functions.
Example APIs: [String::from_utf8_unchecked()](https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8_unchecked), [CStr::from_ptr()](https://doc.rust-lang.org/std/ffi/struct.CStr.html#method.from_ptr)

#### j) Unwrap

$$\text{enum}(T)\in \lbrace Ok, Err, Some, None \rbrace $$

Example APIs: [Option::unwrap_unchecked()](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap_unchecked), [Result::unwrap_unchecked()](https://doc.rust-lang.org/core/result/enum.Result.html#method.unwrap_unchecked), [Result::unwrap_err_unchecked()](https://doc.rust-lang.org/core/result/enum.Result.html#method.unwrap_err_unchecked)

### Alias-related Primitives
This category relates to the core mechanism of Rust which aims to avoid shared mutable aliases and achieve automated memory deallocation. 

#### k) Onwership
Let one value has two owners at the same program point is vulnerable to double free. Refer to the traidional vulnerbility of [mem::forget()](https://doc.rust-lang.org/std/mem/fn.forget.html) compared to [ManuallyDrop](https://doc.rust-lang.org/std/mem/struct.ManuallyDrop.html). The property generally relates to convert a raw pointer to an ownership, and it can be represented as:

$$\text{owner}(*p) = \lbrace true, false \rbrace $$

Example APIs: [Box::from_raw()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw), [ptr::read()](https://doc.rust-lang.org/std/ptr/fn.read.html), [ptr::read_volatile()](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html)

#### m) Alias
There are six types of alias:

$$\text{pointto}(V) = \bigcup p | p\in \lbrace owner, owner_{mut}, ref, ref_{mut}, ptr, ptr_{mut} \rbrace $$

The exclusive mutability principle of Rust requires that if a value has a mutable alias at one program point, it must not have other aliases at that program point. Otherwise, it may incur unsafe status. We need to track the particular unsafe status and avoid unsafe behaviors. For example, the follow status are vulnerable:

$$ \text{pointto}(V) = owner_{mut} \cup ptr \cup ref $$

Because it violates the exclusive mutability principle requires $owner_{mut}$ and $ref$ should not exist at the same program point.

$$ \text{pointto}(V) = owner_{mut} \cup ptr_{mut} \cup ref_{mut} $$

Because it violates the exclusive mutability principle requires $owner_{mut}$ and $ref_{mut}$ should not exist at the same program point.

Example APIs: [pointer.as_mut()](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut), [pointer.as_ref()](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1), [pointer.as_ref_unchecked()](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref_unchecked-1)

#### l) Lifetime

The property generally requires the lifetime of a raw pointer must be valid for both reads and writes for the whole lifetime 'a.

$$\text{lifetime}(*p)>\'a$$

Example APIs: [AtomicPtr::from_ptr()](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicPtr.html#method.from_ptr), [AtomicBool::from_ptr()](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html#method.from_ptr), [CStr::from_ptr()](https://doc.rust-lang.org/std/ffi/struct.CStr.html#method.from_ptr)

### Advanced Primitive SPs

#### n) Trait
If the parameter has implemented some trait, it is guaranteed to be safe. 

$$\text{trait}(T) = \lbrace Copy, Unpin, Send, Sync \rbrace $$

For example, the [Unpin](https://doc.rust-lang.org/std/marker/trait.Unpin.html) marker trait for implementing [Pin](https://doc.rust-lang.org/std/pin/struct.Pin.html) can ensure safety. However, this is not required. 

Example API: [ptr::read()](https://doc.rust-lang.org/std/ptr/fn.read.html), [ptr::read_volatile()](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html), [Pin::new_unchecked()](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked)

#### o) Atomicity (Thread-Safe)
Refer to the [Rustnomicon](https://doc.rust-lang.org/nomicon/send-and-sync.html), it generally relates to the implementation of the Send/Sync attribute that requires update operations of a critical memory to be exclusive. 

For Send, it requires: 

$$\forall f\in St, \texttt{RefCont}(f) = false$$

For Sync, it requires: 

$$\forall f\in St, \texttt{InterirorMutability}(f) = false$$

Example APIs: Auto trait [Send](https://doc.rust-lang.org/std/marker/trait.Send.html), [Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html)

#### p) Pin
Implementing Pin for !Unpin is also valid in Rust, developers should not move the Pin object pointed by p after created.

$$Pinned(*p) = true$$

Example APIs: [Pin::new_unchecked()](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked),[Pin.into_inner_unchecked()](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.into_inner_unchecked), [Pin.map_unchecked()](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked), [Pin.get_unchecked_mut()](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.get_unchecked_mut), [Pin.map_unchecked_mut](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.map_unchecked_mut)

#### q) I/O

$$\text{owned}(fd) = true$$ (there should be similar issues for other RAII resources, we may not need this because FFI memories cannot require owned.)

$$\text{opened}(fd) = true$$

Example APIs: [trait.FromRawFd::from_raw_fd()](https://doc.rust-lang.org/std/os/fd/trait.FromRawFd.html#tymethod.from_raw_fd), [UdpSocket::from_raw_socket()](https://doc.rust-lang.org/std/net/struct.UdpSocket.html#method.from_raw_socket)

#### r) volatile

There are specific APIs for volatile memory access in std-lib, like [ptr::read_volatile](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html) and [ptr::write_volatile](https://doc.rust-lang.org/std/ptr/fn.write_volatile.html). Other memory operations should require non-volatile by default.

$$volatile(*p) = false$$

Example APIs: [ptr::read()](https://doc.rust-lang.org/std/ptr/fn.read.html), [ptr::write()](https://doc.rust-lang.org/std/ptr/fn.write.html)

