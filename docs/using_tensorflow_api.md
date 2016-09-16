Using the TensorFlow API from R
================

Introduction
------------

The **tensorflow** package provides access to the entire TensorFlow API from within R. The principle motivation for the package is to provide a foundation for higher-level R libraries that make building models with TensorFlow productive, straightforward, and well-integrated with the R ecosystem.

While the main motivation is the creation of higher-level R packages, it is also possible to use the lower-level TensorFlow API directly from R. The main technical consideration here is that substantial parts TensorFlow are implemented in Python, which requires bridging between the R and Python runtimes, data type conversion, etc. This document describes the R API to TensorFlow and the various idioms and techniques required to use it productively.

### Hello, TensorFlow

The R API to TensorFlow is based on direct invocation of TensorFlow's Python functions and objects. The two languages have many similarities so the R code will often look quite similar to the equivalent Python code. Here is the R version "Hello, World" for TensorFlow:

``` r
library(tensorflow)

sess = tf$Session()

hello <- tf$constant('Hello, TensorFlow!')
sess$run(hello)

a <- tf$constant(10L)
b <- tf$constant(32L)
sess$run(a + b)
```

Here is the equivalent Python code:

``` python
import tensorflow as tf

sess = tf.Session()

hello = tf.constant('Hello, TensorFlow!')
print(sess.run(hello))

a = tf.constant(10)
b = tf.constant(32)
print(sess.run(a + b))
```

Data Types
----------

When interacting with the TensorFlow API R data types are automatically converted to Python data-types and vice-versa, making the use of the API fairly seamless from R. Core data types are mapped as follows:

| R                      | Python/TensorFlow | Examples                                 |
|------------------------|-------------------|------------------------------------------|
| Single-element vector  | Scalar            | `1`, `1L`, `TRUE`, `"foo"`               |
| Multi-element vector   | List              | `c(1, 2, 3)`, `c(1L, 2L, 3L)`            |
| List of multiple types | Tuple             | `list(1, TRUE, "foo")`                   |
| Named list             | Dict              | `list(a = 1, b = 2)`                     |
| Matrix/Array           | NumPy ndarray     | `matrix(c(1,2,3,4), nrow = 2, ncol = 2)` |
| NULL, TRUE, FALSE      | None, True, False | `NULL`, `TRUE`, `FALSE`                  |

The automatic conversion of data types makes for R code which looks very similar to equivalent Python code (save for the `$` used in R which is analogous to the `.` which is used in Python). Here's another simple example of TensorFlow code in R:

``` r
library(tensorflow)

# Create 100 phony x, y data points, y = x * 0.1 + 0.3
x_data <- runif(100, min=0, max=1)
y_data <- x_data * 0.1 + 0.3

# Try to find values for W and b that compute y_data = W * x_data + b
# (We know that W should be 0.1 and b 0.3, but TensorFlow will
# figure that out for us.)
W <- tf$Variable(tf$random_uniform(shape(1), -1.0, 1.0))
b <- tf$Variable(tf$zeros(shape(1)))
y <- W * x_data + b

# Minimize the mean squared errors.
loss <- tf$reduce_mean((y - y_data) ^ 2)
optimizer <- tf$train$GradientDescentOptimizer(0.5)
train <- optimizer$minimize(loss)

# Launch the graph and initialize the variables.
sess = tf$Session()
sess$run(tf$initialize_all_variables())

# Fit the line (Learns best fit is W: 0.1, b: 0.3)
for (step in 1:201) {
  sess$run(train)
  if (step %% 20 == 0)
    cat(step, "-", sess$run(W), sess$run(b), "\n")
}
```

You can see the equivalent Python code here: <https://www.tensorflow.org/get_started/>. Aside from the use of the `shape` function (discussed in more detail below) the R and Python code are nearly identical.

There are however some important differences between how Python and R treat various data types. Navigating these differences generally requires the use of the helper functions described below.

### Shapes

Many TensorFlow APIs take a `shape` as input to indicate the dimensions of a tensor. Shapes are lists of integers within the Python API so can naively be passed as integer vectors from R. However, shapes can include `NULL` within their dimensions (to indicate any number of elements) and can also consist of only a single dimension. The `c` function within R doesn't handle `NULL` and will yield a scalar for a single-dimension so has some problems when dealing with shapes.

The solution is to use the `shape` function whenever you pass a shape to a TensorFlow API, for example:

``` r
x <- tf$placeholder(tf$float32, shape(NULL, 784))
W <- tf$Variable(tf$zeros(shape(784, 10)))
b <- tf$Variable(tf$zeros(shape(10)))
```

### Typed Lists

The issues with shape parameters discussed in the preceding section come up in other places within the TensorFlow API that require typed lists of values. The `int`, `float`, and `bool` functions can be used in place of the `c` function where a typed list is required. For example:

``` r
tf$nn$conv2d(x, W, strides=int(1, 1, 1, 1), padding='SAME')
```

The `shape` function is in fact merely a convenient alias for the `int` function which reads a bit more clearly than using `int` directly.

### Zero Based Arrays

Python lists and NumPy arrays are 0-based rather than 1-based (as in R). You can simply remember this when interacting with TensorFlow APIs, however if you prefer to use 1-based addressing you can do so using the `idx` function. For example:

``` r
# call tf$reduce_mean on the second dimension of the specified tensor
cross_entropy <- tf$reduce_mean(-tf$reduce_sum(y_ * tf$log(y_conv), reduction_indices=idx(2)))

# call tf$argmax on the second dimension of the specified tensor
correct_prediction <- tf$equal(tf$argmax(y_conv, idx(2)), tf$argmax(y_, idx(2)))
```

Again, you can also choose to just remember that TensorFlow arrays are 0-based and pass a 0-based index directly. If however you are creating a higher-level R wrapper for a TensorFlow API you should always make use of the `idx` function so callers don't need to navigate this difference.

### Default Numeric Type

In Python the default numeric type for a literal is an integer (you need to add a decimal point to get a floating point type). Whereas, in R, the default numeric type is floating point (you need to add the `L` suffix to force an integer).

Therefore, if there are TensorFlow APIs which require integers you should always remember to use the `L` suffix. That said, the `shape`, `int`, and `idx` functions above handle most of the common cases of integer coercion automatically so the need to use `L` may not come up that often.

### Dictionaries

To pass a dictionary keyed by character string to Python you can simply create an R named list and it will be automatically converted to a Python dictionary. However, Python dictionaries can also be keyed by arbitrary object types, and in particular by tensors within the TensorFlow API. To create a Python dictionary keyed by tensor you can use the `dict` function. For example:

``` r
sess$run(train_step, feed_dict = dict(x = batch_xs, y_ = batch_ys))
```

The `x` and `y_` variables in the above example are tensor placeholders which are substituted for by the specified training data.

### With Contexts

The TensorFlow API makes extensive use of the python `with` keyword for creating scoped execution contexts. R has a `with` generic function which is overloaded for use with TensorFlow objects for the same purpose. For example:

``` r
with(tf$name_scope('input'), {
  x <- tf$placeholder(tf$float32, shape(NULL, 784), name='x-input')
  y_ <- tf$placeholder(tf$float32, shape(NULL, 10), name='y-input')
})
```

There is also a custom `%as%` operator defined for creating a local alias to the object passed to the `with` function:

``` r
with(tf$Session() %as% sess, {
  sess$run(hello)
})
```

Getting Help
------------

As you use TensorFlow from R you'll want to get help on the various functions and classes available within the API. You can find the full reference to the TensorFlow Python API here: <https://www.tensorflow.org/api_docs/python/>. It's generally very easy to map Python constructs used in the documentation to R, and most R code will look nearly identical to it's Python counterpart.

If you are running the vary latest [Preview Release](https://www.rstudio.com/products/rstudio/download/preview/) (v1.0.18 or later) of RStudio IDE you can also get code completion and inline help for the TensorFlow API within RStudio. For example:

<img src="images/completion-functions.png" style="margin-bottom: 15px;" width="804" height="260" />

Inline help is also available for function parameters:

<img src="images/completion-params.png" style="margin-bottom: 15px;" width="804" height="177" />

You can also press the F1 key while viewing inline help (or when your cursor is over a TensorFlow API symbol) and you will be navigated to the location of that symbol's help within the [TensorFlow API](https://www.tensorflow.org/api_docs/python/) documentation.