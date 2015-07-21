Common situations where Stata silently returns something different from what you may have in mind. 

I have uploaded this list on Github so that it can be easily modified: feel free to [edit it](https://github.com/matthieugomez/stata-gotchas/edit/master/README.md).

# Missing observations

### Sort

Missing observations are sorted last in Stata. Therefore expressions such as
```
by id (time): gen temp = time[_N]
```
returns a missing value rather than the maximum time if `time` observation is missing. Instead, use

```
egen temp = max(time), by(id)
```

### Arithmetic operations

- *Explicit* arithmetic operations containing a missing value return a missing value

	```
	. clear all
	. set obs 1
	. gen a = 1
	. gen b = .
	. gen sum = a + b
	. gen product = a * b
	list
	   +-----------------------+
	   | a   b   sum   product |
	   |-----------------------|
	1. | 1   .     .         . |
	   +-----------------------+
	```
	In contrast, commands such as `egen` or `collapse` ignore missing values.

- `collapse (sum)` and `egen sum`, and `rowsum` transform missing values to zero. In particular, the sum of a variable over a set of missing observations is zero, rather than missing

	```
	. clear all
	. set obs 1
	. gen a = .
	. collapse (sum) sum = a (mean) mean = a (sd) sd = a
	. list
	   +-----------------+
	   | sum   mean   sd |
	   |-----------------|
	1. |   0      .    . |
	   +-----------------+
	```
	Note that, contrary to `sum`, the functions `mean` or `sd` return missing values : internally, these commands evaluate an expression of the form `0/0` (where the denominator is the number of non missing observations) which evaluates to missing in Stata.

	If you want the sum of missing observations to be missing rather than zero, use a temporary variable to count non - missing variables

	```
	. clear all
	. set obs 1
	. gen a = .
	. gen nonmissing = !missing(a)
	. collapse (sum) sum = a  anynonmissing = nonmissing
	. replace a = . if anynonmissing == 0
	. list

	    +--------------+
	    | a   nonmis~g |
	    |--------------|
	 1. | .          0 |
	    +--------------+
	```




	The issue is discussed on the Statalist [here](http://www.stata.com/statalist/archive/2004-07/msg00779.html), [here](http://www.stata.com/statalist/archive/2007-10/msg00806.html), [here](http://www.stata.com/statalist/archive/2010-02/msg00422.html)


- Don't use summary variables (i.e. obtained after `collapse` or `egen`) in a statistical model without checking first that these variables *are missing for the same observations.*

	As an example, the following code generates `y` and `x` such that `y = 2*x + runiform()`,  then set `x` to missing every 13 rows, and `y` to missing every 11 rows.

	```
	. clear all
	. set seed 10
	. set obs 10000
	. gen a = mod(_n, 1000)
	. gen x = uniform()
	. gen y = 2 * x + uniform()
	. replace y = . if mod(_n, 11) == 0
	. replace x = . if mod(_n, 13) == 0
	. collapse (mean) x y, by (a)
	```

	Regressing `y` on `x` yields a biased estimate for b:

	```
	. reg y x
	      Source |       SS       df       MS              Number of obs =    1000
	-------------+------------------------------           F(  1,   998) = 2330.48
	       Model |  32.6328302     1  32.6328302           Prob > F      =  0.0000
	    Residual |  13.9746306   998  .014002636           R-squared     =  0.7002
	-------------+------------------------------           Adj R-squared =  0.6999
	       Total |  46.6074608   999  .046654115           Root MSE      =  .11833
	
	------------------------------------------------------------------------------
	           y |      Coef.   Std. Err.      t    P>|t|     [95% Conf. Interval]
	-------------+----------------------------------------------------------------
	           x |   1.864172   .0386157    48.28   0.000     1.788395    1.939949
	       _cons |   .5655158   .0195333    28.95   0.000     .5271848    .6038469
	------------------------------------------------------------------------------
	```

	To avoid this situation, first flag/remove all missing observations where any of the variable is missing

	```
	. clear all
	. set seed 10
	. set obs 10000
	. gen a = mod(_n, 1000)
	. gen x = uniform()
	. gen y = 2 * x + uniform()
	. replace y = . if mod(_n, 11) == 0
	. replace x = . if mod(_n, 13) == 0
	. keep if !missing(x) & !missing(y)
	. collapse (mean) x y, by (a)
	. reg y x
	
	      Source |       SS       df       MS              Number of obs =    1000
	-------------+------------------------------           F(  1,   998) = 4291.64
	       Model |  42.2154102     1  42.2154102           Prob > F      =  0.0000
	    Residual |  9.81699173   998  .009836665           R-squared     =  0.8113
	-------------+------------------------------           Adj R-squared =  0.8111
	       Total |  52.0324019   999  .052084486           Root MSE      =  .09918
	
	------------------------------------------------------------------------------
	           y |      Coef.   Std. Err.      t    P>|t|     [95% Conf. Interval]
	-------------+----------------------------------------------------------------
	           x |   2.002737   .0305712    65.51   0.000     1.942746    2.062728
	       _cons |   .4996378   .0154767    32.28   0.000     .4692672    .5300085
	------------------------------------------------------------------------------
	```

	This situation is commonly encountered when constructing leave-one-out regressors.




# Boolean
Stata considers any value different from zero is considered as true


### Missing
Since missing values are considered to be higher than anything, they are, in particular, true. Therefore, in the following command, the last condition evaluates to true for all observations
```
. clear all
. set obs 10
. gen condition = 1 if _n >= 5
. keep if condition
(0 observations deleted)
```

To avoid this issue, create boolean variable alternatively equal 1 and 0,  rather than rather 1 and missing.

```
. clear all
. set obs 10
. gen condition =_n >= 5
. keep if condition
(4 observations deleted)
```

### Count
`egen count` and `collapse (count)` count the number of **non missing observations**, not the number of observations that evaluate to true. Therefore, the following code is wrong

```
. egen temp = count(state == 30)
```
Instead, use

```
. egen temp = sum(state == 30)
```




# Collapse
Beyond the issue with missing value mentioned above, `collapse` can give unexpected results when using weights.

-  Observations with missing or zero weights are removed from  *all* the computations, even those in (rawsum)

	```
	. clear all
	. set obs 2
	. gen a  = 1
	. gen w = 1 if _n == 1
	. collapse (rawsum) a [w = w]
	   +---+
	   | a |
	   |---|
	1. | 1 |
	   +---+
	```

	The topic is discussed on the Statalist [here](http://www.stata.com/statalist/archive/2013-09/msg00958.html), and [here](http://www.stata.com/statalist/archive/2010-03/msg01958.html)


-  `collapse (sum) [aw]` (the default) and `collapse (sum) [pw]` returns` \sum w_i v_i` where w_i is normalized so that \sum w_i = _N. 

	```
	. clear all
	. set obs 2
	. gen a  = _n
	. gen b = 1
	. gen w = _n
	. collapse (sum) a [aw = w]
	```

	If you want `collapse(sum) ` to return `\sum w_i v_i`, use `iweight` or `fweight`

	```
	. clear all
	. set obs 2
	. gen a  = _n
	. gen b = 1
	. gen w = _n
	. collapse (sum) a [iw = w]
	```


# Merge
- merge m:m does not create a dataset with all the possible combinations between the master and the using dataset. To do a "true" m:m join (i.e. a SQL Outer join or in R, a default `merge`), use `joinby`. The issue is discussed [here](http://www.stata.com/statalist/archive/2012-01/msg00773.html).
- When using the option `update`, the variable `_merge` takes values from 1 to 5
	```
	not matched                                 
	        from master                           (_merge==1)
	        from using                            (_merge==2)
	matched                                     
	        not updated                           (_merge==3)
	        missing updated                       (_merge==4)
	        nonmissing conflict                   (_merge==5)

	```
	In particular, usual options such as `keep if _merge == 3` takes a different meaning with the option `update`. To avoid this problem, use the `keep` option as in `merge 1:1 using usingfile, keep(matched)` which corresponds to `keep if inrange(_merge, 3, 5)`


# Type
Variables in Stata are stored in float (called "single precision" or Float32 in other languages) by default, which creates some specific issues.

### Float vs integer
Floats can store integers accurately only up to 2^24 = 16,777,216.  In other words, integers higher than 2^24 are rounded up or down. This makes float variables problematic to use as id variables when some id numbers are higher than 15 million (either because you have more than 16 million unique groups, or because you use EIN or SSN as identifiers). In these cases, be sure to store these id variables as `long` (up to 2 billions) or `double` rather than `float`. Below are three common situations: 


- Creating an identifying variable
	```
	. clear all
	. set obs 20000000
	. gen id = _n
	```	
	The `id` variable is, by default, a float. Therefore, it does not identify each row. 

	```
	. distinct id
	        |        Observations
	        |      total   distinct
	 -------+----------------------
	     id |   2.00e+07   1.84e+07
	```

	Rather, use `long` in the `generate` command
	```
	. clear all
	. set obs 20000000
	. gen long id = _n
	. distinct id
	       |        Observations
	       |      total   distinct
	-------+----------------------
	    id |   2.00e+07   2.00e+07
	```
- Converting a string to an integer, `real()` converts the string variable to a float by default. Rather, use 
	
	````
	. gen long id_int = real(id)
	```
	or
	```
	. destring id
	```
	which automatically creates a double


- Importing a `csv` file. If the `csv` file contains integers that can't be stored in a float type without loss of accuracy, force all variables to be imported as double

	```
	. import delimited filename, asdouble
	```

	Once the dataset is imported, you may convert the non-id variables from `double` to `float` using  `recast`


### Float vs double

Floats should not be used in equality conditions
```
. clear all
. set obs 1
. gen a = 1.1
. display a[1] == 1.1
0
```

While `a` is a stored as a float, 1.1 is a double, and therefore they are not exactly equal. A solution is to apply the `float` function to every non-float expression
```
. display a[1] == float(1.1)
1
```

A better solution is probably to avoid comparing floating numbers to other floating numbers (i.e. convert these variables to strings or as long integers with a value label).

Types issues are discusses [here](http://blog.stata.com/tag/precision/)

# Regex
Stata does not support positive / negative look aheads / behind, nor counting operators (using braces).

# Panel Data
- To lag a variable `v` in an unbalanced panel, use the operators `L.v` and `F.v` (which return the variable value at the past/future period) rather than `v[n-1]` and `v[n+1]` (which return the variable value at the previous/next row)