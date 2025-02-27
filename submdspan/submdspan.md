
---
title: "`Submdspan`"
document: P2630
date: today
audience: LEWG
author:
  - name: Christian Trott 
    email: <crtrott@sandia.gov>
  - name: Damien Lebrun-Grandie 
    email: <lebrungrandt@ornl.gov>
  - name: Mark Hoemmen 
    email: <mhoemmen@nvidia.com>
  - name: Nevin Liber
    email: <nliber@anl.gov>
toc: true
---


# Revision History

## Revision 3: Mailing 2023-03

- Add feature test macro
- fix wording for aggregate type
- make sure sub_map_offset is defined before it is used
- use exposition only _`integral-constant-like`_ concept instead of `integral_constant`

## Revision 2: Mailing 2023-01

- Add discussion choice of customization point mechanism
- rename `strided_index_range` to `strided_slice`
- introduced named struct combining mapping and offset as return type for `submdspan_mapping`
- removed redundant constraints, which are covered by mandates, thus users will get a better error message.
- fixed layout policy type preservation logic - specifically preserve layouts for rank zero mappings, and create layout stride if any of the slice specifiers are `stride_index_range`
- fixed submapping extent calculation for `strided_slice` where the `extent` is smaller than the `stride` (still should give a submapping extent of 1)
- fixed submapping stride calculation for `strided_slice` slice specifiers.
- added precondition to deal with stride 0 `strided_slice`

## Revision 1: Mailing 2022-10

- updated rational for introducing `strided_slice`
- merged `submdspan_mapping` and `submdspan_offset` into a single customization point `submdspan_mapping` returing a `pair` of the new mapping and the offset.
- update wording for argument dependent lookup.

## Initial Version 2022-08 Mailing


# Description


Until one of the last revisions, the `mdspan` paper P0009 contained `submdspan`,
the subspan or "slicing" function
that returns a view of a subset of an existing `mdspan`.
This function was considered critical for the overall functionality of `mdspan`.
However, due to review time constraints it was removed from P0009 in order for
`mdspan` to be included in C++23. 

This paper restores `submdspan`.  It also expands on the original proposal by

* defining customization points so that `submdspan` can work
    with user-defined layout policies, and
* adding the ability to specify slices as compile-time values.

Creating subspans is an integral capability of many, if not all programming languages
with multidimensional arrays.  These include Fortran, Matlab, Python, and Python's NumPy extension.

Subspans are important because they enable code reuse.
For example, the inner loop in a dense matrix-vector product
actually represents a *dot product* -- an inner product of two vectors.
If one already has a function for such an inner product,
then a natural implementation would simply reuse that function.
The LAPACK linear algebra library depends on subspan reuse
for the performance of its one-sided "blocked" matrix factorizations
(Cholesky, LU, and QR).
These factorizations reuse textbook non-blocked algorithms
by calling them on groups of contiguous columns at a time.
This lets LAPACK spend as much time in dense matrix-matrix multiply
(or algorithms with analogous performance) as possible.

The following example demonstrates this code reuse feature of subspans.
Given a rank-3 `mdspan` representing a three-dimensional grid
of regularly spaced points in a rectangular prism,
the function `zero_surface` sets all elements on the surface
of the 3-dimensional shape to zero.
It does so by reusing a function `zero_2d` that takes a rank-2 `mdspan`.

```c++
// Set all elements of a rank-2 mdspan to zero.
template<class T, class E, class L, class A>
void zero_2d(mdspan<T,E,L,A> grid2d) {
  static_assert(grid2d.rank() == 2);
  for(int i = 0; i < grid2d.extent(0); ++i) {
    for(int j = 0; j < grid2d.extent(1); ++j) {
      grid2d[i,j] = 0;
    }
  }
}

template<class T, class E, class L, class A>
void zero_surface(mdspan<T,E,L,A> grid3d) {
  static_assert(grid3d.rank() == 3);
  zero_2d(submdspan(grid3d, 0, full_extent, full_extent));
  zero_2d(submdspan(grid3d, full_extent, 0, full_extent));
  zero_2d(submdspan(grid3d, full_extent, full_extent, 0));
  zero_2d(submdspan(grid3d, grid3d.extent(0)-1, full_extent, full_extent));
  zero_2d(submdspan(grid3d, full_extent, grid3d.extent(1)-1, full_extent));
  zero_2d(submdspan(grid3d, full_extent, full_extent, grid3d.extent(2)-1));
}
```

## Design of `submdspan`

As previously proposed in an earlier revision of P0009, `submdspan` is a free function.
Its first parameter is an `mdspan` `x`,
and the remaining `x.rank()` parameters are slice specifiers,
one for each dimension of `x`.
The slice specifiers describe which elements of the range $[0,$`x.extent(d)`$)$ are part of the
multidimensional index space of the returned `mdspan`.

This leads to the following fundamental signature:

```c++
template<class T, class E, class L, class A,
         class ... SliceArgs)
auto submdspan(mdspan<T,E,L,A> x, SliceArgs ... args);
```

where `E.rank()` must be equal to `sizeof...(SliceArgs)`.


### Slice Specifiers

#### P0009's slice specifiers

In P0009 we originally proposed three kinds of slice specifiers.

* A single integral value.  For each integral slice specifier given to `submdspan`,
    the rank of the resulting `mdspan` is one less than the rank of the input `mdspan`.
    The resulting multidimensional index space contains only elements of the
    original index space, where the particular index matches this slice specifier.
* Anything convertible to a `tuple<mdspan::index_type, mdspan::index_type>`. The resulting multidimensional index space
covers the begin-to-end subrange of elements in the original index space described by the `tuple`'s two values.
* An instance of the tag class `full_extent_t`.
    This includes the full range of indices in that extent in the returned subspan.

#### Strided index range slice specifier

In other languages which provide multi dimensional arrays with slicing, often a third type of slice specifier exists.
This slicing modes takes generally a step or stride parameter in addition to a begin and end value.

For example obtaining a sub-slice starting at the 5th element and taking every 3rd element up to the 12 can be achieved as:

* Matlab: `A(start:step:end)`
* numpy: `A[start:end:step]`
* Fortran: `A[start:end:step]`

However, we propose a slightly different design.
Instead of providing begin, end and a step size, we propose to specify begin, extent and a step size.

This approach has one fundamental advantage: one can combine a runtime start value with a compile time extent,
and thus directly generate a static extent for the sub-mdspan.

We propose an aggregate type `strided_slice` to express this slice specification.

We use a struct with named fields instead of a `tuple`,
in order to avoid confusion with the order of the three values.

##### `extent` vs `step` choice

As stated above, we propose specifiying the extent because it makes taking a submdspan with static extent
at a runtime offset easier.

One situation where this is would be used is in certain linear algebra algorithms,
where one steps through a matrix and obtains submatrices of a specified, compile time known, size.

However, remember the layout and accessor types of a sub mdspan, depend on the input `mdspan` and the slice specifiers.
Thus for a generic `mdspan`, even if one knows the `extents` of the resulting `mdspan` the actual type is somewhat cumbersome to determine.

```c++
// With start:end:step syntax
template<class T, class E, class L, class A>
void foo(mdspan<T,E,L,A> A) {
  using slice_spec_t = pair<int,int>;
  using subm_t = 
    decltype(submdspan(declval<slcie_spec_t>(), declval<slice_spec_t>()));
  using static_subm_t = mdspan<T, extents<typename subm_t::index_type, 4, 4>,
                               typename subm_t::layout_type,
                               typename subm_t::accessor_type>;

  for(int i=0; i<A.extent(0); i)
    for(int j=0; j<A.extent(1); j++) {
      static_subm_t subm = submdspan(A, slice_spec_t(i,i+4), slice_spec_t(j,j+4));
      // ...
  }
}

// With the proposed type:

template<class T, class E, class L, class A>
void foo(mdspan<T,E,L,A> A) {
  using slice_spec_t = 
    strided_slice<int, integral_constant<int,4>, integral_constant<int,1>>;

  for(int i=0; i<A.extent(0); i)
    for(int j=0; j<A.extent(1); j++) {
      auto subm = submdspan(A, slice_spec_t{i}, slice_spec_t{j});
      // ...
  }
}
```

Furthermore, even if one accepts the more complex specification of `subm` it likely that the first variant would require more instructions,
since compile time information about extents is not available during the `submdspan` call.


##### Template argument deduction

We really want template argument deduction to work for `strided_slice`.
Languages like Fortran, Matlab, and Python
have a concise notation for creating the equivalent kind of slice inline.
It would be unfortunate if users had to write

```c++
auto y = submdspan(x, strided_slice<int, int, int>{0, 10, 3});
```

instead of

```c++
auto y = submdspan(x, strided_slice{0, 10, 3});
```

Not having template argument deduction would make it particularly unpleasant
to mix compile-time and run-time values.  For example, to express the offset and extent as compile-time values and the stride as a run-time value without template argument deduction, users would need to write

```c++
auto y = submdspan(x, strided_slice<integral_constant<size_t, 0>, integral_constant<size_t, 10>, 3>{{}, {}, 3});
```

Template argument deduction would permit a consistently value-oriented style.

```c++
auto y = submdspan(x, strided_slice{integral_constant<size_t, 0>{}, integral_constant<size_t, 10>{}, 3});
```

This case of compile-time offset and extent and run-time stride
would permit optimizations like

* preserving the input mdspan's accessor type (e.g., for aligned access) when the offset is zero; and

* ensuring that the `submdspan` result has a static extent.

##### Designated initializers

We would *also* like to permit use of designated initializers with `strided_slice`.
This would let users choose a more verbose, self-documenting style.
It would also avoid any confusion about the order of arguments.

```c++
auto y = submdspan(x, strided_slice{.offset=0, .extent=10, .stride=3});
```

Designated initializers only work for aggregate types.
This has implications for template argument deduction.
Template argument deduction for aggregates is a C++20 feature.
However, few compilers support it currently.
GCC added full support in version 11, MSVC in 19.27, and EDG eccp in 6.3,
according to the [cppreference.com compiler support table](https://en.cppreference.com/w/cpp/compiler_support).
For example, Clang 14.0.0 supports designated initializers,
and *non*-aggregate (class) template argument deduction,
but it does not currently compile either of the following.

```c++
auto a = strided_slice{.offset=0, .extent=10, .stride=3};
auto b = strided_slice<int, int, int>{.offset=0, .extent=10, .stride=3});
```

Implementers may want to make mdspan available
for users of earlier C++ versions.
The result need not comply fully with the specification,
but should be as usable and forward-compatible as possible.
Implementers can back-port `strided_slice`
by adding the following two constructors.

```c++
strided_slice() = default;
strided_slice(OffsetType offset_, ExtentType extent_, StrideType stride_)
  : offset(offset_), extent_(extent), stride(stride_)
{}
```

These constructors make `strided_slice` no longer an aggregate,
but restore (class) template argument deduction.
They also preserve the struct's properties
of being a structural type (usable as a non-type template parameter)
and trivially copyable (for compatibility with other programming languages).

#### Compile-time integral values in slices

We also propose that any integral value
(on its own, in a `tuple`, or in `strided_slice`)
can be specified as an `integral_constant`.
This ensures that the value can be baked into the return *type* of `submdspan`.
For example, layout mappings could be entirely compile time.

Here are some simple examples for rank-1 `mdspan`.

```c++
int* ptr = ...;
int N = ...;
mdspan a(ptr, N);

// subspan of a single element
auto a_sub1 = submdspan(a, 1);
static_assert(decltype(a_sub1)::rank() == 0);
assert(&a_sub1() == &a(1));

// subrange
auto a_sub2 = submdspan(a, tuple{1, 4});
static_assert(decltype(a_sub2)::rank() == 1);
assert(&a_sub2(0) == &a(1));
assert(a_sub2.extent(0) == 3);

// subrange with stride
auto a_sub3 = submdspan(a, strided_slice{1, 7, 2});
static_assert(decltype(a_sub3)::rank() == 1);
assert(&a_sub3(0) == &a(1));
assert(&a_sub3(3) == &a(7));
assert(a_sub3.extent(0) == 4);

// full range
auto a_sub4 = submdspan(a, full_extent);
static_assert(decltype(a_sub4)::rank() == 1);
assert(a_sub4(0) == a(0));
assert(a_sub4.extent(0) == a.extent(0));
```

In multidimensional use cases these specifiers can be mixed and matched.

```c++
int* ptr = ...;
int N0 = ..., N1 = ..., N2 = ..., N3 = ..., N4 = ...;
mdspan a(ptr, N0, N1, N2, N3, N4);

auto a_sub = submdspan(a,full_extent_t(), 3, strided_slice{2,N2-5, 2}, 4, tuple{3, N5-5});

// two integral specifiers so the rank is reduced by 2
static_assert(decltype(a_sub) == 3);
// 1st dimension is taking the whole extent
assert(a_sub.extent(0) == a.extent(0));
// the new 2nd dimension corresponds to the old 3rd dimension
assert(a_sub.extent(1) == (a.extent(2) - 5)/2);
assert(a_sub.stride(1) == a.stride(2)*2);
// the new 3rd dimension corresponds to the old 5th dimension
assert(a_sub.extent(2) == a.extent(4)-8);

assert(&a_sub(1,5,7) == &a(1, 3, 2+5*2, 4, 3+7));
```

### Customization Points

In order to create the new `mdspan` from an existing `mdspan` `src`, we need three things:

* the new mapping `sub_map`,

* the new accessor `sub_acc`, and

* the new data handle `sub_handle`.

Computing the new data handle is done via an *offset* and the original accessor's `offset` function,
while the new accessor is constructed from the old accessor.

That leaves the construction of the new mapping and the calculation of the *offset* handed to the `offset` function.
Both of those operations depend only on the old mapping and the slice specifiers.

In order to support calling `submdspan` on `mdspan` with custom layout policies, we need to introduce two customization
points for computing the mapping and the offset. Both take as input the original mapping, and the slice specifiers.

```c++
template<class Mapping, class ... SliceArgs>
auto submdspan_mapping(const Mapping&, SliceArgs...) { /* ... */ }

template<class Mapping, class ... SliceArgs>
size_t submdspan_offset(const Mapping&, SliceArgs...) { /* ... */ }
```

Since both of these function may require computing similar information,
one should actually fold `submdspan_offset` into `submdspan_mapping` and make
that single customization point return a struct containing both the submapping and offset.


With these components we can sketch out the implementation of `submdspan`.

```c++
template<class T, class E, class L, class A,
         class ... SliceArgs)
auto submdspan(const mdspan<T,E,L,A>& src, SliceArgs ... args) {
  auto sub_map_offset = submdspan_mapping(src.mapping(), args...);
  typename A::offset_policy sub_acc(src.accessor());
  typename A::offset_policy::data_handle_type 
    sub_handle = src.accessor().offset(src.data_handle(), sub_offset);
  return mdspan(sub_handle, sub_map_offset.first, sub_map_offset.second);
}
```

To support custom layouts, `std::submdspan` calls `submdspan_mapping` using argument-dependent lookup.

However, not all layout mappings may support efficient slicing for all possible slice specifier combinations.
Thus, we do *not* propose to add this customization point to the layout policy requirements. 

### Pure ADL vs. CPO vs. tag invoke

In this paper we propose to implement the customization point via pure ADL (argument-dependent lookup), that is, by calling functions with particular reserved names unqualified.
To evaluate whether there would be significant benefits of using customization point objects (CPOs) or a `tag_invoke` approach,
we followed the discussion presented in P2279.

But first we need to discuss the expected use case scenario.
Most users will never call this customization point directly. Instead it is invoked via `std::submdspan`,
which returns an mdspan viewing the subset of elements indicated by the caller's slice specifiers.
The only place where users of C++ would call `submdspan_mapping` directly is when writing functions that behave analogously to `std::submdspan` for other data structures.
For example a user may want to have a shared ownership variant of `mdspan`.
However, in most cases where we have seen higher level data structures with multi-dimensional indexing,
that higher level data structure actually contains an entire `mdspan` (or its predecessor `Kokkos::View` from the Kokkos library).
Getting subsets of the data then delegates to calling `submdspan` instead of directly working on the underlying mapping.

The only implementers of `submdspan_mapping` are developers of a custom layout policy for `mdspan`.
We expect the number of such developers to be multiple orders of magnitude smaller than actual users of `mdspan`.

One important consideration for `submdspan_mapping` is that there is no default implementation.
That is, this customization point is not used to override some default behavior, unlike functions such as `std::swap` that have a default behavior that users might like to override for their types.

In P2279 Pure ADL, CPOs and `tag_invoke` were evaluated on 9 criteria.
Here we will focus on the criteria where ADL, CPOs and `tag_invoke` got different "marks."

#### Explicit Opt In

One drawback for pure ADL is that it is not blazingly obvious that a function is an implementation of a customization point.
Whether some random user function `begin` or `swap` is actually something intended to work with `ranges::begin` or `std::swap`, is impossible to judge at first sight.
`tag_invoke` improves on that situation by making it blazingly obvious since the actual functions one implements is called `tag_invoke` which takes a `std::tag`.
However, in our case the name of the overloaded function is extremely specific: `submdspan_mapping`.
We do not believe that many people will be confused whether such a function, taking a layout policy mapping and slice specifiers as arguments, is intended
as an implementation point for the customization point of `submdspan` or not.


#### Diagnose Incorrect Opt In

CPOs and `tag_invoke` got a "shrug" in P2279, because they may or may not be a bit better checkable than pure ADL.
However, in the primary use of `submdspan_mapping` via call in `submdspan` we can largely check whether the function does
what we expect. Specifically, the return type has to meet a set
of well defined criteria - it needs to be a layout mapping, with `extents` which are defined by `std::submdspan_extents`.
In fact `submdspan` mandates some of that static information matching the expectations.

#### Associated Types

In the case of `submdspan_mapping` there are no associated types per se (in the way we understood that criterion).
For a function like 
```c++
template<class T>
T::iterator begin(T);
```
`T` itself knows the return type. However for `submdspan_mapping` the return type depends not just on the source mapping,
but also all the slice specifier types. Thus the only way to get it is effectively asking the return type of the function,
however that function is implemented. In fact I would argue that:

```c++
decltype(submdspan_mapping(declval(mapping_t), int, int, full_extent_t, int));
```

is more concise than a possible `tag_invoke` scheme here.

```c++
decltype(std::tag_invoke(std::tag<std::submdspan_mapping>,declval(mapping_t, int, int, full_extent_t, int)));
```

#### Easily invoke the customization

This is likely the biggest drawback of the ADL approach. If one indeed just needs the sub-mapping in a generic context,
one has to remember to use ADL. However, there are two mitigating facts:

- it will be very rare that someone needs to call the `submdspan_mapping` directly - at least compared to calling `submdspan`

- the failure mode of calling `std::submdspan` on a custom mapping type, is an error at compile time - not calling a potential faulty default implementation. 


#### Conclusion

All in all this evaluation let us to believe that there are no significant benefits in prefering CPOs or `tag_invoke` over pure ADL for `submdspan_mapping`.
However, considering the expected rareness of someone implementing the customization point, this is not something we consider a large issue.
If the committee disagrees with our evaluation we are happy to use `tag_invoke` instead - which we believe is preferable over a CPO approach. 



### Making sure submdspan behavior meets expectations

The slice specifiers of `submdspan` completely determine two properties of its result:

1. the `extents` of the result, and
2. what elements of the input of `submdspan` are also represented in the result.

Both of these things are independent of the layout mapping policy.

The approach we described above orthogonalizes handling of accessors and mappings.
Thus, we can define both of the above properties via the multidimensional index spaces,
regardless of what it means to "refer to the same element."
(For example, accessors may use proxy references and data handle types other than C++ pointers.
This makes it hard to define what "same element" means.)
That will let us define pre-conditions for `submdspan` which specify the required behavior of any user-provided `submdspan_mapping` function.

One function which can help with that, and additionally is needed to implement `submdspan_mapping` for
the layouts the standard provides, is a function to compute the submdspan's `extents`.
We will propose this function as a public function in the standard:

```c++
template<class IndexType, class ... Extents, class ... SliceArgs>
auto submdspan_extents(const extents<IndexType, Extents...>, SliceArgs ...);
```

The resulting `extents` object must have certain properties for logical correctness.

* The rank of the sub-extents is the rank of the original `extents` minus the number of pure integral arguments in `SliceArgs`.

* The extent of each remaining dimension is well defined by the `SliceArgs`.  It is the original extent if all the `SliceArgs` are `full_extent_t`.

For performance and preservation of compile-time knowledge, we also require the following.

* Any `full_extent_t` argument corresponding to a static extent, preserves that static extent.

* Generate a static extent when possible.  For example, providing a `tuple` of `integral_constant` as a slice specifier ensures that the corresponding extent is static.

# Wording


>  _In <b>[version.syn]</b>, add:_

```c++
#define __cpp_lib_submdspan YYYYMML // also in <mdspan>
```

[1]{.pnum} Adjust the placeholder value as needed so as to denote this proposal's date of adoption.

## Add subsection 22.7.X [mdspan.submdspan] with the following

<b>24.7.� submdspan [mdspan.submdspan]</b>

<b>24.7.�.1 overview [mdspan.submdspan.overview]</b>

[1]{.pnum} The `submdspan` facilities create a new `mdspan` from an existing input `mdspan`.
   The new `mdspan`'s elements refer to a subset of the elements of the input `mdspan`.

```c++
namespace std {
  template<class OffsetType, class LengthType, class StrideType>
    struct strided_slice;

  template<class LayoutMapping>
    struct submdspan_mapping_result;

  template<class IndexType, class... Extents, class... SliceSpecifiers>
    constexpr auto submdspan_extents(const extents<IndexType, Extents...>&,
                                     SliceSpecifiers ...);

  template<class E, class... SliceSpecifiers>
    constexpr auto submdspan_mapping(
      const layout_left::mapping<E>& src, 
      SliceSpecifiers ... slices) -> @_see below_@;

  template<class E, class... SliceSpecifiers>
    constexpr auto submdspan_mapping(
      const layout_right::mapping<E>& src, 
      SliceSpecifiers ... slices) -> @_see below_@;

  template<class E, class... SliceSpecifiers>
    constexpr auto submdspan_mapping(
      const layout_stride::mapping<E>& src, 
      SliceSpecifiers ... slices) -> @_see below_@;

  // [mdspan.submdspan], submdspan creation
  template<class ElementType, class Extents, class LayoutPolicy,
           class AccessorPolicy, class... SliceSpecifiers>
    constexpr auto submdspan(
      const mdspan<ElementType, Extents, LayoutPolicy, AccessorPolicy>& src,
      SliceSpecifiers...slices) -> @_see below_@;

  template<typename T>
  concept @_integral-constant-like_@ =       // @_exposition only_@
         std::is_integral_v<decltype(T::value)>
      && !std::is_same_v<bool, std::remove_const_t<decltype(T::value)>>
      && T() == T::value;
}
```


[2]{.pnum} The `SliceSpecifier` template argument(s)
   and the corresponding value(s) of the arguments of `submdspan` after `src`
   determine the subset of `src` that the `mdspan` returned by `submdspan` views.

[3]{.pnum} Invocations of `submdspan_mapping` shown in subclause ([mdspan.submdspan]) select a function call via overload resolution
on a candidate set that includes the lookup set found by argument dependent lookup ([basic.lookup.argdep]).

[4]{.pnum} For each function defined in subsection [mdspan.submdspan] that takes a parameter pack named `slices` as an argument:

  * [4.1]{.pnum} let `rank` be the number of elements in `slices`;

  * [4.2]{.pnum} let $s_k$ be the $k$-th element of `slices`;
  
  * [4.3]{.pnum} let $S_k$ be the type of $s_k$; and

  * [4.4]{.pnum} let  _`map-rank`_ be an `array<size_t, rank>` such that for each `k` in the range of $[0,$ `rank`$)$, _`map-rank`_`[k]` equals:

    * `dynamic_extent` if `is_convertible_v<`$S_k$`, size_t>` is `true`, or else

    * the number of $S_j$ with $j < k$ such that `is_convertible_v<`$S_j$`, size_t>` is `false`.


<b>24.7.�.2 `strided_slice` [mdspan.submdspan.strided_slice]</b>

[1]{.pnum} `strided_slice` represents a set of `extent` regularly spaced integer indices.
   The indices start at `offset`, and increase by increments of `stride`.

```c++
template<class OffsetType, class ExtentType, class StrideType>
struct strided_slice {
  using offset_type = OffsetType;
  using extent_type = ExtentType;
  using stride_type = StrideType;

  OffsetType offset;
  ExtentType extent;
  StrideType stride;
};
```

[2]{.pnum} `strided_slice` has the data members and special members specified above.
           It has no base classes or members other than those specified.

[3]{.pnum} *Mandates:*

  * `OffsetType`, `ExtentType`, and `StrideType` are signed or unsigned integer types, or satisfy _`integral-constant-like`_.

<i>[Note: </i>
`strided_slice{.offset=1, .extent=10, .stride=3}` indicates the indices 1, 4, 7, and 10.
Indices are selected from the half-open interval [1, 1 + 10).
<i>- end note]</i>


<b>24.7.�.3 `submdspan_mapping_result` [mdspan.submdspan.submdspan_mapping_result]</b>

[1]{.pnum} Specializations of `submdspan_mapping_result` are returned by overloads of `submdspan_mapping`.

```c++
template<class LayoutMapping>
struct submdspan_mapping_result {
  LayoutMapping mapping;
  size_t offset;
};
```

[2]{.pnum} `submdspan_mapping_result` has the data members and special members specified above.
           It has no base classes or members other than those specified.

[3]{.pnum} `LayoutMapping` shall meet the layout mapping requirements.

<b>24.7.�.4 Exposition-only helpers [mdspan.submdspan.helpers]</b>

```c++
template<class T>
struct @_is-strided-slice_@: false_type {};

template<class OffsetType, class ExtentType, class StrideType>
struct @_is-strided-slice_@<strided_slice<OffsetType, ExtentType, StrideType>>: true_type {};
```


```c++
template<class IndexType, class ... SliceSpecifiers>
constexpr IndexType @_first_@_(size_t k, SliceSpecifiers... slices);
```

[1]{.pnum} *Mandates:* `IndexType` is a signed or unsigned integer type.

[2]{.pnum} Let $φ_k$ denote the following value:

   * [2.1]{.pnum} if `is_convertible_v<`$S_k$`, IndexType>` is `true`, then $s_k$;

   * [2.2]{.pnum} otherwise, if `is_convertible_v<`$S_k$`, tuple<IndexType, IndexType>>` is `true`, then `get<0>(` $s_k$ `)`;

   * [2.3]{.pnum} otherwise, if _`is-strided-slice`_`<`$S_k$`>::value` is `true`, then $s_k$`.offset`;

   * [2.4]{.pnum} otherwise, `0`.

[3]{.pnum} *Preconditions:* $φ_k$ is representable as a value of type `IndexType`.

[4]{.pnum} *Returns:* `extents<IndexType>::`_`index-cast`_`(` $φ_k$ `)`.

```c++
template<class Extents, class ... SliceSpecifiers>
constexpr auto @_last_@_(size_t k, const Extents& ext, SliceSpecifiers... slices);
```

[5]{.pnum} *Mandates:* `Extents` is a specialization of `extents`.

[6]{.pnum} Let `index_type` name the type `typename Extents::index_type`.

[7]{.pnum} Let $λ_k$ denote the following value:

   * [7.1]{.pnum} if `is_convertible_v<`$S_k$`, index_type>` is `true`, then $s_k$` + 1`;

   * [7.2]{.pnum} otherwise, if `is_convertible_v<`$S_k$`, tuple<index_type, index_type>>` is `true`, then `get<1>(` $s_k$ `)`;

   * [7.3]{.pnum} otherwise, if _`is-strided-slice`_`<`$S_k$`>::value` is `true`, then $s_k$`.offset + `$s_k$`.extent`;

   * [7.4]{.pnum} otherwise, `ext.extent(k)`.

[8]{.pnum} *Preconditions:* $λ_k$ is representable as a value of type `index_type`.

[9]{.pnum} *Returns:* `Extents::`_`index-cast`_`(` $λ_k$ `)`.

```c++
template<class IndexType, size_t N, class ... SliceSpecifiers>
constexpr array<IndexType, sizeof...(SliceSpecifiers)> @_src-indices_@(const array<IndexType, N>& indices, SliceSpecifiers ... slices);
```

[10]{.pnum} *Mandates:* `IndexType` is a signed or unsigned integer type.

[11]{.pnum} *Returns:* an `array<IndexType, sizeof...(SliceSpecifiers)>` `src_idx` such that `src_idx[k]` equals

  * [11.1]{.pnum} _`first`_`_<IndexType>(k, slices...)` for each `k` where _`map-rank`_`[k]` equals `dynamic_extent`, otherwise

  * [11.2]{.pnum} _`first`_`_<IndexType>(k, slices...) + indices[`_`map-rank`_`[k]]`.


<b>24.7.�.4 `submdspan_extents` function [mdspan.submdspan.extents]</b>

```c++
template<class IndexType, class ... Extents, class ... SliceSpecifiers>
auto submdspan_extents(const extents<IndexType, Extents...>& src_exts, SliceSpecifiers ... slices);
```

[1]{.pnum} *Constraints:*

   * [1.1]{.pnum} `sizeof...(slices)` equals `Extents::rank()`,

[2]{.pnum} *Mandates:* For each rank index `k` of `src_exts.extents()`, *exactly one* of the following is `true`:

   * [2.1]{.pnum} `is_convertible_v<`$S_k$`, IndexType>`,

   * [2.2]{.pnum} `is_convertible_v<`$S_k$`, tuple<IndexType, IndexType>>`,

   * [2.3]{.pnum} `is_convertible_v<`$S_k$`, full_extent_t>`, or

   * [2.4]{.pnum} _`is-strided-slice`_`<`$S_k$`>::value`.

[3]{.pnum} *Preconditions:* For each rank index `k` of `src_exts.extents()`, *all* of the following are `true`:

   * [3.1]{.pnum} if $S_k$ is a specialization of `strided_slice` and $s_k$`.extent == 0` is `false`, $s_k$`.stride > 0` is `true`,
 
   * [3.2]{.pnum} `0 <= `_`first`_`_<IndexType>(k, slices...)`,

   * [3.3]{.pnum} _`first`_`_<IndexType>(k, slices...) <= `_`last_`_`(k, src_exts, slices...)`, and

   * [3.4]{.pnum} _`last_`_`(k, src_exts, slices...) <= src_exts.extent(k)`.

[4]{.pnum} Let `SubExtents` be a specialization of `extents` such that:

  * [4.1]{.pnum} `SubExtents::rank()` equals the number of $k$ such that `is_convertible_v<`$S_k$`, size_t>` is `false`; and

  * [4.2]{.pnum} for all rank index `k` of `Extents` such that _`map-rank`_`[k] != dynamic_extent` is `true`, `SubExtents::static_extent(`_`map-rank`_`[k])` equals:

       * `Extents::static_extent(k)` if `is_convertible_v<`$S_k$`, full_extent_t>` is `true`; otherwise

       * `tuple_element<1, `$S_k$`>() - tuple_element<0, `$S_k$`>()` if $S_k$ is a `tuple` of two types satisfying _`integral-constant-like`_; otherwise

       * `0`, if $S_k$ is a specialization of `strided_slice`, whose `extent_type` satisfies _`integral-constant-like`_, for which `extent_type()` equals zero; otherwise

       *  `1 + (`$S_k$`::extent_type() - 1) / `$S_k$`::stride_type()` if $S_k$ is a specialization of `strided_slice`, whose `extent_type` and `stride_type` satisfy _`integral-constant-like`_; otherwise

       * `dynamic_extent`.

[5]{.pnum} *Returns:* a value of type `SubExtents` `ext` such that for each `k` for which _`map-rank`_`[k] != dynamic_extent` is `true`:

  * [5.1]{.pnum} `ext.extent(`_`map-rank`_`[k])` equals $s_k$`.extent == 0 ? 0 : 1 + (`$s_k$`.extent - 1) / `$s_k$`.stride` if $S_k$ is a specialization of `strided_slice`, otherwise

  * [5.2]{.pnum} `ext.extent(`_`map-rank`_`[k])` equals _`last_`_`(k, src_exts, slices...) - `_`first`_`_<IndexType>(k, slices...)`.

<b>24.7.�.5 Layout specializations of `submdspan_mapping` [mdspan.submdspan.mapping]</b>

```c++
  template<class Extents, class... SliceSpecifiers>
    constexpr auto submdspan_mapping(
      const layout_left::mapping<Extents>& src, 
      SliceSpecifiers ... slices) -> @_see below_@;

  template<class Extents, class... SliceSpecifiers>
    constexpr auto submdspan_mapping(
      const layout_right::mapping<Extents>& src, 
      SliceSpecifiers ... slices) -> @_see below_@;

  template<class Extents, class... SliceSpecifiers>
    constexpr auto submdspan_mapping(
      const layout_stride::mapping<Extents>& src, 
      SliceSpecifiers ... slices) -> @_see below_@;
```

[1]{.pnum} Let `index_type` name the type `typename Extents::index_type`.

[2]{.pnum} *Constraints:*

   * [2.1]{.pnum} `sizeof...(slices)` equals `Extents::rank()`,

[3]{.pnum} *Mandates:* For each rank index `k` of `src.extents()`, *exactly one* of the following is `true`:

   * [3.1]{.pnum} `is_convertible_v<`$S_k$`, index_type>`,

   * [3.2]{.pnum} `is_convertible_v<`$S_k$`, tuple<index_type, index_type>>`,

   * [3.3]{.pnum} `is_convertible_v<`$S_k$`, full_extent_t>`, or

   * [3.4]{.pnum} _`is-strided-slice`_`<`$S_k$`>::value`.

[4]{.pnum} *Preconditions:* For each rank index `k` of `src.extents()`, *all* of the following are `true`:

   * [4.1]{.pnum} if $S_k$ is a specialization of `strided_slice` and $s_k$`.extent == 0` is `false`, $s_k$`.stride > 0` is `true`,  

   * [4.2]{.pnum} `0 <= `_`first`_`_<index_type>(k, slices...)`,

   * [4.3]{.pnum} _`first`_`_<index_type>(r, slices...) <= `_`last_`_`(k, src.extents(), slices...)`, and

   * [4.4]{.pnum} _`last_`_`(k, src.extents(), slices...) <= src.extent(k)`.


[5]{.pnum} Let `sub_ext` be the result of `submdspan_extents(src.extents(), slices...)` and let `SubExtents` be `decltype(sub_ext)`.

[6]{.pnum} Let `sub_strides` be an `array<SubExtents::index_type, SubExtents::rank()` such that for each rank index `k` of `src.extents()` for which _`map-rank`_`[k]` is not `dynamic_extent` `sub_strides[`_`map-rank`_`[k]]` equals:

   * [6.1]{.pnum} `src.stride(k) * `$s_k$`.stride` if _`is-strided-slice`_`<`$S_k$`>::value` is `true`; otherwise

   * [6.2]{.pnum} `src.stride(k)`.

[7]{.pnum} Let `P`  be a parameter pack such that `is_same_v<make_index_sequence<rank()>, index_sequence<P...>>` is `true`.

[8]{.pnum} Let `offset` be a value of type `size_t` equal to `map(`_`first`_`_<index_type>(P, slices...)...)`.

[9]{.pnum} *Returns:*

   * [9.1]{.pnum} `submdspan_mapping_result{src, 0}`, if `Extents::rank()==0` is `true`; otherwise

   * [9.2]{.pnum} `submdspan_mapping_result{layout_left::mapping(sub_ext), offset}`, if

      * `decltype(src)::layout_type` is `layout_left`; and

      * for each `k` in the range $[0,$ `SubExtents::rank()-1`$)$, $S_k$ is `full_extent_t`; and

      * for `k` equal to `SubExtents::rank()-1`, `is_convertible_v<`$S_k$`, tuple<index_type, index_type>> || is_convertible_v<`$S_k$`, full_extent_t>` is `true`; otherwise

   * [9.3]{.pnum} `submdspan_mapping_result{layout_right::mapping(sub_ext), offset}`, if

      *  `decltype(src)::layout_type` is `layout_right`; and

      * for each `k` in the range $[$ `Extents::rank() - SubExtents::rank()+1, Extents.rank()`$)$, $S_k$ is `full_extent_t`; and

      * for `k` equal to `Extents::rank()-SubExtents::rank()`, `is_convertible_v<`$S_k$`, tuple<index_type, index_type>> || is_convertible_v<`$S_k$`, full_extent_t>` is `true`; otherwise

   * [9.4]{.pnum} `submdspan_mapping_result{layout_stride::mapping(sub_ext, sub_strides), offset}`.


<b>24.7.�.6 `submdspan` function [mdspan.submdspan.submdspan]</b>

```c++
// [mdspan.submdspan], submdspan creation
template<class ElementType, class Extents, class LayoutPolicy,
         class AccessorPolicy, class... SliceSpecifiers>
  constexpr auto submdspan(
    const mdspan<ElementType, Extents, LayoutPolicy, AccessorPolicy>& src,
    SliceSpecifiers...slices) -> @_see below_@;
```

[1]{.pnum} Let `index_type` name the type `typename Extents::index_type`.

[2]{.pnum} Let `sub_map_offset` be the result of `submdspan_mapping(src.mapping(), slices...)`.

[3]{.pnum} *Constraints:*

   * [3.1]{.pnum} `sizeof...(slices)` equals `Extents::rank()`,

   * [3.2]{.pnum} `submdspan_mapping(src.mapping(), slices...)` is well formed.

[4]{.pnum} *Mandates:* 

   * [4.1]{.pnum} `is_same_v<decltype(sub_map_offset.extents()), decltype(submdspan_extents(src.mapping(), slices...))>` is `true`, and

   * [4.2]{.pnum} For each rank index `k` of `src.extents()`, *exactly one* of the following is `true`:

     * `is_convertible_v<`$S_k$`, index_type>`,

     * `is_convertible_v<`$S_k$`, tuple<index_type, index_type>>`,

     * `is_convertible_v<`$S_k$`, full_extent_t>`, or

     * _`is-strided-slice`_`<`$S_k$`>::value`.


[5]{.pnum} *Preconditions:* 

   * [5.1]{.pnum} For each rank index `k` of `src.extents()`, *all* of the following are `true`:

      * if $S_k$ is a specialization of `strided_slice` and $s_k$`.extent == 0` is `false`, $s_k$`.stride > 0` is `true`,  

      * `0 <= `_`first`_`_<index_type>(k, slices...)`,

      * _`first`_`_<index_type>(k, slices...) <= `_`last_`_`(k, src.extents(), slices...)`, and

      * _`last_`_`(k, src.extents(), slices...) <= src.extent(k)`;

   * [5.2]{.pnum} `sub_map_offset.mapping.extents() == submdspan_extents(src.mapping(), slices...)` is `true`; and

   * [5.3]{.pnum} for each integer pack `I` which is a multidimensional index in `sub_map_offset.mapping.extents()`,
     `sub_map_offset.mapping(I...) + sub_map_offset.offset == src.mapping()(`_`src-indices`_`(array{I...}, slices ...))` is `true`.

<i>[Note: </i>
Conditions 5.2 and 5.3 make it so that it is not valid to call `submdspan` with layout mappings, for which an overload of `submdspan_mapping`,
does not return a mapping, which maps to the algorithmicly expected indices, given the slice specifiers.
<i>- end note]</i>

[5]{.pnum} *Effects:* Equivalent to

```c++
  auto sub_map_offset = submdspan_mapping(src.mapping(), args...);
  return mdspan(src.accessor().offset(src.data(), sub_map_offset.offset),
                sub_map_offset.mapping,
                AccessorPolicy::offset_policy(src.accessor()));
```

<i>\[Example:</i>

Given a rank-3 `mdspan` `grid3d` representing a three-dimensional grid
of regularly spaced points in a rectangular prism,
the function `zero_surface` sets all elements on the surface
of the 3-dimensional shape to zero.
It does so by reusing a function `zero_2d` that takes a rank-2 `mdspan`.

```c++
// zero out all elements in an mdspan
template<class T, class E, class L, class A>
void zero_2d(mdspan<T,E,L,A> a) {
  static_assert(a.rank() == 2);
  for(int i=0; i<a.extent(0); i++)
    for(int j=0; j<a.extent(1); j++)
      a[i,j] = 0;
}

// zero out just the surface
template<class T, class E, class L, class A>
void zero_surface(mdspan<T,E,L,A> grid3d) {
  static_assert(grid3d.rank() == 3);
  zero_2d(submdspan(a, 0, full_extent, full_extent));
  zero_2d(submdspan(a, full_extent, 0, full_extent));
  zero_2d(submdspan(a, full_extent, full_extent, 0));
  zero_2d(submdspan(a, a.extent(0)-1, full_extent, full_extent));
  zero_2d(submdspan(a, full_extent, a.extent(1)-1, full_extent));
  zero_2d(submdspan(a, full_extent, full_extent, a.extent(2)-1));
}
```
<i>- end example\]</i>
 
<!---
[5]{.pnum} *Remarks:*

   * [5.1]{.pnum} Let `SubExtents` be a specialization of `extents` such that:
  
      * `SubExtents::rank()` equals the number of $k$ such that `is_convertible_v<`$S_k$`, size_t>` is `false`.

      * For all rank index `k` of `Extents` such that _`map-rank`_`[k] != dynamic_extent` is `true` `SubExtents::static_extent(`_`map-rank`_`[k])` equals:

         * `Extents::static_extent(k)` if `is_convertible_v<`$S_k$`, full_extent_t>` is `true`, otherwise 

         * `dynamic_extent`.

   * [5.2]{.pnum} Let `SubMapping` be the `decltype(submdspan_mapping(src.mapping(),slices...))`.

   * [5.3]{.pnum} If `SubMapping::extents_type` is not `SubExtents` the program is illformed. 

   * [5.3]{.pnum} The return type is `mdspan<ElementType, SubExtents, typename SubMapping::layout_type, typename AccessorPolicy::offset_policy>`.



[3]{.pnum} Let `sub` be the return value of `submdspan(src, slices...)`,
let $s_k$ be the $k$-th element of `slices`, and
let $S_k$ be the type of the $k$-th element of `slices`.

[4]{.pnum} Let _`map-rank`_ be an `array<size_t, Extents::rank()>`
such that for each rank index `j` of `Extents` _`map-rank`_`[j]` equals:

   * [4.1]{.pnum} `dynamic_extent` if `is_convertible_v<`$S_j$`, size_t>` is `true`, or else

   * [4.2]{.pnum} the number of $S_k$ with $k < j$ such that `is_convertible_v<`$S_k$`, size_t>` is `false`.

[5]{.pnum} Let _`first`_ and _`last`_ be `array<size_t, Extents::rank()>`.
For each rank index `r` of `src.extents()`,
define the values of _`first`_`[r]` and _`last`_`[r]` as follows:

   * [5.1]{.pnum} if `is_convertible_v<`$S_r$`, size_t>` is `true`,
  then _`first`_`[r]` equals $s_r$, and
  _`last`_`[r]` equals _`first`_`[r]` + 1;

   * [5.2]{.pnum} otherwise, if `is_convertible_v<`$S_r$`, tuple<size_t, size_t>>` is `true`,
  then _`first`_`[r]` equals `get<0>(t)`, and _`last`_`[r]` equals `get<1>(t)`,
  where `t` is the result of converting $s_r$ to `tuple<size_t, size_t>`;

   * [5.3]{.pnum} otherwise, _`first`_`[r]` equals `0`, and _`last`_`[r]` equals `src.extent(r)`.



[9]{.pnum} *Effects:*

   * [9.1]{.pnum} Direct-non-list-initializes `sub.`_`acc_`_ with `src.accessor()`.

   * [9.2]{.pnum} Let `sub_offset` be `apply(src.mapping(), `_`first`_`)` if _`first`_`[r] < src.extent(r)` for all rank index `r` of `src.extents()`, and
                  `src.mapping().required_span_size()` otherwise. 
                  Direct-non-list-initializes `sub.`_`ptr_`_ with `src.accessor().offset(src.data(), sub_offset)`.
                  <i>[Note:</i> The condition protects against applying invalid indices to the source mapping in cases such as `submdspan(a, tuple{a.extent(0), a.extent(0)})`, where
                      _`first`_`[0] ==` _`last`_`[0]` and _`last`_`[0] == src.extent(0)`. <i>- end note]</i> 

[10]{.pnum} *Postconditions:*

   * [10.1]{.pnum} For $0\:\le$ `k` < `Extents::rank()`, if _`map-rank`_`[k] != dynamic_extent` is `true`, then
     `sub.extent(`_`map-rank`_`[k])` equals _`last`_`[k] - `_`first`_`[k]`.
   
   * [10.2]{.pnum} Let `j` be a multidimensional index in `sub.extents()`, let `J` be `array{static_cast<size_t>(j)...}`, let `I` be `array<size_t, decltype(src)::rank()>` such that
     `I[k] == `_`first`_`[k] + (`_`map-rank`_`[k]==dynamic_extent?0:J[`_`map-rank`_`[k]])` is `true`, then `sub[J]` and `src[I]` refer to the same element.

   * [10.3]{.pnum} If `src.is_strided()` is `true`, then `sub.is_strided()` is `true`.

   * [10.4]{.pnum} If `src.is_unique()` is `true`, then `sub.is_unique()` is `true`.

[11]{.pnum} *Remarks:* 

   * [11.1]{.pnum} Let `SubExtents` be a specialization of `extents` such that:
  
      * `SubExtents::rank()` equals the number of $k$ such that `is_convertible_v<`$S_k$`, size_t>` is `false`.

      * For all rank index `k` of `Extents` such that _`map-rank`_`[k] != dynamic_extent` is `true` `SubExtents::static_extent(`_`map-rank`_`[k])` equals:

         * `Extents::static_extent(k)` if `is_convertible_v<`$S_k$`, full_extent_t>` is `true`, otherwise 

         * `dynamic_extent`.

   * [11.2]{.pnum} Let `SubLayout` be a type that meets the requirements of layout mapping policy and:

      * if `LayoutPolicy` is not one of `layout_left`, `layout_right`, `layout_stride`, then `SubLayout` is implementation defined, otherwise
     
      * if `SubExtents::rank()` is `0`, then `SubLayout` is `LayoutPolicy`, otherwise

      * if `LayoutPolicy` is `layout_left`, `is_convertible_v<`$S_k$`, full_extent_t>` is `true` for all
    $k$ in the range $[0,$`SubExtents::rank()-1`$)$, and `is_convertible_v<`$S_k$`, size_t>` is `false` for $k$ equal `SubExtents::rank()-1`, then
    `SubLayout` is `layout_left`, otherwise
      
      * if `LayoutPolicy` is `layout_right`, `is_convertible_v<`$S_k$`, full_extent_t>` is `true` for all
    $k$ in the range $[$`Extents::rank()-SubExtents::rank()+1`$,$ `Extents::rank()`$)$ and `is_convertible_v<`$S_k$`, size_t>` is `false` for $k$ equal `Extents::rank()-SubExtents::rank()`, then
    `SubLayout` is `layout_right`, otherwise
    
      * `SubLayout` is `layout_stride`.

   * [11.3]{.pnum} The return type is `mdspan<ElementType, SubExtents, SubLayout, typename Accesssor::offset_policy>`.
-->

