# N-Dimensional Arrays


N-dimensional array (a.k.a ndarray) is the building block of Owl library. Ndarray to Owl is like Numpy to Scipy. It serves as the core dense data structure and many advanced numerical functions are built atop of it. For example, `Algodiff`, `Optimise`, `Neural`, and `Lazy` all these functors take Ndarray module as their module input.

Due to its importance, I have implemented a comprehensive set of operations on Ndarray, all of which are defined in [owl_dense_ndarray_generic.mli](https://github.com/owlbarn/owl/blob/master/src/owl/dense/owl_dense_ndarray_generic.mli). Many of these functions (especially the critical ones) in Owl's core library have corresponding C-stub code to guarantee the best performance. If you have a look at the Ndarray's `mli` file, you probably can see hundreds. But do not get scared by the number, many of them are similar and can be grouped together. In this chapter, I will explain these functions in details w.r.t these several groups.


## Ndarray Types

The very first thing to understand is the types used in Ndarray. Owl's Ndarray module is built directly on top of OCaml's native `Bigarray`, more specifically it is `Bigarray.Genarray`. Ndarray has the same type as that of `Genarray`, therefore exchanging data between Owl and other libraries relying on Bigarray is trivial.

OCaml's Bigarray uses `kind` GADT to specify the number type, precision, and memory layout. Owl only keeps the first two but fixes the last one because Owl only uses `C-layout`, or `Row-based layout` in its implementation. The same design decisions can also be seen in ONNX. See the type definition in Ndarray module.

```ocaml
  type ('a, 'b) t = ('a, 'b, c_layout) Genarray.t
```

Technically, `C-layout` indicates the memory address is continuous at the highest dimensions, comparing to the `Fortran-layout` whose continuous memory address is at the lowest dimensions. The reason why I made this decision is as follows.

* Mixing two layouts together opens a can of worms and is the source of bugs. Especially, indexing in Fortran starts from 1 whereas indexing in C starts form 0. Many native OCaml data structures such as `Array` and `List` all start indexing from 0, so using `C-layout` avoids many potential troubles in using the library.

* Supporting both layouts adds a significant amount of complexity in implementing underlying Ndarray functions. Due to the difference in memory layout, code performs well on one layout may not does well on another. Many functions may require different implementations given different layout. This will add too much complexity and increase the code base with marginal benefits.

* Owl has rather different design principles comparing to OCaml's Bigarray. The Bigarray serves as a basic tool to operate on a chunk of memory living outside OCaml's heap, facilitating exchanging data between different libraries (including Fortran ones). Owl focuses on providing high-level numerical functions allowing programmers to write concise analytical code. The simple design and small code base outweighs the benefits of supporting both layouts.

Because of Bigarray, Owl's Ndarray is also subject to maximum 16 dimensions limits. Moreover, Matrix is just a special case of n-dimensional array, and in fact many functions in `Matrix` module simply calls the same functions in Ndarray. But the module does provide more matrix-specific functions such as iterating rows or columns, and etc.



## Creation Functions

The first group of functions I want to introduce is the creation functions. They generate a dense data structure for you to work on further. The most often used ones are probably these four.

```ocaml file=../../examples/code/ndarray/interface_00.mli
open Owl.Dense.Ndarray.Generic

val empty : ('a, 'b) kind -> int array -> ('a, 'b) t

val create : ('a, 'b) kind -> int array -> 'a -> ('a, 'b) t

val zeros : ('a, 'b) kind -> int array -> ('a, 'b) t

val ones : ('a, 'b) kind -> int array -> ('a, 'b) t
```

These four functions return an ndarray of the specified shape, number type, and precision. `empty` function is different from the other three -- it does not really allocate any memory until you access it. Therefore, calling `empty` function is very fast.

The other three functions are self-explained, `zeros` and `ones` fill the allocated memory with zeors and one respectively, whereas `create` function fills the memory with the specified value.

If you need random numbers, you can use another three creation functions that return an ndarray where the elements following certain distributions.

```ocaml file=../../examples/code/ndarray/interface_01.mli
open Owl.Dense.Ndarray.Generic

val uniform : ('a, 'b) kind -> ?a:'a -> ?b:'a -> int array -> ('a, 'b) t

val gaussian : ('a, 'b) kind -> ?mu:'a -> ?sigma:'a -> int array -> ('a, 'b) t

val bernoulli : ('a, 'b) kind -> ?p:float -> int array -> ('a, 'b) t
```

Sometimes, we want to generate numbers with equal distance between two consecutive elements. These ndarrays are useful in generating intervals and plotting figures.

```ocaml file=../../examples/code/ndarray/interface_02.mli
open Owl.Dense.Ndarray.Generic

val sequential : ('a, 'b) kind -> ?a:'a -> ?step:'a -> int array -> ('a, 'b) t

val linspace : ('a, 'b) kind -> 'a -> 'a -> int -> ('a, 'b) t

val logspace : ('a, 'b) kind -> ?base:float -> 'a -> 'a -> int -> ('a, 'b) t
```

If these functions cannot satisfy your need, Ndarray provides a more flexible mechanism allowing you to have more control over the initialisation of an ndarray.

```ocaml file=../../examples/code/ndarray/interface_03.mli
open Owl.Dense.Ndarray.Generic

val init : ('a, 'b) kind -> int array -> (int -> 'a) -> ('a, 'b) t

val init_nd : ('a, 'b) kind -> int array -> (int array -> 'a) -> ('a, 'b) t
```

The difference between the two is: `init` passes 1-d indices to the user-defined function whereas `init_nd` passes n-dimensional indices. As a result, `init` is much faster than `init_nd`. The following code creates an ndarray where all the elements are even numbers.

```ocaml

# let x = Arr.init [|6;8|] (fun i -> 2. *. (float_of_int i))
val x : Arr.arr =

   C0 C1 C2 C3 C4 C5 C6 C7
R0  0  2  4  6  8 10 12 14
R1 16 18 20 22 24 26 28 30
R2 32 34 36 38 40 42 44 46
R3 48 50 52 54 56 58 60 62
R4 64 66 68 70 72 74 76 78
R5 80 82 84 86 88 90 92 94

```



## Properties Functions

After an ndarray is created, you can use various functions in the module to obtain its properties. For example, the following functions are commonly used ones.

```ocaml file=../../examples/code/ndarray/interface_04.mli
open Owl.Dense.Ndarray.Generic

val shape : ('a, 'b) t -> int array
(** [shape x] returns the shape of ndarray [x]. *)

val num_dims : ('a, 'b) t -> int
(** [num_dims x] returns the number of dimensions of ndarray [x]. *)

val nth_dim : ('a, 'b) t -> int -> int
(** [nth_dim x] returns the size of the nth dimension of [x]. *)

val numel : ('a, 'b) t -> int
(** [numel x] returns the number of elements in [x]. *)

val nnz : ('a, 'b) t -> int
(** [nnz x] returns the number of non-zero elements in [x]. *)

val density : ('a, 'b) t -> float
(** [density x] returns the percentage of non-zero elements in [x]. *)

val size_in_bytes : ('a, 'b) t -> int
(** [size_in_bytes x] returns the size of [x] in bytes in memory. *)

val same_shape : ('a, 'b) t -> ('a, 'b) t -> bool
(** [same_shape x y] checks whether [x] and [y] has the same shape or not. *)

val kind : ('a, 'b) t -> ('a, 'b) kind
(** [kind x] returns the type of ndarray [x]. *)
```

Note that `nnz` and `density` need to traverse through all the elements in an ndarray, but because the implementation is in C so even for a very large ndarray the performance is still good.

Property functions are easy to understand. In the following, I want to focus on three typical operations on n-dimensional array worth your special attention - `map`, `fold`, and `scan`.



## Map Functions

`map` function transforms from one ndarray to another with a given function, which is often done by applying the transformation function to every element in the original ndarray. The `map` function in Owl is pure and always generates a fresh new data structure rather than modifying original one.

For example, the following code add 1 to every element in `x`

```ocaml non-deterministic=output
# let x = Arr.uniform [|3;4;5|]
val x : Arr.arr =
  
              C0        C1       C2       C3         C4 
R[0,0]  0.378545  0.861025 0.712662 0.563556   0.964339 
R[0,1]  0.582878  0.834786 0.722758 0.265025   0.712912 
R[0,2] 0.0894476   0.13984 0.475555 0.616536   0.202631 
R[0,3]  0.983487 0.0167333  0.25018 0.483741   0.736418 
R[1,0] 0.0757294  0.662478 0.460645 0.203446   0.725446 
             ...       ...      ...      ...        ... 
R[1,3]   0.83694  0.897979 0.912516 0.833211     0.4145 
R[2,0]  0.903692  0.883623 0.809134 0.859235   0.188514 
R[2,1]  0.236758  0.566636 0.613932 0.215875 0.00911335 
R[2,2]  0.859797  0.708086 0.518328 0.974299   0.472426 
R[2,3]  0.126273  0.946126  0.42223 0.955181   0.422184 


# let y = Arr.map (fun a -> a +. 1.) x
val y : Arr.arr =
  
            C0      C1      C2      C3      C4 
R[0,0] 1.37854 1.86103 1.71266 1.56356 1.96434 
R[0,1] 1.58288 1.83479 1.72276 1.26503 1.71291 
R[0,2] 1.08945 1.13984 1.47556 1.61654 1.20263 
R[0,3] 1.98349 1.01673 1.25018 1.48374 1.73642 
R[1,0] 1.07573 1.66248 1.46065 1.20345 1.72545 
           ...     ...     ...     ...     ... 
R[1,3] 1.83694 1.89798 1.91252 1.83321  1.4145 
R[2,0] 1.90369 1.88362 1.80913 1.85923 1.18851 
R[2,1] 1.23676 1.56664 1.61393 1.21588 1.00911 
R[2,2]  1.8598 1.70809 1.51833  1.9743 1.47243 
R[2,3] 1.12627 1.94613 1.42223 1.95518 1.42218 

```

`map` function can be very useful in implementing vectorised math functions. Many functions in Ndarray can be categorised into this group, such as `sin`, `cos`, `neg`, and etc. Here are some examples to show how to make your own vectorised functions.

```ocaml

  let vec_sin x = Arr.map sin x;;

  let vec_cos x = Arr.map cos x;;

  let vec_log x = Arr.map log x;;

```

If you need indices in the transformation function, you can use `mapi` function which passes in the 1-d index of the element being accessed.

```text

  val mapi : (int -> 'a -> 'a) -> ('a, 'b) t -> ('a, 'b) t

```


## Fold Functions

`fold` function is often referred to as reduction in other programming languages. `fold` function has a named parameter called `axis`, with which you can specify along what axis you want to fold a given ndarray.

```text

  val fold : ?axis:int -> ('a -> 'a -> 'a) -> 'a -> ('a, 'b) t -> ('a, 'b) t

```

The `axis` parameter is optional, if you do not specify one, the ndarray will be flattened first folding happens along the zero dimension. In other words, the all the elements will be folded into a one-element one-dimensional ndarray. The `fold` function in Ndarray is actually folding from left, and you can also specify an initial value of the folding.

The code below demonstrates how to implement your own `sum'` function.

```ocaml

  let sum' ?axis x = Arr.fold ?axis ( +. ) 0. x;;

```

`sum`, `sum'`, `prod`, `prod'`, `min`, `min'`, `mean`, `mean'` all belong to this group. About the difference between the functions with/without prime ending, please refer to the chapter on :doc:`Function Naming Conventions <naming>`.

Similarly, if you need indices in folding function, you can use `foldi` which passes in 1-d indices.

```text

  val foldi : ?axis:int -> (int -> 'a -> 'a -> 'a) -> 'a -> ('a, 'b) t -> ('a, 'b) t

```


## Scan Functions

To some extent, the `scan` function is like the combination of `map` and `fold`. It accumulates the value along the specified axis but it does not change the shape of the input. Think about how we generate a cumulative distribution function (CDF) from a probability density/mass function (PDF/PMF).

The type signature of `scan` looks like this in Ndarray.

```text

  val scan : ?axis:int -> ('a -> 'a -> 'a) -> ('a, 'b) t -> ('a, 'b) t

```

There are several functions belong to this group, such as `cumsum`, `cumprod`, `cummin`, `cummax`, and etc. To implement one `cumsum` for yourself, you can write in the following way.

```ocaml

  let cumsum ?axis x = Arr.scan ?axis ( +. ) x;;

```

Again, you can use `scani` to obtain the indices in the passed in cumulative functions.



## Vectorised Math

Many common operations on ndarrays can be decomposed as a series of `map`, `fold`, and `scan` operations. There is even a specific programming paradigm built atop of this called `Map-Reduce`, which was hyped several years ago in many data processing frameworks. Nowadays, map-reduce is one dominant data-parallel processing paradigm.

The ndarray module has included a very comprehensive set of mathematical functions and all have been vectorised. This means you can apply them directly on an ndarray and the function will be automatically applied to every element in the ndarray.

Conceptually, I can implement all these functions using the aforementioned `map`, `fold`, and `scan`. In reality, these vectorised math is done in C code to guarantee the best performance. Accessing the elements in a bigarray is way faster in C than in OCaml.

For binary math operators, there are `add`, `sub`, `mul`, and etc. For unary operators, there are `sin`, `cos`, `abs`, and etc. You can obtain the complete list of functions in [owl_dense_ndarray_generic.mli](https://github.com/owlbarn/owl/blob/master/src/owl/dense/owl_dense_ndarray_generic.mli>).



## Comparison Functions

The comparison functions themselves can be divided into several groups. The first group compares two ndarrays then returns a boolean value.

```text

  val equal : ('a, 'b) t -> ('a, 'b) t -> bool

  val not_equal : ('a, 'b) t -> ('a, 'b) t -> bool

  val less : ('a, 'b) t -> ('a, 'b) t -> bool

  val greater : ('a, 'b) t -> ('a, 'b) t -> bool

  ...
```


The second group compares two ndarrays but returns an 0-1 ndarray of the same shape. The elements where the predicate is satisfied have value 1 otherwise 0.

```text

  val elt_equal : ('a, 'b) t -> ('a, 'b) t -> ('a, 'b) t

  val elt_not_equal : ('a, 'b) t -> ('a, 'b) t -> ('a, 'b) t

  val elt_less : ('a, 'b) t -> ('a, 'b) t -> ('a, 'b) t

  val elt_greater : ('a, 'b) t -> ('a, 'b) t -> ('a, 'b) t

  ...
```


The third group is similar to the first one but compares an ndarray with a scalar value, the return is a boolean value.

```text

  val equal_scalar : ('a, 'b) t -> 'a -> bool

  val not_equal_scalar : ('a, 'b) t -> 'a -> bool

  val less_scalar : ('a, 'b) t -> 'a -> bool

  val greater_scalar : ('a, 'b) t -> 'a -> bool

  ...
```


The fourth group is similar to the second one but compares an ndarray with a scalar value, the return is an 0-1 ndarray.

```text

  val elt_equal_scalar : ('a, 'b) t -> 'a -> ('a, 'b) t

  val elt_not_equal_scalar : ('a, 'b) t -> 'a -> ('a, 'b) t

  val elt_less_scalar : ('a, 'b) t -> 'a -> ('a, 'b) t

  val elt_greater_scalar : ('a, 'b) t -> 'a -> ('a, 'b) t

  ...
```


You probably noticed the pattern in naming these functions. In general, I recommend using operators rather than calling these function name directly, since it leads to more concise code. Please refer to the chapter on [Conventions](convention.html).


The comparison functions can do a lot of useful things for us. As an example, the following code shows how to keep the elements greater than `0.5` as they are but set the rest to zeros in an ndarray.

```ocaml
let x = Arr.uniform [|10; 10|];;

(* the first solution *)
let y = Arr.map (fun a -> if a > 0.5 then a else 0.) x;;

(* the second solution *)
let z = Arr.((x >.$ 0.5) * x);;
```

As you can see, comparison function combined with operators can lead to more concise code. Moreover, it sometimes outperforms the first solution at the price of higher memory consumption, because the loop is done in C rather than in OCaml.

At this point, you might start understanding why I chose to let comparison functions return 0-1 ndarray as the result.



## Iteration Functions

Like native OCaml array, Owl also provides `iter` and `iteri` functions with which you can iterate over all the elements in an ndarray.

```text

  val iteri :(int -> 'a -> unit) -> ('a, 'b) t -> unit

  val iter : ('a -> unit) -> ('a, 'b) t -> unit

```

One common use case is iterating all the elements and checks if one (or several) predicate is satisfied, there is a special set of iteration functions to help you finish this task.

```text

  val is_zero : ('a, 'b) t -> bool

  val is_positive : ('a, 'b) t -> bool

  val is_negative : ('a, 'b) t -> bool

  val is_nonpositive : ('a, 'b) t -> bool

  val is_nonnegative : ('a, 'b) t -> bool

  val is_normal : ('a, 'b) t -> bool

```

The predicates can be very complicated sometimes. In that case you can use the following three functions to pass in arbitrarily complicated functions to check them.

```text

  val exists : ('a -> bool) -> ('a, 'b) t -> bool

  val not_exists : ('a -> bool) -> ('a, 'b) t -> bool

  val for_all : ('a -> bool) -> ('a, 'b) t -> bool

```

All aforementioned functions only tell us whether the predicates are met or not. They cannot tell which elements satisfy the predicate. The following `filter` function can return the 1-d indices of those elements satisfying the predicates.

```text

  val filteri : (int -> 'a -> bool) -> ('a, 'b) t -> int array

  val filter : ('a -> bool) -> ('a, 'b) t -> int array

```

We have mentioned many times that 1-d indices will be passed in. The reason is passing in 1-d indices is way faster than passing in n-d indices. However, if you do need n-dimensional indices, you can use the following two functions to convert between 1-d and 2-d indices, both are defined in `Owl.Utils` module.

```text

  val ind : ('a, 'b) t -> int -> int array
  (* 1-d to n-d index conversion *)

  val i1d : ('a, 'b) t -> int array -> int
  (* n-d to 1-d index conversion *)

```

Note that you need to pass in the original ndarray because the shape information is required for calculating index conversion.



## Manipulation Functions

Ndarray module contains many useful functions to manipulate ndarrays. For exmaple, you can tile and repeat an ndarray along a specified axis. 

Let's first create a sequential ndarray.

```ocaml

# let x = Arr.sequential [|3;4|]
val x : Arr.arr =

   C0 C1 C2 C3
R0  0  1  2  3
R1  4  5  6  7
R2  8  9 10 11

```

The code below tiles `x` once on both dimentions.

```ocaml
# let y = Arr.tile x [|2;2|]
val y : Arr.arr =

   C0 C1 C2 C3 C4 C5 C6 C7
R0  0  1  2  3  0  1  2  3
R1  4  5  6  7  4  5  6  7
R2  8  9 10 11  8  9 10 11
R3  0  1  2  3  0  1  2  3
R4  4  5  6  7  4  5  6  7
R5  8  9 10 11  8  9 10 11

```

Comparing to `tile`, `repeat` function replicates each element in their adjacent places along specified dimension.

```ocaml
# let z = Arr.repeat x [|2;1|]
val z : Arr.arr =

   C0 C1 C2 C3
R0  0  1  2  3
R1  0  1  2  3
R2  4  5  6  7
R3  4  5  6  7
R4  8  9 10 11
R5  8  9 10 11

```


You can also expand the dimensionality of an ndarray, or squeeze out those dimensions having only one element, or even padding elements to an existing ndarray.

```text

  val expand : ('a, 'b) t -> int -> ('a, 'b) t

  val squeeze : ?axis:int array -> ('a, 'b) t -> ('a, 'b) t

  val pad : ?v:'a -> int list list -> ('a, 'b) t -> ('a, 'b) t

```

Another two useful functions are `concatenate` and `split`. `concatenate` allows us to concatenate an array of ndarrays along the specified axis. The constraint on the shapes is that, except the dimension for concatenation, the rest dimension must be equal. For matrices, there are two operators associated with concatenation: `@||` for concatenating horizontally (i.e. along axis 1); `@=` for concatenating vertically (i.e. along axis 0).

`split` is simply the inverse operation of concatenation.

```text

  val concatenate : ?axis:int -> ('a, 'b) t array -> ('a, 'b) t

  val split : ?axis:int -> int array -> ('a, 'b) t -> ('a, 'b) t array

```

You can also sort an ndarray but note that modification will happen in place.

```text

  val sort : ('a, 'b) t -> unit

```

Converting between ndarrays and OCaml native arrays can be efficiently done with these functions.

```text

  val of_array : ('a, 'b) kind -> 'a array -> int array -> ('a, 'b) t

  val to_array : ('a, 'b) t -> 'a array

```

Again, for matrix this special case, there are `to_arrays` and `of_arrays` two functions.



## Serialisation

Serialisation and deserialisation are simply done with `save` and `load` two functions.

```text

  val save : out:string -> ('a, 'b) t -> unit

  val load : ('a, 'b) kind -> string -> ('a, 'b) t

```

Note that you need to pass in type information in `load` function otherwise Owl cannot figure out what is contained in the chunk of binary file. Alternatively, you can use the corresponding `load` functions in `S/D/C/Z` module to save the type information.

```ocaml
# let x = Mat.uniform 8 8 in
  Mat.save "data.mat" x;
  let y = Mat.load "data.mat" in
  Mat.(x = y)
- : bool = true
```

`save` and `load` currently use the Marshall module which is brittle since it depends on specific OCaml versions. In the future, these two functions will be improved.


There are way more functions contained in the Ndarray module than the ones I have introduced here. Please refer to the API documentation for the full list.
