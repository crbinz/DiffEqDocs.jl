# Delay Differential Equations

This tutorial will introduce you to the functionality for solving delay differential
equations.

Delay differential equations are equations which have a delayed argument. To allow
for specifying the delayed argument, the function definition for a delay differential
equation is expanded to include a history function `h(t)` which uses interpolations
throughout the solution's history to form a continuous extension of the solver's
past. The function signature for a delay differential equation is `f(t,u,h)` for
not in-place computations, and `f(t,u,h,du)` for in-place computations.

In this example we will solve [a model of breast cancer growth kinetics](http://www.nature.com/articles/srep02473):

```math
\begin{align}
dx_{0} &= \frac{v_{0}}{1+\beta_{0}\left(x_{2}(t-\tau)\right)^{2}}\left(p_{0}-q_{0}\right)x_{0}(t)-d_{0}x_{0}(t)\\
dx_{1} &= \frac{v_{0}}{1+\beta_{0}\left(x_{2}(t-\tau)\right)^{2}}\left(1-p_{0}+q_{0}\right)x_{0}(t)\\
       +& \frac{v_{1}}{1+\beta_{1}\left(x_{2}(t-\tau)\right)^{2}}\left(p_{1}-q_{1}\right)x_{1}(t)-d_{1}x_{1}(t)\\
dx_{2} &= \frac{v_{1}}{1+\beta_{1}\left(x_{2}(t-\tau)\right)^{2}}\left(1-p_{1}+q_{1}\right)x_{1}(t)-d_{2}x_{2}(t)
\end{align}
```

For this problem we note that ``\tau`` is constant, and thus we can use a method
which exploits this behavior. We first write out the equation using the appropriate
function signature. Most of the equation writing is the same, though we use the
history function by first interpolating and then choosing the components. Thus
the `i`th component at time `t-tau` is given by `h(t-tau)[i]`. Components with
no delays are written as in the ODE.

Thus, the function for this model is given by:

```julia
const p0 = 0.2; const q0 = 0.3; const v0 = 1; const d0 = 5
const p1 = 0.2; const q1 = 0.3; const v1 = 1; const d1 = 1
const d2 = 1; const beta0 = 1; const beta1 = 1; const tau = 1
function bc_model(t,u,h,du)
  du[1] = (v0/(1+beta0*(h(t-tau)[3]^2))) * (p0 - q0)*u[1] - d0*u[1]
  du[2] = (v0/(1+beta0*(h(t-tau)[3]^2))) * (1 - p0 + q0)*u[1] +
          (v1/(1+beta1*(h(t-tau)[3]^2))) * (p1 - q1)*u[2] - d1*u[2]
  du[3] = (v1/(1+beta1*(h(t-tau)[3]^2))) * (1 - p1 + q1)*u[2] - d2*u[3]
end
```

To use the constant lag model, we have to declare the lags. Here we will use `tau=1`.

```julia
lags = [tau]
```

Now we build a `ConstantLagDDEProblem`.  The signature is very similar to ODEs,
where we now have to give the lags and
an `h`. `h` is the history function, or a function that declares what the values
were before the time the model starts. Here we will assume that for all time before
`t0` the values were 1:

```julia
h(t) = ones(3)
```

We have `h` output a 3x1 vector since our differential equation is given by
a system of the same size. Next, we choose to solve on the timespan `(0.0,10.0)`
and create the problem type:


```julia
tspan = (0.0,10.0)
prob = ConstantLagDDEProblem(bc_model,h,u0,lags,tspan)
```

An efficient way to solve this problem (given the constant lags) is with the
MethodOfSteps solver. Through the magic that is Julia, it translates an OrdinaryDiffEq.jl
ODE solver method into a method for delay differential equations which is highly
efficient due to sweet compiler magic. A good choice is the order 5 `Tsit5()`
method:

```julia
alg = MethodOfSteps(Tsit5())
```

For lower tolerance solving, one can use the `BS3()` algorithm to good effect (this
combination is similar to the MATLAB `dde23`), and
for high tolerances the `DP8()` algorithm will give an 8th order solution. Note
that the Verner methods will not work here due to their lazy interpolation scheme.

To solve the problem with this algorithm, we do the same thing we'd do with other
methods on the common interface:

```julia
sol = solve(prob,alg)
```

Note that everything available to OrdinaryDiffEq.jl can be used here, including
event handling and other callbacks. The solution object has the same interface
as for ODEs. For example, we can use the same plot recipes to view the results:

```julia
using Plots; plot(sol)
```

![DDE Example Plot](../assets/dde_example_plot.png)

### State-Dependent Delays

State-dependent delays are problems where the delay is allowed to be a function
of the current state. To do this in DifferentialEquations.jl, one simply writes
it in the natural manner `h(g(t,u))` where `g` is the lag function. As before,
you must declare the lag functions to the solver. Other than that, everything
else is the same, where one instead constructs a `DDEProblem` type:

```julia
prob = DDEProblem(f,h,u0,lags,tspan)
```

and solves that using the common interface.

However, currently there are no good algorithms for this. The developers are
working on a defect control method which will give good quality results
and have strict error bounds.
