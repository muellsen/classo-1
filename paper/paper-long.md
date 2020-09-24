---
title: 'c-lasso - a package for constrained sparse and robust regression and classification in Python'
tags:
  - Python
  - Linear regression
  - Optimization
authors:
  - name: Léo Simpson
    affiliation: 1
  - name: Patrick L. Combettes
    affiliation: 2
  - name: Christian L. Müller
    affiliation: 3
affiliations:
 - name: TUM  
   index: 1
 - name: NC State
   index: 2
 - name: LMU
   index: 3
date: 13 August 2020
bibliography: paper.bib

# Optional fields if submitting to a AAS journal too, see this blog post:

---

# Summary

This article illustrates c-lasso, a Python package that enables sparse and robust linear
regression and classification with linear equality constraints. 


The forward model is assumed to be:

$$
y = X \beta + \sigma \epsilon \qquad \textrm{s.t.} \qquad C\beta=0
$$

Here, $X$ and $y$ are given outcome and predictor data. The vector y can be continuous (for regression) or binary (for classification). $C$ is a general constraint matrix. The vector $\beta$ comprises the unknown coefficients and $\sigma$ an unknown scale.


# Statement of need 

The package handles several estimators for inferring location and scale, including the constrained Lasso, the constrained scaled Lasso, and sparse Huber M-estimation with linear equality constraints Several algorithmic strategies, including path and proximal splitting algorithms, are implemented to solve the underlying convex optimization problems. We also include two model selection strategies for determining the sparsity of the model parameters: k-fold cross-validation and stability selection. This package is intended to fill the gap between popular python tools such as `scikit-learn` which <em>cannot</em> solve sparse constrained problems and general-purpose optimization solvers such as `cvx` that do not scale well for the considered problems or are inaccurate. We show several use cases of the package, including an application of sparse log-contrast regression tasks for compositional microbiome data. We also highlight the seamless integration of the solver into `R` via the `reticulate` package. 


# Current functionalities

## Formulations 

Depending on the prior on the solution $\beta, \sigma$ and on the noise $\epsilon$, the previous forward model can lead to different types of estimation problems. 

Our package can solve six of those : four regression-type and two classification-type formulations.


### *R1* Standard constrained Lasso regression:             

$$
    \arg \min_{\beta \in \mathbb{R}^d} \left\lVert X\beta - y \right\rVert^2 + \lambda \left\lVert \beta\right\rVert_1 \qquad s.t. \qquad  C\beta = 0
$$

This is the standard Lasso problem with linear equality constraints on the $\beta$ vector. 
The objective function combines Least-Squares for model fitting with $L_1$ penalty for sparsity.   
 

### *R2* Contrained sparse Huber regression:                   

$$
    \arg \min_{\beta \in \mathbb{R}^d} h_{\rho} (X\beta - y) + \lambda \left\lVert \beta\right\rVert_1 \qquad s.t. \qquad  C\beta = 0
$$

This regression problem uses the [Huber loss](https://en.wikipedia.org/wiki/Huber_loss) as objective function 
for robust model fitting with $L_1$ and linear equality constraints on the $\beta$ vector. The parameter $\rho$ is set to $1.345$ by default [@Aigner:1976]

### *R3* Contrained scaled Lasso regression: 

$$
    \arg \min_{\beta \in \mathbb{R}^d} \frac{\left\lVert X\beta - y \right\rVert^2}{\sigma} + \frac{n}{2} \sigma + \lambda \left\lVert \beta\right\rVert_1 \qquad s.t. \qquad  C\beta = 0
$$



This formulation is similar to *R1* but allows for joint estimation of the (constrained) $\beta$ vector and 
the standard deviation $\sigma$ in a concomitant fashion [@Combettes:2020.1; @Combettes:2020.2].
This is the default problem formulation in c-lasso.

### *R4* Contrained sparse Huber regression with concomitant scale estimation:        

$$
    \arg \min_{\beta \in \mathbb{R}^d} \left( h_{\rho} \left( \frac{X\beta - y}{\sigma} \right) + n \right) \sigma + \lambda \left\lVert \beta\right\rVert_1 \qquad s.t. \qquad  C\beta = 0
$$

This formulation combines *R2* and *R3* to allow robust joint estimation of the (constrained) $\beta$ vector and 
the scale $\sigma$ in a concomitant fashion [@Combettes:2020.1; @Combettes:2020.2].

### *C1* Contrained sparse classification with Square Hinge loss: 

$$
    \arg \min_{\beta \in \mathbb{R}^d} L(y^T X\beta - y) + \lambda \left\lVert \beta\right\rVert_1 \qquad s.t. \qquad  C\beta = 0
$$

where $L \left((r_1,...,r_n)^T \right) := \sum_{i=1}^n l(r_i)$ and $l$ is defined as :

$$
l(r) = \begin{cases} (1-r)^2 & if \quad r \leq 1 \\ 0 &if \quad r \geq 1 \end{cases}
$$

This formulation is similar to *R1* but adapted for classification tasks using the Square Hinge loss with (constrained) sparse $\beta$ vector estimation [@Lee:2013].

### *C2* Contrained sparse classification with Huberized Square Hinge loss:        

$$
    \arg \min_{\beta \in \mathbb{R}^d} L_{\rho}(y^T X\beta - y) + \lambda \left\lVert \beta\right\rVert_1 \qquad s.t. \qquad  C\beta = 0
$$

where $L_{\rho} \left((r_1,...,r_n)^T \right) := \sum_{i=1}^n l_{\rho}(r_i)$ and $l_{\rho}$ is defined as :

$$
l_{\rho}(r) = \begin{cases} (1-r)^2 &if \quad \rho \leq r \leq 1 \\ (1-\rho)(1+\rho-2r) & if \quad r \leq \rho \\ 0 &if \quad r \geq 1 \end{cases}
$$

This formulation is similar to *C1* but uses the Huberized Square Hinge loss for robust classification with (constrained) sparse $\beta$ vector estimation [@Rosset:2007].




## Optimization schemes

The available problem formulations *R1-C2* require different algorithmic strategies for 
efficiently solving the underlying optimization problem. We have implemented four 
algorithms (with provable convergence guarantees) that vary in generality and are not 
necessarily applicable to all problems. For each problem type, c-lasso has a default algorithm 
setting that proved to be the fastest in our numerical experiments.

- **Path algorithms (*Path-Alg*)** : 
This is the default algorithm for non-concomitant problems *R1,R2,C1,C2*. 
The algorithm uses the fact that the solution path along &lambda; is piecewise-affine (as shown, e.g., in [@Gaines:2018]). When Least-Squares is used as objective function, we derive a novel efficient procedure that allows us to also derive the solution for the concomitant problem *R3* along the path with little extra computational overhead.

- **Projected primal-dual splitting method (*P-PDS*)** : 
This algorithm is derived from [@Briceno:2020] and belongs to the class of 
proximal splitting algorithms. It extends the classical Forward-Backward (FB) 
(aka proximal gradient descent) algorithm to handle an additional linear equality constraint
via projection. In the absence of a linear constraint, the method reduces to FB.
This method can solve problem *R1*. For the Huber problem *R2*, 
P-PDS can solve the mean-shift formulation of the problem  [@Mishra:2019].

- **Projection-free primal-dual splitting method (*PF-PDS*)** :
This algorithm is a special case of an algorithm proposed in [@Combettes:2011] (Eq.4.5) and also belongs to the class of 
proximal splitting algorithms. The algorithm does not require projection operators 
which may be beneficial when C has a more complex structure. In the absence of a linear constraint, 
the method reduces to the Forward-Backward-Forward scheme. This method can solve problem *R1*. 
For the Huber problem *R2*, PF-PDS can solve the mean-shift formulation of the problem [@Mishra:2019].

- **Douglas-Rachford-type splitting method (*DR*)** : 
This algorithm is the most general algorithm and can solve all regression problems 
*R1-R4*. It is based on Doulgas Rachford splitting in a higher-dimensional product space.
It makes use of the proximity operators of the perspective of the LS objective [@Combettes:2020.1; @Combettes:2020.2].
The Huber problem with concomitant scale *R4* is reformulated as scaled Lasso problem 
with the mean shift [@Mishra:2019] and thus solved in (n + d) dimensions.

## Model selections

Different models are implemented together with the optimization schemes, to overcome the difficulty of choosing the penalization free parameter $\lambda$. 

- *Fixed Lambda* : This approach is simply letting the user choose the parameter $\lambda$, or to choose $l \in [0,1]$ such that $\lambda = l\times \lambda_{\max}$. 
The default value is a scale-dependent tuning parameter that has been proposed in [Combettes:2020.2] and derived in [@Shi:2016].

- *Path Computation* :The package also leaves the possibility to us to compute the solution for a range of $\lambda$ parameters in an interval $[\lambda_{\min}, \lambda_{\max}]$. It can be done using *Path-Alg* or warm-start with any other optimization scheme. 

[comment]: <> (This can be done much faster than by computing separately the solution for each $\lambda$ of the grid, by using the Path-alg algorithm. One can also use warm starts : starting with $\beta_0 = 0$ for $\lambda_0 = \lambda_{\max}$, and then iteratvely compute $\beta_{k+1}$ using one of the optimization schemes with $\lambda = \lambda_{k+1} := \lambda_{k} - \epsilon$ and with a warm start set to $\beta_{k}$. )

- *Cross Validation* : Then one can use a model selection, to choose the appropriate penalisation. This can be done by using k-fold cross validation to find the best $\lambda \in [\lambda_{\min}, \lambda_{\max}]$ with or without "one-standard-error rule" [@Hastie:2009].

- *Stability Selection* : Another variable selection model than can be used is stability selection [@Lin:2014; @Meinshausen:2010; Combettes:2020.2].

# Basic workflow


Here is a basic example that shows how to run c-lasso on synthetic data.

#### Installation 
c-lasso is available on pip. You can install the package
in the shell using

```shell
pip install c_lasso
```
To use the c-lasso package in Python, type 

```python
from classo import *
```


The c-lasso package depends on several standard Python packages. 
The dependencies are included in the package. Those are, namely : 

`numpy` ; 
`matplotlib` ; 
`scipy` ; 
`pandas` ; 
`h5py` . 


#### Generate random data
The c-lasso package includes
the routine ```random_data``` that allows you to generate problem instances using normally distributed data.

```python
n,d,d_nonzero,k,sigma =100,100,5,1,0.5
(X,C,y),sol = random_data(n,d,d_nonzero,k,sigma,zerosum=True, seed = 123 )
```
This code snippet generates randomly the vectors $\beta \in R^d$ , $X \in R^{n\times d}$ , $C \in R^{k\times d}$ (here it is the all-one vector instead because of the input ```zerosum```), and $y \in R^n$ normally distributed with respect to the model $C\beta=0$, $y-X\beta \sim N(0,I_n\sigma^2)$ and $\beta$ has only d_nonzero non-null componant.

#### Use of c-lasso on the data
Here is an example of problem instance one can create with those set of data. 

```python
# let's define a c-lasso problem instance with default setting
problem  = classo_problem(X,y,C)


# let's change the formulation of the problem
problem.formulation.huber  = True
problem.formulation.concomitant = False
problem.formulation.rho = 1.5


# let's add a computation of beta for a fixed lambda 
problem.model_selection.LAMfixed = True
# and set it to to 0.1*lambdamax
problem.model_selection.LAMfixedparameters.rescaled_lam = True
problem.model_selection.LAMfixedparameters.lam = 0.1

# let's add a computation of the lambda-path
problem.model_selection.PATH = True


# let's solve our problem instance
problem.solve()
```

Here, we have modified the [formulation](##formulations) of the problem in order to use [*R2*](###*R2*-contrained-sparse-Huber-regression), with $\rho=1.5$. 

Then we have chosen the [model selections](##model-selections) we want to compute : *Fixed Lambda* with $\lambda = 0.1\lambda_{\max}$ ; *Path computation* and *Stability Selection* which is computed by default. 

Finally, those problems are solved using the method `solve` which computes everything. 

#### Visualize the result 
One can, before or after having solve the problem, plot the main caracteristics of the problem solved and of its solution: 

```python
>>> # let's visualize the main parameters set in the problem instance
>>> problem

FORMULATION: R2
 
MODEL SELECTION COMPUTED:  
     Lambda fixed
     Path
     Stability selection
 
LAMBDA FIXED PARAMETERS: 
     numerical_method = DR
     rescaled lam : True
     threshold = 0.177
     lam = 0.1
     theoretical_lam = 0.1994
 
PATH PARAMETERS: 
     numerical_method : Path-Alg
     Npath = 40
     lamin = 0.013
     lamax = 1.0
 
STABILITY SELECTION PARAMETERS: 
     numerical_method : Path-Alg
     method : first
     B = 50
     q = 10
     percent_nS = 0.5
     threshold = 0.7
     lamin = 0.01
     Nlam = 50

>>> # let's the solutions found
>>> problem.solution

 LAMBDA FIXED : 
   Selected variables :  43    47    74    79    84    
   Running time :  0.094 s

 PATH COMPUTATION : 
   Running time :  0.221 s

 STABILITY SELECTION : 
   Selected variables :  43    47    74    79    84    
   Running time :  2.468 s

```

![Graphics plotted after calling problem.solution ](figures/figure-concat.png)



As this variable selection has been computed for generated data, one can plot the real relevant variables :

```python
>>> print( list(numpy.nonzero(sol)) )
[43, 47, 74, 79, 84]
```

It is indeed the variables that have been selected with the solution threshold for a fixed lambda, and with stability selection. Let us also note that the running time is still very low in our example. 

Those remarks are comforting, but not surprising because in this example the noise is little and the number of variable is still small. 



# Acknowledgements

This work was supported by the Flatiron Institute and the Helmholtz Zentrum Munchen. 

# References


