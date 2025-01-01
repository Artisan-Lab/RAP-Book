# Appendix A: Privimitive Safety Properties for Rust Contract Design

This document proposes a draft that defines the basic safety properties useful for contract definition. Note that the Rust community is advancing the standardization of contract design, as referenced in the following links. We believe this proposal would be useful to facilitate contract specifications.

[Rust Contracts RFC (draft)](https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2022-11-25-contracts.md)  
[MCP759](https://github.com/rust-lang/compiler-team/issues/759)  
[std-contracts-2025h1](https://rust-lang.github.io/rust-project-goals/2025h1/std-contracts.html)  

## Overall Idea
In contract design, there are two types of safety properties:

**Precondition**: Safety requirements that must be satisfied before calling an unsafe API.  
**Postcondition**: Traditionally, this refers to properties the system must satisfy after the API call. However, in Rust, it signifies that calling an unsafe API may leave the program in a vulnerable state.  

Sometimes, it can be challenging to classify a safety property as either a precondition or a postcondition. To address this, we further break down safety properties into primitives. Each primitive safety property can serve as either a precondition or a postcondition, depending on the context. The idea also addresses the ambiguity of certain high-level or compound safety properties, such as a ``valid pointer.'' In practice, a valid pointer may need to satisfy several primitive conditions, including being non-null, non-dangling, and pointing to an object of type T. We will elaborate on these details in the sections that follow.

## Safety Properties
### I. Layout-related Primitives
Refer to the document of [type-layout](https://doc.rust-lang.org/reference/type-layout.html), we define three primitives: alignment, size, and padding.

#### a) Alignment
Alignment is measured in bytes. It must be at least 1, and is always a power of 2. It can be represented as $2^x, s.t. x\ge 0$. We say the memory address of a Type T is aligned if the address is a multiple of alignment(T). We can formulate an alignment requirement as:

$$\text{addressof}(\text{instance}(T)) \\% \text{alignment}(T) = 0$$

If requiring a pointer $p$ of type T* to be aligned, the property can be formularized as:

$$p \\% \text{alignment}(T) = 0$$

Example APIs: [ptr::read()](https://doc.rust-lang.org/nightly/std/ptr/fn.read.html), [ptr::write()](https://doc.rust-lang.org/std/ptr/fn.write.html)

#### b) Size 
The size of a value is the offset in bytes between successive elements in an array with that item type including alignment padding. It is always a multiple of its alignment (including 0), i.e., $\text{sizeof}(T) \\% \text{alignment}(T)=0$. 

A safety property may require the size to be not ZST. We can formulate the requirement as 

$$\text{sizeof}(T) > 0$$

Example API: [NonNull.offset_from](https://doc.rust-lang.org/core/ptr/struct.NonNull.html#method.offset_from)

#### c) Padding 
Padding is the unused space required between successive elements in an array, and it will be considered when calculating the size of the element. For example, the following data structure has 1 byte padding, and its size is 4.
```rust
struct MyStruct { a: u16,  b: u8 } // alignment: 2; padding 1
mem::size_of::<MyStruct>(); // size: 4
```

A safety property may require the type T has no padding. We can formulate the requirement as 

$$\text{padding}(T)=0$$

Example API: intrinsic [raw_eq()](https://doc.rust-lang.org/std/intrinsics/fn.raw_eq.html)

### II. Pointer Validity

Refering to the documents about [pointer validity](https://doc.rust-lang.org/std/ptr/index.html#safety), whether a pointer is valid depends on the context of pointer usage, and the criteria varies for different APIs. To better descript the pointer validity and avoid ambiguity, we breakdown the concept related pointer validity into several primitives. 

#### d) Address (Primitive)
The memory address that the pointer points to. A safety property may require the pointer address to be null, namely ``non-null``, because the address of a null pointer is undefined. We can fomulate the property as 

$$ p != null $$

Example APIs: [NonNull::new_unchecked()](https://doc.rust-lang.org/std/ptr/struct.NonNull.html#method.new_unchecked), [Box::from_non_null()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_non_null)

#### e) Allocation (Primitive)
To indicate whether the memory address pointed by the pointer is available to use or has been allocated by the system, either on heap or stack. There is a related safety requirements non-dangling, which means the pointer should point to a valid memory address that has not been deallocated in the heap or is valid in the stack. We can fomulate the requirement as 

$$ \text{alloca}(p) \in \lbrace GlobalAllocator, OtherAllocator, stack \rbrace $$

Example API: [ptr::offset()](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset-1)

Besides, some properties may require the allocator to be consistent, i.e., the memory address pointed by the pointer p should be allocated by a specific allocator, like the GlobalAllocator.

$$ \text{alloca}(p) = GlobalAllocator $$

Example APIs: [Arc::from_raw()](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.from_raw), [Arc::from_raw_in()](https://doc.rust-lang.org/std/sync/struct.Arc.html#method.from_raw_in), [Box::from_raw_in()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw_in)

#### f) Point-to (Primitive)
A safety property may require the pointer point to a value of a particular type. We can fomulate the property as 

$$ \text{typeof}(*p) = T $$

Point-to implies non-dangling and non-null(not sure, need to be confirmed).

#### Derived Safety Properties
There are two useful derived safety properties based on the primitives.

**Bounded Address (derived)**
$$ \text{typeof}(*(p + \text{sizeof}(T) * offset))  = T $$

Example API: [ptr::offset()](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset-1)

**Overlap (derived)**

A safety property may require the two pointers do not overlap with respect to T: 

$$ |p_{dst} - p_{src}| > \text{sizeof}(T)$$

Example API: [ptr::copy()](https://doc.rust-lang.org/std/ptr/fn.copy.html) 

It may also require the two pointers do not overlap with respect to $T\times n$ : 

$$ |p_{dst} - p_{src}| > \text{sizeof}(T) * n $$

Example API: [ptr::copy_nonoverlapping()](https://doc.rust-lang.org/std/ptr/fn.copy_nonoverlapping.html)
 
### Content-related Primitives

#### g) Initialization
A memory of type T pointed by a pointer is either initialized or not. This is a binary primitive property.

$$\text{init}(*p)\in \lbrace true, false \rbrace $$

Example APIs: [MaybeUninit.assume_init()](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html#method.assume_init), [Box::assume_init()](https://doc.rust-lang.org/std/boxed/struct.Box.html#method.assume_init)

#### h) Integer

$$ isize:MAX \leq isize(\text{binop} (x_1, x_2)) \geq isize:MIN $$

$$ usize:MAX \leq usize(\text{binop} (x_1, x_2)) \geq usize:MIN $$

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

Example API: [Pin::new_unchecked()](https://doc.rust-lang.org/std/pin/struct.Pin.html#method.new_unchecked)

#### q) I/O

$$\text{owned}(fd) = true$$ (there should be similar issues for other RAII resources, we may not need this because FFI memories cannot require owned.)

$$\text{opened}(fd) = true$$

Example API: [trait.FromRawFd::from_raw_fd()](https://doc.rust-lang.org/std/os/fd/trait.FromRawFd.html#tymethod.from_raw_fd), [UdpSocket::from_raw_socket()](https://doc.rust-lang.org/std/net/struct.UdpSocket.html#method.from_raw_socket)

#### r) volatile

There are specific APIs for volatile memory access in std-lib, like [ptr::read_volatile](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html) and [ptr::write_volatile](https://doc.rust-lang.org/std/ptr/fn.write_volatile.html). Other memory operations should require non-volatile by default.

$$volatile(*p) = false$$

Example API: [ptr::read()](https://doc.rust-lang.org/std/ptr/fn.read.html), [ptr::write()](https://doc.rust-lang.org/std/ptr/fn.write.html)

