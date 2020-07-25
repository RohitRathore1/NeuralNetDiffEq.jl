# NeuralPDE

[![Join the chat at https://gitter.im/JuliaDiffEq/Lobby](https://badges.gitter.im/JuliaDiffEq/Lobby.svg)](https://gitter.im/JuliaDiffEq/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://travis-ci.com/SciML/NeuralPDE.jl.svg?branch=master)](https://travis-ci.com/SciML/NeuralPDE.jl)
[![Coverage Status](https://coveralls.io/repos/SciML/NeuralPDE.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/SciML/NeuralPDE.jl?branch=master)
[![codecov.io](http://codecov.io/github/SciML/NeuralPDE.jl/coverage.svg?branch=master)](http://codecov.io/github/SciML/NeuralPDE.jl?branch=master)
[![Stable](https://img.shields.io/badge/docs-stable-blue.svg)](http://neuralpde.sciml.ai/stable/)
[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](http://neuralpde.sciml.ai/dev/)

NeuralPDE.jl is a solver package which consists neural network solvers for 
partial differential equations using scientific machine learning (SciML)
techniques such as physics-informed neural networks (PINNs) and deep
BSDE solvers. This package utilizes deep neural networks and 
neural stochastic differential equations to solve high dimensional PDEs
at a greatly reduced cost and greatly increased generality compared to classical methods.

## Tutorials and Documentation

For information on using the package,
[see the stable documentation](https://neuralpde.sciml.ai/stable/). Use the
[in-development documentation](https://neuralpde.sciml.ai/dev/) for the version of
the documentation, which contains the unreleased features.

## Features

- Physics-Informed Neural Networks for automated PDE solving
- Forward-Backwards Stochastic Differential Equation (FBSDE) methods for parabolic PDEs
- Deep learning based solvers for optimal stopping time and Kolmogorov backwards equations

## Example: Solving 2D Poisson Equation via Physics-Informed Neural Networks

```julia
using NeuralPDE, ModelingToolkit

@parameters x y θ
@variables u(..)
@derivatives Dxx''~x
@derivatives Dyy''~y

# 2D PDE
eq  = Dxx(u(x,y,θ)) + Dyy(u(x,y,θ)) ~ -sin(pi*x)*sin(pi*y)

# Boundary conditions
bcs = [u(0,y) ~ 0.f0, u(1,y) ~ -sin(pi*1)*sin(pi*y),
       u(x,0) ~ 0.f0, u(x,1) ~ -sin(pi*x)*sin(pi*1)]
# Space and time domains
domains = [x ∈ IntervalDomain(0.0,1.0),
           y ∈ IntervalDomain(0.0,1.0)]
# Discretization
dx = 0.1
discretization = PhysicsInformedNN(dx)

# Neural network and optimizer
opt = Flux.ADAM(0.02)
dim = 2 # number of dimensions
chain = FastChain(FastDense(dim,16,Flux.σ),FastDense(16,16,Flux.σ),FastDense(16,1))

pde_system = PDESystem(eq,bcs,domains,[x,y],[u])
prob = discretize(pde_system,discretization)
alg = NNDE(chain,opt,autodiff=false)

phi,res  = solve(prob,alg,verbose=true, maxiters=5000)
```

And some analysis:

```julia
xs,ys = [domain.domain.lower:dx/10:domain.domain.upper for domain in domains]
analytic_sol_func(x,y) = (sin(pi*x)*sin(pi*y))/(2pi^2)

u_predict = reshape([first(phi([x,y],res.minimizer)) for x in xs for y in ys],(length(xs),length(ys)))
u_real = reshape([analytic_sol_func(x,y) for x in xs for y in ys], (length(xs),length(ys)))
diff_u = abs.(u_predict .- u_real)

using Plots
p1 = plot(xs, ys, u_real, linetype=:contourf,title = "analytic");
p2 = plot(xs, ys, u_predict, linetype=:contourf,title = "predict");
p3 = plot(xs, ys, diff_u,linetype=:contourf,title = "error");
plot(p1,p2,p3)
```

![](https://user-images.githubusercontent.com/12683885/88294196-a29d4300-cd04-11ea-8e72-18867662583d.png)

## Example: Solving a 100 Dimensional Hamilton-Jacobi-Bellman Equation

```julia
using NeuralPDE
using Flux
using DifferentialEquations
using LinearAlgebra
d = 100 # number of dimensions
X0 = fill(0.0f0, d) # initial value of stochastic control process
tspan = (0.0f0, 1.0f0)
λ = 1.0f0

g(X) = log(0.5f0 + 0.5f0 * sum(X.^2))
f(X,u,σᵀ∇u,p,t) = -λ * sum(σᵀ∇u.^2)
μ_f(X,p,t) = zero(X)  # Vector d x 1 λ
σ_f(X,p,t) = Diagonal(sqrt(2.0f0) * ones(Float32, d)) # Matrix d x d
prob = TerminalPDEProblem(g, f, μ_f, σ_f, X0, tspan)
hls = 10 + d # hidden layer size
opt = Flux.ADAM(0.01)  # optimizer
# sub-neural network approximating solutions at the desired point
u0 = Flux.Chain(Dense(d, hls, relu),
                Dense(hls, hls, relu),
                Dense(hls, 1))
# sub-neural network approximating the spatial gradients at time point
σᵀ∇u = Flux.Chain(Dense(d + 1, hls, relu),
                  Dense(hls, hls, relu),
                  Dense(hls, hls, relu),
                  Dense(hls, d))
pdealg = NNPDENS(u0, σᵀ∇u, opt=opt)
@time ans = solve(prob, pdealg, verbose=true, maxiters=100, trajectories=100,
                            alg=EM(), dt=1.2, pabstol=1f-2)
```
