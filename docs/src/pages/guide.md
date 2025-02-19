# Basic Usage
## Introduction
This page will guide you through generating and simulating the MiCRM model as described by the set of equations:

```math
\begin{aligned}
    \frac{dC_i}{dt} &= \sum_{\alpha = 0}^{M} C_i R_{\alpha} u_{i\alpha}  (1 - \sum_{\beta = 0}^{M} l_{\alpha,\beta}) - C_i m_i \\
    \frac{dR_\alpha}{dt} &= \rho_{\alpha} - R_{\alpha} \omega_{\alpha} - \sum_{i = 0}^{N} C_i R_{\alpha} u_{i\alpha} + \sum_{i = 0}^{N} \sum_{\beta = 0}^{M} C_i R_{\beta} u_{i \beta} l_{\beta \alpha}
\end{aligned}
```
The `MiCRM` package provides two ways to simulate the MiCRM system depending on your requirements:
1. Directly using the `DifferentialEquations` package with the `ODEProblem` function.
2. Using the symbolic reperesntation provided by `ModelingToolKit` which allows symbolic manipualtion of the system in addition to numerical integration. 

In general we recomend using the second method with the `MTK` which allows for greater flexibility and is significantly faster for simulations due to the automaticly generated derivative functions and jacobian matrix. This method does however incur a significant intial computational cost whist contstructing the symbollic representation so for cases where you want to play around with system size it may be faster to use the first direct numberical intergration method. 

## Method 1: Numerical Integration Only

Explain how to directly simualte with manual derivative function. Note that this is slower and less flexible (no symbolic computation) but does not require construction of the problem at runtime. 

## Method 2: Symbolic representation and `ModelingToolKit`

This method requires 4 steps  is divided into x steps:
1. Set parameters
2. Generate `ODESystem`
3. Convert to `ODEProblem`
4. Simulate system

### Setting Parameters
First we must generate a dictionary with values for the parameters of the MiCRM model. The MiCRM is described by the set of equations above which have the following set of parameters:

| Parameter    	| Description                                                                  	| Key  	| Type              	| Required? |
|--------------	|------------------------------------------------------------------------------	|------	|-------------------    |-----------|
| ``C_i``      	| Biomass of the ``i``th consumer                                              	| -    	| `Float64`         	| ✖         |
| ``R_{\alpha}``| Mass of the ``\alpha``th resource                                             | -    	| `Float64`         	| ✖         |
| ``N``        	| Number of consumer populations                                               	| `:N` 	| `Int64`           	| ✔         |
| ``M``        	| Number of resources                                                          	| `:M` 	| `Int64`           	| ✔         |
| ``u_{i \alpha}``| Uptake rate of the ``\alpha``th resource  by the `i`th consumer             | `:u` 	| `Matrix{Float64}` 	| ✔         |
| ``m_i``      	| Loss term for the ``i``th consumer                                           	| `:m` 	| `Vector{Float64}` 	| ✔         |
| ``\rho_{\alpha}``| Inflow rate for the ``\alpha``th resource                                  | `:ρ` 	| `Vector{Float64}` 	| ✔         |
| ``\omega_{\alpha}``| Outflow term for the ``j``resource                                       | `:ω` 	| `Vector{Float64}` 	| ✔         |
| ``l_{\alpha \beta}``| Proportion of uptake of the ``\alpha``th resource leaked to the ``\beta``th resource.| `:l` 	| `Matrix{Float64}` 	| ✔         |

The parameter dictionary must have the entries indicated above with the relevant key-value pairs. To aid in the construction of these parameter the package comes with several functions to randomly generate these parameter dictionaries with different communtiy structures (see #generating communities). 

In this example we will choose the simplest communtiy generation function `random_community` which generates communities with uniform random uptake and leakage matricies. 

```@example
using MiCRM, Random #hide
Random.seed!(1) #hide
N,M = 2,2 #set system size 
params = random_micrm_params(N,M, 0.3) #leakage = 0.3
```

### Generating the `ODESystem`

Once we have our parameter dictionary we next generate an `ODESystem` object using the provided `micrm_system` function. This function takes the parameter dictionary provided, creates a symbolic representation of the MiCRM system and then wraps it in an `ODESystem` object. The `ODESystem` object is from the package `ModelingToolKit` and provides a symbolic represntation of the system as well as many helper functions to extract and calculate properties of interest such as the Jacobian. The function also optionally takes a threshold biomass extinction value `δ` below which consumer populations are considered extinct during the numerical simulations.  

```@example
using MiCRM, Random #hide
Random.seed!(1) #hide
N,M = 2,2 #set system size #hide
params = random_micrm_params(N,M, 0.3) #leakage = 0.3 #hide
@named sys = micrm_system(params)
```

### Convert to `ODEProblem`

Next we convert the `ODESystem` object to an `ODEProblem`. This is the object type that is used by the `DifferentialEquations` package and contains information on the simulation such as the starting biomass, simulation duration and the structure and parameters of the system. This is done by simply calling the `ODEProblem` constructor function on our `ODESystem` object. We also need to provide the inital state of our system which is provided as a vector of `Pair{Num,Float64}`s and a tuple for the simulation duration:

```@example
using MiCRM, Random #hide
Random.seed!(1) #hide
N,M = 2,2 #set system size #hide
params = random_micrm_params(N,M, 0.3) #leakage = 0.3 #hide
@named sys = micrm_system(params) #hide
#define starting mass
u0 = fill(0.1, N+M)
u0 = [states(sys)[i] => u0[i] for i = eachindex(u0)]

tspan = (0.0, 10.0) #define tspan
#define ODEProblem (and include automatic calculation of the jacobian)
prob = ODEProblem(sys,u0,(0.0,100.0),[], jac = true)
```

### Simulate system
Finally we simualte the system using the `solve` function from `DifferentialEquations`. This can be used directly or with additional arguments specifying the solver and callbacks to be used during the simulation. This function will return a solution object which contains the trajectories of the consumers.