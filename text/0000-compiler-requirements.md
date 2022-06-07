# High Performance, Correct Compiler Requirements

## Current State

The entire font compilation pipeline is written in Python, with some parts being written in Cython for speed.

### Current Tools

fontmake. It's the glue that binds together ufo2ft, fontTools and some other codebases to take any of the usual font formats (UFOs, Designspace + UFOs and Glyphs.app files) and produce font binaries (TTF, CFF(2); static, interpolated static, variable).

### Key Limitations

* Python is bad at embarrasingly parallel tasks like instance generation and glyph compilation due to overhead and lacking multicore facilities.
* The memory consumption of the compilation process can hinder the amount of build processes running in parallel.
* It's easy to run into performance problems with OpenType Layout table repacking, which quickly consumes a large part of the compile time.
* Codebases have grown organically by adding new things on top, some parts are byzantine and hard to change, making optimizations or rearranging code paths hard.
    * VF generation requires compiling all masters fully and then merging them, instead of generating variable data directly.
    * fontmake is underloved from a maintenance perspective, with some low hanging performance fruits continuing to hang.
    * The layering of fontmake → ufo2ft → fontTools makes changing the architecture hard. User-relevant options need to be made available and passed through all layers, e.g. GPOS compression recently.
    * Compilation of e.g. Glyphs.app files requires converting them into UFOs because ufo2ft can only compile those, which runs into impedance mismatches and is highly fragile and error-prone.

## What Do We Need?

An at least 10x faster compiler using less memory.

## Why Do We Need It?

Projects like Noto take 30+ minutes to compile completely, and a certain secret font project takes 5+ minutes to compile even with Pypy. Given how fickle font sources are and the experimentation and scripting needed to achieve certain design goals, this severly hinders incremental improvements during font design and engineering. Additionally, the resource waste is a problem for paid-for continuous integration.

## When Do We Need It?

As soon as possible.

## Implementation Requirements

* Should be divided into front- and backend. The frontend takes an input and does any necessary transformations, then presents the bare data ready to be compiled to a simple and stupid backend. This isolates format pecularities to the frontend and reduces the complexity of the backend, avoiding the ufo2ft problem.
* The compiler should be in the driving seat of the compilation process, always, instead of layering middleware on top of middleware, to avoid the fontmake layering problem.
* Should make use of parallelism where possible.
* Should use little memory.
* Should maybe be able to separate basic table and outline compilation (stage 1) from feature compilation (stage 2). Basic tables and outlines can be arrived at through interpolation or cutting them from variable fonts or producing multiple variable fonts, features can then add (subsetted) behavior to the fonts afterwards. Producing multiple fonts for a single project is common.

### Shapes of Desired Inputs to Compiler

Can take UFOs, Designspaces plus UFOs and Glyphs.app and potentially other file formats and present them directly to the compiler backend, to avoid UFO being the "bottleneck".

[how do these diverge from those required for subsetting?]