---
layout: post
title: Making Compile-Time Guarantees for Matrix Computations
comments: true
---

Numpy is a great library, a core library of scientific computing in Python. However,
without compile-time guarantees matrix computations, it's easy to push out code ridden
with errors.

In fact, any limited wrapper around BLAS or LAPACK is constrained by deep stack tracebacks
that are impossible to retrace.

For example, take the dot product:

info > 1

info < 1

Statically-typed languages like Rust have a the great advantage of providing compile-time
guarantees for code execution.

{% highlight rust %}
#[derive(Debug)]
pub enum MatrixError{
    MismatchedDimensions,
    NonSquareMatrix,
    MalformedMatrix,
    SingularMatrix,
    LapackComputationError,
    LapackInputError,
    UnknownError,
    IndexError
}

impl fmt::Display for MatrixError{

    fn fmt(&self, f : &mut fmt::Formatter ) -> fmt::Result{

        match *self{
            MatrixError::MismatchedDimensions=>
                write!(f, "Operation cannot be performed. Mismatched dimensions."),
            MatrixError::NonSquareMatrix=>
                write!(f, "Operation cannot be performed. Matrix is not square."),
            MatrixError::MalformedMatrix =>
                write!(f, "Matrix is malformed."),
            MatrixError::SingularMatrix =>
                write!(f, "Operation cannot be performed. Matrix has zero determinant."),
            MatrixError::LapackComputationError =>
                write!(f, "Failure in the course of computation."),
            MatrixError::LapackInputError =>
                write!(f, "Illegal argument detected."),
            MatrixError::IndexError =>
                write!(f, "Indexed outside of matrix bounds."),
            MatrixError::UnknownError =>
                write!(f,"Unknown error, please submit bug.")
        }
    }
}


trait RectMat : Num + Rand + Clone{
    fn new(e : Vec<f64>, r_size : usize, c_size : usize) -> Result<   Self, MatrixError>;
    fn random(r_size : usize, c_size: usize) -> Self ;
    fn replace(&mut self,row: usize, col:usize, value :f64) -> ();
    fn zeros (r_size : usize, c_size : usize) -> Self;
    fn diag_mat (a : Vec<f64>) -> Self;
    fn get_element(&self, row : usize, col : usize) -> f64;
    fn get_ind(&self, row :usize, col : usize) -> usize;
    fn transpose(&self) -> Self;
    fn diagonal (&self) -> Vec<f64>;
    fn tri (row_size:usize, col_size : usize, k : usize, upper_or_lower : Triangular) -> Result<Self,MatrixError>;
}

trait SqMat: RectMat{
    fn new(e : Vec<f64>, r_size : usize, c_size : usize) -> Result<   Self, MatrixError>;
    fn diag_mat (a : Vec<f64>) -> Self;
    fn identity(row_size : usize) -> Self;
    fn pseudoinverse(a: &mut Self) ->Result<Self,MatrixError>;
}

trait NonSingularMat : SqMat{
    fn inverse(a : &mut Self ) ->Result<Self,MatrixError>;
}


{% endhighlight %}


{% highlight rust %}
pub fn is_normal<T: SqMat>(a : &mut T) -> Result<bool, MatrixError> {

    let mut at = a.transpose();

    let inner = try!(dot (a, &mut at));
    let outer = try!(dot (&mut at, a));

    matrix_equal!(inner,outer)

}

{% endhighlight %}
