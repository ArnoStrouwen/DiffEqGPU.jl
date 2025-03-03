# DiffEqGPU

[![Join the chat at https://julialang.zulipchat.com #sciml-bridged](https://img.shields.io/static/v1?label=Zulip&message=chat&color=9558b2&labelColor=389826)](https://julialang.zulipchat.com/#narrow/stream/279055-sciml-bridged)
[![Global Docs](https://img.shields.io/badge/docs-SciML-blue.svg)](https://docs.sciml.ai/DiffEqGPU/stable/)
[![Build status](https://badge.buildkite.com/409ab4d885030062681a444328868d2e8ad117cadc0a7e1424.svg)](https://buildkite.com/julialang/diffeqgpu-dot-jl)

[![codecov](https://codecov.io/gh/SciML/DiffEqGPU.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/SciML/DiffEqGPU.jl)

[![ColPrac: Contributor's Guide on Collaborative Practices for Community Packages](https://img.shields.io/badge/ColPrac-Contributor's%20Guide-blueviolet)](https://github.com/SciML/ColPrac)
[![SciML Code Style](https://img.shields.io/static/v1?label=code%20style&message=SciML&color=9558b2&labelColor=389826)](https://github.com/SciML/SciMLStyle)

This library is a component package of the DifferentialEquations.jl ecosystem. It includes functionality for making
use of GPUs in the differential equation solvers.

## Within-Method GPU Parallelism with Direct CuArray Usage

The native Julia libraries, including (but not limited to) OrdinaryDiffEq, StochasticDiffEq, and DelayDiffEq, are
compatible with `u0` being a `CuArray`. When this occurs, all array operations take place on the GPU, including
any implicit solves. This is independent of the DiffEqGPU library. These speedup the solution of a differential
equation which is sufficiently large or expensive. This does not require DiffEqGPU.jl.

For example, the following is a GPU-accelerated solve with `Tsit5`:

```julia
using OrdinaryDiffEq, CUDA, LinearAlgebra
u0 = cu(rand(1000))
A  = cu(randn(1000,1000))
f(du,u,p,t)  = mul!(du,A,u)
prob = ODEProblem(f,u0,(0.0f0,1.0f0)) # Float32 is better on GPUs!
sol = solve(prob,Tsit5())
```

## Parameter-Parallelism with GPU Ensemble Methods

Parameter-parallel GPU methods are provided for the case where a single solve is too cheap to benefit from
within-method parallelism, but the solution of the same structure (same `f`) is required for very many
different choices of `u0` or `p`. For these cases, DiffEqGPU exports the following ensemble algorithms:

- `EnsembleGPUArray`: Utilizes the CuArray setup to parallelize ODE solves across the GPU.
- `EnsembleGPUKernel`: Utilizes the GPU kernels to parallelize each ODE solve with their separate ODE integrator on each kernel. 
- `EnsembleCPUArray`: A test version for analyzing the overhead of the array-based parallelism setup.

For more information on using the ensemble interface, see
[the DiffEqDocs page on ensembles](https://docs.sciml.ai/dev/modules/DiffEqDocs/features/ensemble/)

For example, the following solves the Lorenz equation with 10,000 separate random parameters on the GPU:

```julia
using DiffEqGPU, OrdinaryDiffEq
function lorenz(du,u,p,t)
    du[1] = p[1]*(u[2]-u[1])
    du[2] = u[1]*(p[2]-u[3]) - u[2]
    du[3] = u[1]*u[2] - p[3]*u[3]
end

u0 = Float32[1.0;0.0;0.0]
tspan = (0.0f0,100.0f0)
p = [10.0f0,28.0f0,8/3f0]
prob = ODEProblem(lorenz,u0,tspan,p)
prob_func = (prob,i,repeat) -> remake(prob,p=rand(Float32,3).*p)
monteprob = EnsembleProblem(prob, prob_func = prob_func, safetycopy=false)
@time sol = solve(monteprob,Tsit5(),EnsembleGPUArray(),trajectories=10_000,saveat=1.0f0)
```

### EnsembleGPUKernel

The `EnsembleGPUKernel` requires a specialized ODE algorithm which is written on the GPU kernel. These implementations are part of DiffEqGPU. These implementations do not allow mutation of arrays, hence use out-of-place (OOP) `ODEProblem`.

#### Support

- `Tsit5`: The kernelized version can be called using `GPUTsit5()` with the `EnsembleProblem`. 

Taking the example above, we simulate the lorenz equation:

```julia
using DiffEqGPU, OrdinaryDiffEq, StaticArrays

function lorenz(u, p, t)
    σ = p[1]
    ρ = p[2]
    β = p[3]
    du1 = σ * (u[2] - u[1])
    du2 = u[1] * (ρ - u[3]) - u[2]
    du3 = u[1] * u[2] - β * u[3]
    return SVector{3}(du1, du2, du3)
end

u0 = @SVector [1.0f0; 0.0f0; 0.0f0]
tspan = (0.0f0, 10.0f0)
p = @SVector [10.0f0, 28.0f0, 8 / 3.0f0]
prob = ODEProblem{false}(lorenz, u0, tspan, p)
prob_func = (prob, i, repeat) -> remake(prob, p = (@SVector rand(Float32, 3)).*p)
monteprob = EnsembleProblem(prob, prob_func = prob_func, safetycopy = false)

@time sol = solve(monteprob, GPUTsit5(), EnsembleGPUKernel(), trajectories = 10_000, adaptive = false, dt = 0.1f0)

@time sol = solve(monteprob, GPUTsit5(), EnsembleGPUKernel(), trajectories = 10_000, adaptive = true, dt = 0.1f0, save_everystep = false)
```
#### Callbacks with EnsembleGPUKernel

Using callbacks with EnsembleGPUKernel methods requires their own GPU-compatible callback implementations. MWE:

```julia
using DiffEqGPU, StaticArrays, OrdinaryDiffEq
function f(u, p, t)
    du1 = -u[1]
    return SVector{1}(du1)
end

u0 = @SVector [10.0f0]
prob = ODEProblem{false}(f, u0, (0.0f0, 10.0f0))
prob_func = (prob, i, repeat) -> remake(prob, p = prob.p)
monteprob = EnsembleProblem(prob, safetycopy = false)

condition(u, t, integrator) = t == 4.0f0
affect!(integrator) = integrator.u += @SVector[10.0f0]

gpu_cb = DiscreteCallback(condition, affect!; save_positions = (false, false))

sol = solve(monteprob, GPUTsit5(), EnsembleGPUKernel(),
            trajectories = 10,
            adaptive = false, dt = 0.01f0, callback = gpu_cb, merge_callbacks = true,
            tstops = [4.0f0])
```

#### Current Support

Automated GPU parameter parallelism support is continuing to be improved, so there are currently a few limitations.
Not everything is supported yet, but most of the standard features have support, including:

- Explicit Runge-Kutta methods
- Implicit Runge-Kutta methods
- Rosenbrock methods
- DiscreteCallbacks and ContinuousCallbacks
- Multiple GPUs over clusters

#### Current Limitations

Stiff ODEs require the analytical solution of every derivative function it requires. For example,
Rosenbrock methods require the Jacobian and the gradient with respect to time, and so these two functions are required to
be given. Note that they can be generated by the
[modelingtoolkitize](https://docs.juliadiffeq.org/latest/tutorials/advanced_ode_example/#Automatic-Derivation-of-Jacobian-Functions-1)
approach. For example, 10,000 trajectories solved with Rodas5 and TRBDF2 is done via:

```julia
function lorenz_jac(J,u,p,t)
    σ = p[1]
    ρ = p[2]
    β = p[3]
    x = u[1]
    y = u[2]
    z = u[3]
    J[1,1] = -σ
    J[2,1] = ρ - z
    J[3,1] = y
    J[1,2] = σ
    J[2,2] = -1
    J[3,2] = x
    J[1,3] = 0
    J[2,3] = -x
    J[3,3] = -β
end

function lorenz_tgrad(J,u,p,t)
    nothing
end

func = ODEFunction(lorenz,jac=lorenz_jac,tgrad=lorenz_tgrad)
prob_jac = ODEProblem(func,u0,tspan,p)
monteprob_jac = EnsembleProblem(prob_jac, prob_func = prob_func)

@time solve(monteprob_jac,Rodas5(),EnsembleGPUArray(),dt=0.1,trajectories=10_000,saveat=1.0f0)
@time solve(monteprob_jac,TRBDF2(),EnsembleGPUArray(),dt=0.1,trajectories=10_000,saveat=1.0f0)
```

These limitations are not fundamental and will be eased over time.

#### Setting Up Multi-GPU

To setup a multi-GPU environment, first setup a processes such that every process
has a different GPU. For example:

```julia
# Setup processes with different CUDA devices
using Distributed
addprocs(numgpus)
import CUDAdrv, CUDAnative

let gpuworkers = asyncmap(collect(zip(workers(), CUDAdrv.devices()))) do (p, d)
  remotecall_wait(CUDAnative.device!, p, d)
  p
end
```

Then setup the calls to work with distributed processes:

```julia
@everywhere using DiffEqGPU, CuArrays, OrdinaryDiffEq, Test, Random

@everywhere begin
    function lorenz_distributed(du,u,p,t)
        du[1] = p[1]*(u[2]-u[1])
        du[2] = u[1]*(p[2]-u[3]) - u[2]
        du[3] = u[1]*u[2] - p[3]*u[3]
    end
    CuArrays.allowscalar(false)
    u0 = Float32[1.0;0.0;0.0]
    tspan = (0.0f0,100.0f0)
    p = [10.0f0,28.0f0,8/3f0]
    Random.seed!(1)
    function prob_func_distributed(prob,i,repeat)
        remake(prob,p=rand(3).*p)
    end
end
```

Now each batch will run on separate GPUs. Thus we need to use the `batch_size`
keyword argument from the Ensemble interface to ensure there are multiple batches.
Let's solve 40,000 trajectories, batching 10,000 trajectories at a time:

```julia
prob = ODEProblem(lorenz_distributed,u0,tspan,p)
monteprob = EnsembleProblem(prob, prob_func = prob_func_distributed)

@time sol2 = solve(monteprob,Tsit5(),EnsembleGPUArray(),trajectories=40_000,
                                                 batch_size=10_000,saveat=1.0f0)
```

This will `pmap` over the batches, and thus if you have 4 processes each with
a GPU, each batch of 10,000 trajectories will be run simultaneously. If you have
two processes with two GPUs, this will do two sets of 10,000 at a time.

#### Example Multi-GPU Script

In this example we know we have a 2-GPU system (1 eGPU), and we split the work
across the two by directly defining the devices on the two worker processes:

```julia
using DiffEqGPU, CuArrays, OrdinaryDiffEq, Test
CuArrays.device!(0)

using Distributed
addprocs(2)
@everywhere using DiffEqGPU, CuArrays, OrdinaryDiffEq, Test, Random

@everywhere begin
    function lorenz_distributed(du,u,p,t)
        du[1] = p[1]*(u[2]-u[1])
        du[2] = u[1]*(p[2]-u[3]) - u[2]
        du[3] = u[1]*u[2] - p[3]*u[3]
    end
    CuArrays.allowscalar(false)
    u0 = Float32[1.0;0.0;0.0]
    tspan = (0.0f0,100.0f0)
    p = [10.0f0,28.0f0,8/3f0]
    Random.seed!(1)
    pre_p_distributed = [rand(Float32,3) for i in 1:100_000]
    function prob_func_distributed(prob,i,repeat)
        remake(prob,p=pre_p_distributed[i].*p)
    end
end

@sync begin
    @spawnat 2 begin
        CuArrays.allowscalar(false)
        CuArrays.device!(0)
    end
    @spawnat 3 begin
        CuArrays.allowscalar(false)
        CuArrays.device!(1)
    end
end

CuArrays.allowscalar(false)
prob = ODEProblem(lorenz_distributed,u0,tspan,p)
monteprob = EnsembleProblem(prob, prob_func = prob_func_distributed)

@time sol = solve(monteprob,Tsit5(),EnsembleGPUArray(),trajectories=100_000,batch_size=50_000,saveat=1.0f0)
```

#### Optimal Numbers of Trajectories

There is a balance between two things for choosing the number of trajectories:

- The number of trajectories needs to be high enough that the work per kernel
  is sufficient to overcome the kernel call cost.
- More trajectories means that every trajectory will need more time steps since
  the adaptivity syncs all solves.

From our testing, the balance is found at around 10,000 trajectories being optimal.
Thus for larger sets of trajectories, use a batch size of 10,000. Of course,
benchmark for yourself on your own setup!
