

# MomentOpt.jl: Moment Optimization Library for Julia

## Upstream
Linux/MacOS: [![Build Status](https://travis-ci.org/floswald/MomentOpt.jl.svg?branch=master)](https://travis-ci.org/floswald/MomentOpt.jl)

Windows: [![Build Status](https://ci.appveyor.com/api/projects/status/github/floswald/MomentOpt.jl?branch=master&svg=true)](https://ci.appveyor.com/project/floswald/MomentOpt-jl/branch/master)

## This fork
[![Build Status](https://travis-ci.com/JulienPascal/MomentOpt.svg?token=qvUh77uHnTEg8Fi4ytAR&branch=master)](https://travis-ci.com/JulienPascal/MomentOpt)

[![Coverage Status](https://coveralls.io/repos/JulienPascal/MomentOpt.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/JulienPascal/MomentOpt.jl?branch=master)

[![codecov.io](http://codecov.io/github/JulienPascal/MomentOpt.jl/coverage.svg?branch=master)](http://codecov.io/github/JulienPascal/MomentOpt.jl?branch=master)

This package provides a `Julia` infrastructure for *[Simulated Method of Moments](http://en.wikipedia.org/wiki/Method_of_simulated_moments)* estimation, or other problems where we want to optimize a non-differentiable objective function. The setup is suitable for all kinds of **likelihood-free estimators** - in general, those require evaluating the objective at many regions. The user can supply their own algorithms for generating successive new parameter guesses. We provide a set of MCMC template algorithms. The code can be run in serial or on a cluster.


## Installation

```julia
Pkg.clone("https://github.com/floswald/MomentOpt.jl")
```

## Documentation

Still work in progres, although most of the docstrings have been written - so checkout `?MOpt.BGPChain` for example in the REPL.
I recommend to look at `src/Examples.jl`:

### Example Usage of the BGP Algorithm

Baragatti, Grimaud and Pommeret (BGP) in ["Likelihood-free parallel tempring"](http://arxiv.org/abs/1108.3423) propose an approximate Bayesian Computation (ABC) algorithm that incorporates the parallel tempering idea of Geyer (1991). We provide the BGP algorithm as a template called `MAlgoBGP`. Here we use it to run a simple toy example where we want to estimate the means of a bivariate normal distribution by using MCMC. We use 3 parallel chains, each with different temperature. The chains can exchange locations along the process if this is suitable.

**Track BGP proposals by iteration**  

We can allow for the variance of the shock to be changed adaptively. Here this is fixed to obtain a certain acceptance probability. Showing chain number 1.

![Poposals](proposals.gif)

```julia

# run this with
# using MomentOpt; MomentOpt.serialNormal()

function serialNormal()
	# data are generated from a bivariate normal
	# with mu = [a,b] = [0,0]
	# aim:
	# 1) sample [a',b'] from a space [-3,3] x [-2,2] and
	# 2) find true [a,b] by computing distance(S([a',b']), S([a,b]))
	#    and accepting/rejecting [a',b'] according to BGP
	# 3) S([a,b]) returns a summary of features of the data: 2 means

	# initial value
	pb    = OrderedDict("p1" => [0.2,-3,3] , "p2" => [-0.2,-2,2] )
	moms = DataFrame(name=["mu1","mu2"],value=[-1.0,1.0],weight=ones(2))
	mprob = MProb()
	addSampledParam!(mprob,pb)
	addMoment!(mprob,moms)
	addEvalFunc!(mprob,objfunc_norm)

	nchains = 3

	opts =Dict("N"=>nchains,
		"maxiter"=>200,
		"maxtemp"=> 5,
            # choose inital sd for each parameter p
            # such that Pr( x \in [init-b,init+b]) = 0.975
            # where b = (p[:ub]-p[:lb])*opts["coverage"] i.e. the fraction of the search interval you want to search around the initial value
		"coverage"=>0.025,  # i.e. this gives you a 95% CI about the current parameter on chain number 1.
		"sigma_update_steps"=>10,
		"sigma_adjust_by"=>0.01,
		"smpl_iters"=>1000,
		"parallel"=>false,
		"maxdists"=>[0.05 for i in 1:nchains],
		"mixprob"=>0.3,
		"acc_tuner"=>12.0,
		"animate"=>true)

	# plot slices of objective function
	s = doSlices(mprob,30)
	plot(s,:value)  # plot objective function over param values
	savefig(joinpath(dirname(@__FILE__),"../../slices-v.png"))
	plot(s,:mu1)  # plot value of moment :mu1 over param values
	savefig(joinpath(dirname(@__FILE__),"../../slices-m.png"))
	plot(s,:mu2)  # plot value of moment :mu2 over param values
	savefig(joinpath(dirname(@__FILE__),"../../slices-m2.png"))

	# setup the BGP algorithm
	MA = MAlgoBGP(mprob,opts)

	# run the estimation
	runMOpt!(MA)
	@show summary(MA)
	# summary(MA) = 3×5 DataFrames.DataFrame
	# │ Row │ id │ acc_rate │ perc_exchanged │ exchanged_most_with │ best_val    │
	# ├─────┼────┼──────────┼────────────────┼─────────────────────┼─────────────┤
	# │ 1   │ 1  │ 0.596899 │ 57.0           │ 3                   │ 0.000439814 │
	# │ 2   │ 2  │ 0.461538 │ 61.3333        │ 3                   │ 0.000355612 │
	# │ 3   │ 3  │ 0.388889 │ 58.3333        │ 2                   │ 0.000355612 │

	# BGP Algorithm with 3 BGPChains
	# ============================
	#
	# Algorithm
	# ---------
	# Current iteration: 300
	# Number of params to estimate: 2
	# Number of moments to match: 2


	histogram(MA.chains[1]);
	savefig(joinpath(dirname(@__FILE__),"../../histogram.png"))
	plot(MA.chains[1]);
	savefig(joinpath(dirname(@__FILE__),"../../lines.png"))
	return MA
end

```

### Example Output

**param histogram**  

![Histogram](histogram.png)  


**param history**  

![Lines](lines.png)  

**Slices of objective function wrt parameters**  

![Slices](slices-v.png)  

**Slices of moments wrt parameters**  

![Slices1](slices-m.png)  
![Slices2](slices-m2.png)  


## Contributing

We encourage user contributions. Please submit a pull request for any improvements you would like to suggest, or a new algorithm you implemented.

New algorithms:
* You can model your algo on the basis of `src/AlgoBGP.jl` -
* you need to implement the function `computeNextIteration!( algo )` for your `algo`
