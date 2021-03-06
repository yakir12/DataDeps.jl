# DataDeps
Master: [![Build Status](https://travis-ci.org/oxinabox/DataDeps.jl.svg?branch=master)](https://travis-ci.org/oxinabox/DataDeps.jl)

## What is DataDeps?
DataDeps is a package for simplifying the management of data in your julia application.
In particular it is designed to make getting static data from some server into the local machine,
and making programs know where that data is trivial.

For some examples of it being useful and cool see [this blog post](http://white.ucc.asn.au/2018/01/18/DataDeps.jl-Repeatabled-Data-Setup-for-Repeatable-Science.html)

### Why not store the data in Git?
Git is good for files that meet 3 requirements:

 - Plain text (not binary)
 - Smallish (Github will not accept files >50Mb in size)
 - Dynamic (Git is version control, it is good at knowing about changes)

There is certainly some room around the edges for this, like sorting a few image in repository is OK, but storing all of ImageNet is a no go.

DataDeps.jl is good for:

 - Any file format
 - Any size
 - Static (that is to say it doesn't change)
 
The main use case is downloading large dataset for machine learning, and corpora for NLP.
In this case the data is not even normally yours to begin with.
It lives on some website somewhere.

#### But my data is dynamic
Well how dynamic?
If you are willing to tag a new relase of your package each time the data changes, then maybe this is no worry, but maybe it is.

But the real question is are you managing your data properly in the first place.
DataDeps.jl does not provide for versioning of data -- you can't force users to download new copies of your data using DataDeps.
There are work arounds, such as using DataDeps.jl + `deps/build.jl` to `rm(datadep"MyData", recursive=true, force=true` every package update. Or considering each version of the data as a different datadep with a different name.

But maybe DataDeps isn't the solution for you.
DataDeps.jl may form part of your overall solution or it may not.
That is a discussion to have on Slack or Discourse maybe. Or in the issues for this repo.
See also the list of related packages at the bottom

The other option is that if your data a good fit for git (being in the overlapping area of plaintext & small (or close enough to those things)),
then you could add it as a `ManualDataDep` in `deps/data/MyData`.


## Usage for package developers

### Registering a DataDep
A DataDeps registration is a block of code delaring a dependency.
You should put it somewhere that it will be executed before any other code in your script that depends on that data.
In most cases it is best to put it inside the  [modules's `__init__()` function](https://docs.julialang.org/en/stable/manual/modules/#Module-initialization-and-precompilation-1).

It is pretty flexible.
Perhaps easiest is to look at the [examples](test/examples.jl).


The basic Registration block looks like: (Type parameters are shown below are a simplifaction)
```
RegisterDataDep(
    name::String, 
    message::String,
    remote_path::Union{String,Vector{String}...},
    [checksum::Union{String,Vector{String}...},]; # Optional
    # keyword args:
    fetch_method=download # (remote_fikepath, local_filepath)->Any 
    post_fetch_method=download # (local_filepath)->Any
)
```
This is the bare minium to setup a datadep.

 - *Name**: the name used to refer to this datadep, coresponds to a folder name where it will be stored
    - It can have spaces or any other character that is allowed in a Windows filestring (which is a strict subset of the restriction for unix filenames).
 - *Message*: A message displayed to the user for they are asked if they want to downloaded it.
    - This is normally used to give a link to the original source of the data, a paper to be cited etc.
 - *remote_path*: where to fetch the data from. Normally a string or strings) containing an URL
    - This is usually a string, or a vector of strings (or a vector of vector... see [Recursive Structure](Recursive Structure) below)
    

 - *checksum* this is very flexible, it is used to check the files downloaded correctly
    - By far the most common use is to just provide a SHA256 sum as a hex-string for the files
    - If not provided, then a warning message with the  SHA256 sum is displayed; this is to help package devs workout the sum for there files.
    - If you want to use a different hashing algorithm, then you can provide a tuple `(hashfun, targethex)`
        - `hashfun` should be a function which takes an IOStream, and returns a `Vector{UInt8}`. 
	      - Such as any of the functions from [SHA.jl](https://github.com/staticfloat/SHA.jl), eg `sha3_384`, `sha1_512`
	      - or `md5` from [MD5.jl](https://github.com/oxinabox/MD5.jl)
  - If you want to use a different hashing algorithm, but don't know the sum, you can provide just the `hashfun` and a warning message will be displayed, giving the correct tuple of `(hashfun, targethex)` that should be added to the registration block.

	- If you don't want to provide a checksum,  because your data can change pass in the type `Any`. (But see above warnings about "what if my data is dynamic")
    - Can take a vector of checksums, being one for each file, or a single checksum in which case the per file hashes are `xor`ed to get the target hash. (See [Recursive Structure](Recursive Structure) below)


 -  `fetch_method=download` a function to run to download the files.
    - Function should take 2 parameters (remote_fikepath, local_filepath), and can return anything
    - Defaults to `Base.download` which invokes commandline download tools. 
    - Can take a vector of methods, being one for each file, or a single method, in which case that method is used to download all of them. (See [Recursive Structure](Recursive Structure) below)
    
 - `post_fetch_method` a function to run after the files have download
    - Should take the local filepath as its first and only argument. Can return anything.
    - Default is to do nothing.
    - Can do what it wants from there, but most likes wants to extract the file into the data directory.
    - towards this end DataDeps includes a command: `unpack` which will extract an compressed folder, deleting the original.
    - It should be noted that it `post_fetch_method` runs from within the data directory
       - which means operations that just write to the current working directory (like `rm` or `mv` or ```run(`SOMECMD`))``` just work.
       - You can call `cwd()` to get the the data directory for your own functions. (Or `dirname(local_filepath)`)
    - Can take a vector of methods, being one for each file, or a single method, in which case that ame method is applied to all of the files. (See [Recursive Structure](Recursive Structure) below)
    
    
##### Recursive Structure
`fetch_method`, `post_fetch_method` and `checksum` all can match the structure of `remote_path`.
If `remote_path` is just an single path, then they each must be single items.
If `remote_path` is a vector, then if those properties are a vector (which must be the same length) then they are applied each to the corresponding element; or if not then it is applied to all of them.
This means you can for example provide check-sums per file, or per-the-all.
It also means you can specify different `post_fetch_methods` for different files, e.g. doing nothing to some, and extracting others.
Further more this applies recursively.

For example:
```
RegisterDataDep("eg", "eg message",
    ["http//example.com/text.txt", "http//example.com/sub1.zip", "http//example.com/sub2.zip"]
    post_fetch_method = [identity, file->run(`unzip $file`), file->run(`unzip $file`)]
)
```
So `identity`  (i.e. nothing) will be done to the first paths resulting file, and the second and third will be unzipped.

can also be written:
```
RegisterDataDep("eg", "eg message",
    ["http//example.com/text.txt", ["http//example.com/sub1.zip", "http//example.com/sub2.zip"]]
    post_fetch_method = [identity, file->run(`unzip $file`)]
)
```
The unzip will be applied to both elements in the child array



#### ManualDataDep
ManualDataDeps are datadeps that have to be managed by some means outside of DataDeps.jl,
but DataDeps.jl will still provide the convient `datadep"MyData"` string macro for finding them.
As mentions above, if you put the data in your git repo for your package under `deps/data/NAME` then it will be managed by julia package manager.

A manaul DataDep registration is just like a normal `DataDep` registration,
except that only a `name` and `message` are provided. 

Inside the message you should give instructions on how to acquire the data.


### Using a datadep string to get hold of the data.
For any registered DataDep, `datadep"Name"` returns a path to a folder containing the data.
If when that string macro is evaluated no such folder exists, then DataDeps will swing into action and coordiante the acquisition of the data into a folder, then return the path that now contains the data.

You can also use `datadep"Name/subfolder/file.txt"` (and similar) to get a path to the file at  `subfolder/file.txt` within the data directory for `Name`.
Just like when using the plain `datadep"Name"` this will if required downloadload the whole datadep (**not** just the file).
However, it will also engage additional features to verify that that file exists (and is readable),
and if not will attempt to help the user resolve the situation.
This is useful if files may have been deleted by mistake, or if a ManualDataDep might have been incorrectly installed.




### Installing Data Lazily.
Most packages using more than one data source, will want to download them only when the user requires them.
That is to say if the user never calls a function that requires that data, then the data should not be downloaded.

The basic way is to hide the datadep in some code not being evaluated except on a condition.
For example, say some webcam security system can be run in training mode, in which case data should be used from the datadep,
or in deployment mode, in which case the data should be read from the webcam's folder:

```
data_folder = training_mode ? datadep"SecurityFootage" : "/srv/webcam/today"       
```
The data will not be downloaded if `training_mode==false`, because the referred to folder is never required.
Of-course if the data was already downloaded, then it wouldn't be downloaded again either way.

Another example of a particularly nice way of doing this is to use the datadep string as the default value for a function paramater
`function foo(path=datadep"FooData")`.
If the user passses a value when they call `foo` then the datadep string will never be evaluated.
So the data will not be sourced via DataDeps.jl


#### Installing Data Eagerly
If you want the data to be installed when the package is first loaded,
just put the datadep string `datadep"Name"` anywhere it will immediately run.
For example, in the `__init__` function immediately after the registration block.

If you want it to be installed at `Pkg.build` time.
The best thing to do is to put your data dep registration block in a file (eg `src/dataregistrations.jl`),
and then `include` it into `deps/build.jl` followed by putting in the datadep string somewhere at global scope.
(Including would be done by `include(pathjoin(@__DIR__,"..","src","dataregistrations.jl"`).
One would also `include` that registrations file into the `__init__` function in the  main source of the package as well.


### DataDepsGenerators
[DataDepsGenerators.jl](https://github.com/oxinabox/DataDepsGenerators.jl) is a julia package to help generate DataDeps registration blocks from well known data sources.
It attempts to use webscraping and such to workout what should be in the registration block.


### Assuming direct control
For more hardcore devs customising the user experience,
and people needing to do debugging you may assume (nearly) full control over the download operation
by directly invoking the method `Base.download(::DataDep, localpath; kwargs...)`.
It is fully documented in its docstring.


### Extending DataDeps with custom `AbstractDataDep` types
One of the intended point of extension for DataDeps.jl is in developers defining their own DataDep types.
99% of developers won't need to do this, a `ManualDataDep` or a normal (automatic) `DataDep` covers most use cases.
However, if for example you want to have a DataDep that after the download is complete and after the `post_fetch_method` is run, does an additional validation, or some data synthesis step that requires working with multiple of the files simultaneously (which `post_fetch_method` can not do), or a `SemiManualDataDep` where the user does some things and then other things happen automatically, then this can be done by creating your own `AbstractDataDep` type.

The code for `ManualDataDep` is a good place to start looking to see how that is done.
You can also encapsulate an `DataDep` as one of the elements of your new type.

If you do this you might like to contribute the type back up to this repository, so others can use it also.




## Usage for users
The mail goal of DataDeps.jl is to simplify life for the user.
They should just plain not be thinking about the data the package needs so much anymore.

### Moving Data
Moving data is a great idea.
DataDeps.jl is infavour of moving data
When data is automatically downloaded it will almost always go to the same location.
The first (existant, writable) directory on your `DATADEPS_LOADPATH`.
Which by-default is `~/.julia/datadeps/`.
But you can move them from there to anywhere in the `DATADEPS_LOADPATH`.
If you have some data that everyone in your lab is using, and it is like 200GB large,
you probably want to shift it to a shared area, like `/usr/share/datadeps`.
Even if you don't have write permission, you can have a sysadmin move it, and so long as you still have read permission DataDeps.jl will find it and use it for you.


### Having multiple copies of the same DataDir
You probably don't want to have multiple copies of a DataDir with the same name.
DataDeps.jl tried to handle it as gracefuly as it can.
But having different DataDep under the same name, is probably going to lead to packages loading the wrong one.
Except if they are (both) located in their packages `deps/data` folder.

By moving a package's data dependency into its package directory under `deps/data`, it becomes invisible except to that package.
For example `~/.julia/v0.6/EXAMPLEPKG/deps/data/EXAMPLEDATADEP/`,
for the package `EXAMPLEPKG`, and the datadep `EXAMPLEDATADEP`.

Ideally though you should probably raise an issue with the package maintainers and see if one (or both) of them want to change the DataDep name.

Note also when it comes to file level loading, e.g. `datadep"Name/subfolder/file.txt"`,
DataDeps does not check all folders with that `Name` (if you do have multiple).
If the file is not in the first folder it finds you will be presented with the recovery dialog.
From which the easiest option is to select the option to delete the folder and retry,
since that will result in it checked the second folder (as the first will not exist).



## Configuration

Currently configuration is done via Enviroment Variables.
It is likely to stay that way, as they are also easy to setup in CI tools.
You can set these in the `.juliarc` file using the `ENV` dictionary if you don't want to mess up your `.profile`.
However, you shouldn't need to.
DataDeps.jl tries to have very sensible defaults.

 - `DATADEPS_ALWAY_ACCEPT` -- bypasses the confirmation before downloading data. Set to `true` (or similar string)
    - This is provided for scripting (in particular CI) use
    - Note that it remains your responsibility to understand and read any terms of the data use (this is remains true even if you don't turn on this bypass)
	- default `false`
 - `DATADEPS_LOAD_PATH` -- The list of paths, other than the package directory (`PKGNAME/deps/data`) to save and load data from
    - default values is complex. On all systems it includes the equivalent of `~/.julia/datadeps`. It also includes a large number of other locations such as `/usr/share/datadeps` on linux, and `C:/ProgramData` on Windows.
 - `DATADEPS_DISABLE_DOWNLOAD` -- causes any action that would result in the download being triggered to throw an exception.
   - useful e.g. if you are in an environment with metered data, where your datasets should have already been downloaded earlier, and if there were not you want to respond to the situation rather than let DataDeps download them for you.
   - default `false`

## Other similar packages:
DataDeps.jl isn't the answer to all your download needs.
It is focused squarely on static data.
It might not be good for your use case.

Alternatives that I am aware of are:

 - [RemoteFiles.jl](https://github.com/helgee/RemoteFiles.jl): keeps local files up to date with remotes
 - [BinaryProvider.jl](https://github.com/JuliaPackaging/BinaryProvider.jl) downloads binaries intended as part of a build chain.
