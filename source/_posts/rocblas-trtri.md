---
title: rocBLAS矩阵求逆函数trtri()源码分析（更新中）
date: 2020-04-23 21:36:22
tags: BLAS
category: 异构计算  
---

本文的分析基于rocmBLAS的3.3.0版本，源代码的著作权属于AMD公司，其GitHub repo在https://github.com/ROCmSoftwarePlatform/rocBLAS 。  

## rocBLAS数据类型说明  

### `rocblas_float_complex`
单精度复数。  
*本质是`float2`*  
```c++ library/include/rocblas-complex-types.h
typedef struct
{
    float x, y;
} rocblas_float_complex;
```

### `rocblas_double_complex`
双精度复数。  
*本质是`double2`*  
```c++ library/include/rocblas-complex-types.h
typedef struct
{
    double x, y;
} rocblas_double_complex;
```
### `rocblas_handle`
rocBLAS库中使用的上下文队列的句柄。下面这个代码块告诉我们其本质是指向`_rocblas_handle`的实例的指针。  
```c++ library/include/rocblas-type.h
typedef struct _rocblas_handle* rocblas_handle;
```
而`_rocblas_handle`则是近300行的结构体，定义在`library/src/include/handle.h`中。<i>上下文的概念请参见我以后会写的OpenCL原理or cuda分析<s>（咕咕咕）</s>，</i>这里我简要引用其部分注释：  
> `_rocblas_handle` is a structure holding the rocblas library context. It must be initialized using `rocblas_create_handle()` and the returned handle mus. It should be destroyed at the end using `rocblas_destroy_handle()`.  
> Exactly like CUBLAS, ROCBLAS only uses one stream for one API routine.
   
这是在说，`_rocblas_handle`是一个包含rocBLAS库的上下文的结构体。该结构体的初始化必须由函数`rocblas_create_handle()`来进行，并返回一个句柄；该结构体的销毁也必须通过专门的函数`rocblas_destroy_handle()`来完成。  
而且和老黄的cuBLAS一样，苏妈的rocBLAS一个API流程中只使用一个stream。  

### `rocblas_fill`
是否是三角矩阵。  
```c++ library/include/rocblas-types.h
typedef enum rocblas_fill_
{
    rocblas_fill_upper = 121, /**< Upper triangle. */
    rocblas_fill_lower = 122, /**< Lower triangle. */
    rocblas_fill_full  = 123
} rocblas_fill;
```

### `rocblas_diagonal`
是否是单位三角矩阵。  
*主对角线上的元素都是1的三角矩阵是单位三角矩阵。*  
```c++ library/include/rocblas-types.h
typedef enum rocblas_diagonal_
{
    rocblas_diagonal_non_unit = 131, /**< Non-unit triangular. */
    rocblas_diagonal_unit     = 132, /**< Unit triangular. */
} rocblas_diagonal;
```

## 复数的运算符重载
TODO  

## API分析
rocBLAS中矩阵求逆函数有三种12个。三种分别是：

* `rocblas_<type>trtri()`  
这个函数用于计算矩阵A的逆矩阵，即inv(A)。输出的inv(A)会覆盖在原来的A的空间里。  

* `rocblas_<type>trtri_batched()`  
这个函数用于一次计算n个矩阵的逆矩阵。换而言之就是求矩阵A_i的逆矩阵inv(A_i)，不过{A_i, i = 0..n-1}这n个矩阵是装在同一个二维数组里的。另，输出的n个inv(A_i)也覆盖了输入数组的内存空间。  

* `rocblas_<type>trtri_strided_batched()`   


其中的`<type>`字段分别都可以替换成`s`、`d`、`c`、`z`，分别代表该函数处理的矩阵的数据类型是`float`、`double`、`float_2`、`double_2`。  

> `s`可以理解为single单精度；`d`则是double双精度；`c`就是complex复数；`z`我不确定它的来由。  

rocBLAS在C语言版的接口中，则依次定义了这12个函数。这些函数先令`rocblas_int NB = 16;`，再`try`相应的`rocblas_trtri_impl()`。一旦`throw`了异常，`catch`就会返回`exception_to_rocblas_status()`的返回值。  
这里以`rocblas_strtri()`为例：  
```c++ library/src/blas3/rocblas_trtri.cpp
rocblas_status rocblas_strtri(rocblas_handle   handle,
                              rocblas_fill     uplo,
                              rocblas_diagonal diag,
                              rocblas_int      n,
                              const float*     A,
                              rocblas_int      lda,
                              float*           invA,
                              rocblas_int      ldinvA)
try
{
    constexpr rocblas_int NB = 16;
    return rocblas_trtri_impl<NB>(handle, uplo, diag, n, A, lda, invA, ldinvA);
}
catch(...)
{
    return exception_to_rocblas_status();
}
```
函数`rocblas_strtri()`的参数有：  
* [in] `handle`: `rocblas_handle`  
这是rocBLAS库中负责上下文（context）队列的句柄。  

* [in] `uplo`: `rocblas_fill`  
输入矩阵A是上三角矩阵还是下三角矩阵，或者就是普通的矩阵。如果是上三角矩阵，那么A的下半部分就不会被使用；下三角矩阵同理。  

* [in] `diag`: `rocblas_diagonal`  
输入矩阵A是否是单位三角矩阵。  

* [in] `n`: `rocblas_int`  
输入矩阵A的边长。  

* [in] `A`: `float*`  
输入矩阵A的数组的指针。  

* [in] `lda`:  `rocblas_int`  
输入矩阵A的“主序元素”个数。因为矩阵A是以一维数组的形式来输入的，lda用来界定第一行（这里是C语言）有几个元素。  
> Assuming the fortran column-major ordering (which is the case in LAPACK), the LDA is used to define the distance in memory between elements of two consecutive columns which have the same row index. If you call B = A(91:100 , 1:100) then B(1,1) and B(1,2) are 100 memory locations far from each other.  
> *http://icl.cs.utk.edu/lapack-forum/viewtopic.php?f=2&t=217*  

* [out] `invA`: `float*`  
输出结果invA的指针。  

* [in] `ldinvA`: `rocblas_int`  
输入矩阵inv(A)的“主序元素”个数。  


### 函数`rocblas_<type>trtri()`
先上源码。开头的这一段是模板特化的操作，再配上C++11中新增的“常量表达式”`constexpr`关键字，希望在编译阶段，通过判断数组A的`typename`来给`rocblas_trtri_name[]`赋值：如果`typename`是`float`，那么`rocblas_trtri_name[] = "rocblas_strtri"`。  
```c++ library/src/blas3/rocblas_trtri.cpp
template <typename>
constexpr char rocblas_trtri_name[] = "unknown";
template <>
constexpr char rocblas_trtri_name<float>[] = "rocblas_strtri";
template <>
constexpr char rocblas_trtri_name<double>[] = "rocblas_dtrtri";
template <>
constexpr char rocblas_trtri_name<rocblas_float_complex>[] = "rocblas_ctrtri";
template <>
constexpr char rocblas_trtri_name<rocblas_double_complex>[] = "rocblas_ztrtri";

template <rocblas_int NB, typename T>
rocblas_status rocblas_trtri_impl(rocblas_handle   handle,
                                  rocblas_fill     uplo,
                                  rocblas_diagonal diag,
                                  rocblas_int      n,
                                  const T*         A,
                                  rocblas_int      lda,
                                  T*               invA,
                                  rocblas_int      ldinvA)
{
```

接下来则是一系列判断，确定是否相应的log模式  
另一方面保证输入的矩阵能正常参与后续的矩阵运算  

> `layer_mode`会被`auto`自动匹配成`rocblas_layer_mode`类型。  
> ```
> rocblas_layer_mode_none        = 0b0000000000
> rocblas_layer_mode_log_trace   = 0b0000000001
> rocblas_layer_mode_log_bench   = 0b0000000010
> rocblas_layer_mode_log_profile = 0b0000000100
> ```

值得一提的是，这个函数输入的一定得是三角矩阵，不然只能返回一个`rocblas_status_not_implemented`  

```c++ library/src/blas3/rocblas_trtri.cpp
    if(!handle)
        return rocblas_status_invalid_handle;
     auto layer_mode = handle->layer_mode;
    if(layer_mode & rocblas_layer_mode_log_trace)
        log_trace(handle, rocblas_trtri_name<T>, uplo, diag, n, A, lda, invA, ldinvA);

    if(layer_mode & rocblas_layer_mode_log_profile)
        log_profile(handle,
                    rocblas_trtri_name<T>,
                    "uplo",
                    rocblas_fill_letter(uplo),
                    "diag",
                    rocblas_diag_letter(diag),
                    "N",
                    n,
                    "lda",
                    lda,
                    "ldinvA",
                    ldinvA);
    if(uplo != rocblas_fill_lower && uplo != rocblas_fill_upper)
        return rocblas_status_not_implemented;
    if(n < 0)
        return rocblas_status_invalid_size;
    if(!A)
        return rocblas_status_invalid_pointer;
    if(lda < n)
        return rocblas_status_invalid_size;
    if(!invA)
        return rocblas_status_invalid_pointer;
```



```c++ library/src/blas3/rocblas_trtri.cpp
    size_t size = rocblas_trtri_temp_size<NB>(n, 1) * sizeof(T);
    if(handle->is_device_memory_size_query())
        return handle->set_optimal_device_memory_size(size);

    auto mem = handle->device_malloc(size);
    if(!mem)
        return rocblas_status_memory_error;

    return rocblas_trtri_template<NB, false, false, T>(handle,
                                                       uplo,
                                                       diag,
                                                       n,
                                                       A,
                                                       0,
                                                       lda,
                                                       lda * n,
                                                       0,
                                                       invA,
                                                       0,
                                                       ldinvA,
                                                       ldinvA * n,
                                                       0,
                                                       1,
                                                       1,
                                                       (T*)mem);
}
```
