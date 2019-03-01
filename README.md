This repo is work-in-progress containing two things:

1. TensorFlow.FSharp: An F# API for TensorFlow

2. FM: A prototype of a DSL "F# for AI Models". This currently executes using TensorFlow.FSharp but could have additional backends such as DiffSharp.

# TensorFlow.FSharp

See "src" and "tests" directory. Some examples under "examples".

This component is 
* similar in scope to [TensorFlow.NET](https://github.com/SciSharp/TensorFlow.NET) and [TensorFlow for Scala](https://github.com/eaplatanios/tensorflow_scala/)
* larger in scope than [TensorFlowSharp](https://github.com/migueldeicaza/TensorFlowSharp), as it intends to include training scenarios (though not all pieces to support those scenarios are currently in place)

# FM: F# for AI Models 

This is an F# eDSL for writing numeric models. Models written in FM can be passed to 
optimization and training algorithms utilising automatic differentiation without
any change to modelling code, and can be executed on GPUs and TPUs using TensorFlow.

There is also experimental tooling for interactive tensor shape-checking, inference, tooltips and other nice things. 

This is a POC that it is possible to configure F# to be suitable for authoring AI models. We
execute them as real, full-speed TensorFlow graphs, achieving cohabitation and win-win with the TF ecosystem.
Live trajectory execution tooling can give added correctness guarantess interactively.

FM currently executes using TensorFlow.FSharp, which is why it's in this repo.

The aim of FM is to support the authoring of numeric functions and AI models - including
neural networks - in F# code. For example:

```fsharp
/// A numeric function of two parameters, returning a scalar, see
/// https://en.wikipedia.org/wiki/Gradient_descent
let f (xs: DT<double>) = 
    sin (v 0.5 * sqr xs.[0] - v 0.25 * sqr xs.[1] + v 3.0) * -cos (v 2.0 * xs.[0] + v 1.0 - exp xs.[1])
```

These functions and models can then be passed to optimization algorithms that utilise gradients, e.g.

```fsharp
// Pass this Define a numeric function of two parameters, returning a scalar
let train numSteps = GradientDescent.train f (vec [ -0.3; 0.3 ]) numSteps

let results = train 200 |> Seq.last
```

FM supports the live "trajectory" checking of key correctness properties of your numeric code,
including vector, matrix and tensor size checking, and tooling to interactively report the sizes.  To active
this tooling you need to specify a `LiveCheck` that is interactively executed by the experimental tooling
described further below.

```fsharp
[<LiveCheck>] 
let check1 = train 4 |> Seq.last 
```
When using live-checks, underlying tensors are not actually populated with data - instead only their
shapes are analyzed.  Arrays and raw numerics values are computed as normal.

Typically each model is equipped with one `LiveCheck` that instantiates the model on training data.


### Optimization algorithms utilising gradients

The aim of FM is to allow the clean description of numeric code and yet still allow this code to be
either executed using TensorFlow and - in the future - other tensor fabrics such as Torch (TorchSharp)
and DiffSharp.  These fabrics automatically compute the gradients of your models and functions with respect to
model parameters and/or function inputs.  Gradients are usually computed inside an optimization
algorithm.

For example, a naive version of Gradient Descent is shown below:

```fsharp
module GradientDescent =

    // Note, the rate in this example is constant. Many practical optimizers use variable
    // update (rate) - often reducing.
    let rate = 0.005

    // Gradient descent
    let step f xs =   
        // Get the partial derivatives of the function
        let df xs =  DT.diff f xs  
        printfn "xs = %A" xs
        let dzx = df xs 
        // Evaluate to output values 
        xs - v rate * dzx |> DT.Eval

    let train f initial steps = 
        initial |> Seq.unfold (fun pos -> Some (pos, step f pos)) |> Seq.truncate steps 
```

Here the crucial call is `DT.diff` - FM allows optimizers to derive the gradients of FM
functions and models in a way inspired by the design of `DiffSharp`. For example:

```fsharp
// Define a function which will be executed using TensorFlow
let f x = x * x + v 4.0 * x 

// Get the derivative of the function. This computes "x*2 + 4.0"
let df x = DT.diff f x  

// Run the derivative 
df (v 3.0) |> DT.RunScalar // returns 6.0 + 4.0 = 10.0
```

To differentiate a scalar function with multiple input variables:

```fsharp
// Define a function which will be executed using TensorFlow
// computes [ x1*x1*x3 + x2*x2*x2 + x3*x3*x1 + x1*x1 ]
let f (xs: DT<'T>) = sum (xs * xs * DT.Reverse xs)

// Get the partial derivatives of the scalar function
// computes [ 2*x1*x3 + x3*x3; 3*x2*x2; 2*x3*x1 + x1*x1 ]
let df xs = DT.diff f xs   

// Run the derivative 
df (vec [ 3.0; 4.0; 5.0 ]) |> DT.RunArray // returns [ 55.0; 48.0; 39.0 ]
```

### A Larger Example

Below we show fitting a linear model to training data, by differentiating a loss function w.r.t. coefficients, and optimizing
using gradient descent (200 data points generated by linear  function, 10 parameters, linear model).

```fsharp
module ModelExample =

    let modelSize = 10

    let checkSize = 5

    let trainSize = 500

    let validationSize = 100

    let rnd = Random()

    let noise eps = (rnd.NextDouble() - 0.5) * eps 

    /// The true function we use to generate the training data (also a linear model plus some noise)
    let trueCoeffs = [| for i in 1 .. modelSize -> double i |]

    let trueFunction (xs: double[]) = 
        Array.sum [| for i in 0 .. modelSize - 1 -> trueCoeffs.[i] * xs.[i]  |] + noise 0.5

    let makeData size = 
        [| for i in 1 .. size -> 
            let xs = [| for i in 0 .. modelSize - 1 -> rnd.NextDouble() |]
            xs, trueFunction xs |]
         
    /// Make the data used to symbolically check the model
    let checkData = makeData checkSize

    /// Make the training data
    let trainData = makeData trainSize

    /// Make the validation data
    let validationData = makeData validationSize
 
    let prepare data = 
        let xs, y = Array.unzip data
        let xs = batchOfVecs xs
        let y = batchOfScalars y
        (xs, y)

    /// Evaluate the model for input and coefficients
    let model (xs: DT<double>, coeffs: DT<double>) = 
        DT.Sum (xs * coeffs,axis= [| 1 |])
           
    let meanSquareError (z: DT<double>) tgt = 
        let dz = z - tgt 
        DT.Sum (dz * dz) / v (double modelSize) / v (double z.Shape.[0].Value) 

    /// The loss function for the model w.r.t. a true output
    let loss (xs, y) coeffs = 
        let y2 = model (xs, batchExtend coeffs)
        meanSquareError y y2
          
    let validation coeffs = 
        let z = loss (prepare validationData) (vec coeffs)
        z |> DT.Eval

    let train inputs steps =
        let initialCoeffs = vec [ for i in 0 .. modelSize - 1 -> rnd.NextDouble()  * double modelSize ]
        let inputs = prepare inputs
        GradientDescent.train (loss inputs) initialCoeffs steps
           
    [<LiveCheck>]
    let check1 = train checkData 1  |> Seq.last

    let learnedCoeffs = train trainData 200 |> Seq.last |> DT.toArray
         // [|1.017181246; 2.039034327; 2.968580146; 3.99544071; 4.935430581;
         //   5.988228378; 7.030374908; 8.013975714; 9.020138699; 9.98575733|]

    validation trueCoeffs

    validation learnedCoeffs
```

More examples/tests are in [dsl-live.fsx](https://github.com/fsprojects/TensorFlow.FSharp/blob/master/examples/dsl-live.fsx).

The approach scales to the complete expression of deep neural networks 
and full computation graphs. The links below show the implementation of a common DNN sample (the samples may not
yet run, this is wet paint):

* [NeuralStyleTransfer in DSL form](https://github.com/fsprojects/TensorFlow.FSharp/blob/master/examples/NeuralStyleTransfer-dsl.fsx)

The design is intended to allow alternative execution with Torch or DiffSharp.
DiffSharp may be used once Tensors are available in that library.


### Technical notes:

* `DT` stands for `differentiable tensor` and the one type of `DT<_>` values are used to represent differentiable scalars, vectors, matrices and tensors.
  If you are familiar with the design of `DiffSharp` there are similarities here: DiffSharp defines `D` (differentiable scalar), `DV` (differentiable
  vector), `DM` (differentiable matrix).

* `DT.gradients` is used to get gradients of arbitrary outputs w.r.t. arbitrary inputs

* `DT.diff` is used to differentiate of `R^n -> R` scalar-valued functions (loss functions) w.r.t. multiple input variables. If 
  a scalar input is used, a single total deriative is returned. If a vector of inputs is used, a vector of
  partial derivatives are returned.

* `DT.jacobian` is used to differentiate `R^n -> R^2` vector-valued functions w.r.t. multiple input variables. A vector or
  matrix of partial derivatives is returned.

* Other gradient-based functions include `DT.grad`, `DT.curl`, `DT.hessian` and `DT.divergence`.

* In the prototype, all gradient-based functions are implemented using TensorFlow's `AddGradients`, i.e. the C++ implementation of
  gradients. Thus not all gradient-based functions are implemented efficiently for all inputs.

* `DT.*` is a DSL for expressing differentiable tensors using the TensorFlow fundamental building blocks.  The naming
  of operators in this DSL are currently TensorFLow specific and may change.

* A preliminary pass of shape inference is performed _before_ any TensorFlow operations are performed.  This
  allows you to check the shapes of your differentiable code indepentently of TensorFlow's shape computations.
  A shape inference system is used which allows for many shapes to be inferred and is akin to F# type inference.
  Not all TensorFlow automatic shape transformations are applied during shape inference.

# The TensorFlow API for F# 

See `TensorFlow.FSharp`.  This API is designed in a similar way to `TensorFlowSharp`, but is implemented directly in F# and
contains some additional functionality.

# Live Checking Tooling for AI models

There is some tooling to do "live trajectory execution" of models and training on limited training sets,
reporting tensor sizes and performing tensor size checking.

LiveCheck for a vector addition:

<p align="center">
  <img src="https://user-images.githubusercontent.com/7204669/52524060-90eee980-2c90-11e9-9b0e-2752480dbe7d.gif" width="512"  title="LiveCheck example for vectors">
</p>


LiveCheck for a DNN:

<p align="center">
  <img src="https://user-images.githubusercontent.com/7204669/52758231-6c33a280-2fff-11e9-9098-2c47b60f71fe.gif" width="512"  title="LiveCheck example for vectors">
</p>


1. Build the VS tooling with the extensibility "hack" to allow 3rd party tools to add checking and tooltips

       git clone http://github.com/Microsoft/visualfsharp
       cd visualfsharp
       git fetch https://github.com/dsyme/visualfsharp livecheck
       git checkout livecheck
       .\build.cmd vs
       cd ..

2. Compile the extra tool
	
       git clone http://github.com/fsprojects/FSharp.Compiler.PortaCode
       dotnet build FSharp.Compiler.PortaCode

3. Start the tool and edit using experimental VS instance

       cd TensorFlow.FSharp\examples
       ..\..\FSharp.Compiler.PortaCode\FsLive.Cli\bin\Debug\net471\FsLive.Cli.exe --eval --writeinfo --watch --vshack --livechecksonly  --define:LIVECHECK dsl-live.fsx

       devenv.exe /rootsuffix RoslynDev
       (open dsl-live.fsx)

# Building

First time only:

    fsiAnyCpu setup\setup.fsx

Then:

    dotnet restore
    dotnet build
    dotnet test
    dotnet pack

# Roadmap - Core API

* Port gradient and training/optimization code (from the Scala-Tensorflow)

* Port testing for core API

* GPU execution

* TPU execution

* Add docs

# Roadmap - DSL

* Reuse graphs (deal with shape variables)

* Switch to using ported gradient code when it is available in core API

* Hand-code or generate larger TF surface area in FM DSL

* Add proper testing for DSL 

* Consider control flow translation in DSL

* Add docs

* Add examples of how to do static graph building and analysis based on FCS and quotations, e.g. for visualization

* Add DiffSharp as a backend

* Add Torch as a backend

* Performance testing

* Improve debugging for DSL through __CALLER__DEBUG__ feature for F#

# Roadmap - Live checking

* Make it into an F# Anaylzer, including adding tooltips to F# analyzer design

* Make it into a nuget you reference and start live checking from the script

# Roadmap - F# language things to consider

* Make use of `return` optional

* Consider adding `0.5T` constants (for scalars that broadcast to tensors)

* Remove `params` as a reserved keyword, possible some others too

* Consider more overloading, e.g. overloading single keyword members

* Optional and named arguments on module-defined functions

* Consider having anonymous records all implement some base indicator interface (`IFSharpRecord`) or something

* Consider making constraints `'T : (int|double|int64|single)` be exposed as a proper F# language feature

* Extension methods satisfying constraints

* Make existing constraints generalizable independently of Extension methods satisfying constraints




