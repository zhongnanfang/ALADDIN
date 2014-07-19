ALADDIN 0.1 Pre-release
============================================
Aladdin is a pre-RTL, power-performance simulator for fixed-function
accelerators. 

Please read the licence distributed with this release in the same directory as
this file. 

If you use Aladdin in your research, please cite:

Aladdin: A Pre-RTL, Power-Performance Accelerator Simulator Enabling Large
Design Space Exploration of Customized Architectures,
Yakun Sophia Shao, Brandon Reagen, Gu-Yeon Wei and David Brooks, 
International Symposium on Computer Architecture, June, 2014

============================================
Requirements:
-------------------
1. Boost Graph Library 1.55.0

   The Boost Graph Library is a header-only library and does not need to be
   built most of the time. But we do need a function that requires building
   `libboost_graph` and `libboost_regex`. To build the two libraries:

   1. Download the Boost library from here:

      `wget http://sourceforge.net/projects/boost/files/boost/1.55.0/boost_1_55_0.tar.gz`

   2. Unzip the tarball and set your `$BOOST_ROOT`: 
        ```
        tar -xzvf boost_1_55_0.tar 
        cd boost_1_55_0
        export BOOST_ROOT=/your/path/to/boost_1_55_0/
        ```
   3. Run:
        
      `./bootstrap.sh`

   4. Install:
      
      ```  
      mkdir build
      ./b2 --build-dir=./build --with-graph --with-regex
      ```

   5. It will compile these two libraries to `$BOOST_ROOT/stage/lib`

2. Recent version of GCC including some C++11 features (GCC 4.5+ should be ok).
   We use GCC 4.8.1.

3. LLVM 3.4 and Clang 3.4 64-bit

4. LLVM IR Trace Profiler (LLVM-Tracer)
   LLVM-Tracer is an LLVM compiler pass that instruments code in LLVM's 
   machine-independent IR. It prints out a dynamic trace of your program, which can
   then be taken as an input for Aladdin. 

   You can find  LLVM-Tracer here: 
     
   [https://github.com/ysshao/LLVM-Tracer.git]
     
   To build LLVM-Tracer:
     
   1. Set `LLVM_HOME` to where you installed LLVM
         
       ```
       export LLVM_HOME=/your/path/to/llvm
       export PATH=$LLVM_HOME/bin/:$PATH
       export LD_LIBRARY_PATH=$LLVM_HOME/lib/:$LD_LIBRARY_PATH
      ```
    
   2. Go to where you put LLVM-Tracer source code

       ```
       cd /path/to/LLVM-Tracer
       cd /path/to/LLVM-Tracer/full-trace
       make
       cd /path/to/LLVM-Tracer/profile-func
       make
       ```

Build:
------
1. Set `$ALDDIN_HOME` to where put Aladdin source code. 
    
      `export ALADDIN_HOME=/your/path/to/aladdin`

2. Set `$BOOST_ROOT` to where put Boost source code and update `$LD_LIBRARY_PATH`

     ```
     export BOOST_ROOT=/your/path/to/boost
     export LD_LIBRARY_PATH=$BOOST_ROOT/stage/lib:$LD_LIBRARY_PATH
     ```
     
3. Build aladdin

     ```
     cd $ALADDIN_HOME
     make
     ```

4. Add `$ALDDIN_HOME/lib` to `$LD_LIBRARY_PATH`

     `export LD_LIBRARY_PATH=$ALADDIN_HOME/lib/:$LD_LIBRARY_PATH`
    

Run:
----
After you build Aladdin and LLVM-Tracer, you can use example SHOC programs in the SHOC
directory to test Aladdin. 

Example program: triad
----------------------
  We prepare a script to automatically run all the steps above: generating
  trace, config file and run Aladdin here: 
  
  `$ALADDIN_HOME/SHOC/scripts/run_aladdin.py`

  You can run:
  
  ```
  cd $ALADDIN_HOME/SHOC/scripts
  python run_aladdin.py triad 2 2
  ```
  
  It will start Aladdin run at `$ALADDIN_HOME/SHOC/triad/sim/p2_u2`

  The last two parameters are unrolling and partition factors in the config
  files. You can sweep these parameters easily with Aladdin. 

Step-by-step:
----------------------
1. Go to `$ALADDIN_HOME/SHOC/triad`
2. Run LLVM-Tracer to generate a dynamic LLVM IR trace
   1. Both Aladdin and LLVM-Tracer track regions of interest inside a program. In the
      triad example, we want to analyze the triad kernel instead of the setup
      and initialization work done in main. To tell LLVM-Tracer the functions we are
      interested, set enviroment variable `WORKLOAD` to be the function names (if you 
      have multiple functions interested, separated by comma):

    ```
    export WORKLOAD=triad
    export WORKLOAD=md,md_kernel
    ```
        
   2. Generate LLVM IR:

    `clang -g -O1 -S -fno-slp-vectorize -fno-vectorize -fno-unroll-loops -fno-inline -emit-llvm -o triad.llvm triad.c`
     
   3. Run LLVM-Tracer pass:
      Before you run, make sure you already built LLVM-Tracer. 
      Set `$TRACER_HOME` to where you put LLVM-Tracer code.
        
     ```
     export TRACER_HOME=/your/path/to/LLVM-Tracer
     opt -S -load=$TRACER_HOME/full-trace/full_trace.so -full trace triad.llvm -o triad-opt.llvm
     llvm-link -o full.llvm triad-opt.llvm $TRACER_HOME/profile-func/tracer_logger.llvm
     ```
     
   4. Generate machine code:
        
     ```
     llc -filetype=asm -o ful.s full.llvm
     gcc -fno-inline -o triad-instrumented full.s
     ```
     
   5. Run binary:
        
     `./triad-instrumented`
        
     It will generate a file called `dynamic_trace' under current directory. 
     We provide a python script to run the above steps automatically for SHOC. 
        
     ```
     cd $ALADDIN_HOME/SHOC/scripts/
     python llvm_compile.py $ALADDIN_HOME/SHOC/triad triad
     ```
  
3.config file

  Aladdin takes user defined parameters to model corresponding accelerator
  designs. We prepare an example of such config file at 
  ```
  cd $ALADDIN_HOME/SHOC/triad/example
  cat config_example
  ```
  ```
  partition,cyclic,a,2048,2  //cyclic partition array a, size 2048, with partition factor 2
  partition,cyclic,b,2048,2  //cyclic partition array b, size 2048, with partition factor 2
  partition,cyclic,c,2048,2  //cyclic partition array c, size 2048, with partition factor 2
  unrolling,triad,5,2        //unroll loop in triad, define at line 5 in triad.c, with unrolling factor 2
  ```

  The format of config file is:
     
  ```
  partition,cyclic,array_name,array_size,partition_factor
  unrolling,function_name,loop_increment_happened_at_line,unrolling_factor
  ```
     
  Two more configs:
     
  ```
  partition,complete,array_name,array_size //convert the array into register
  flatten,function_name,loop_increment_happend_at_line  //flatten the loop
  ```
     
4. Run Aladdin
   Aladdin takes three parameters: 
   a. benchmark name
   b. path to the dynamic trace generated by LLVM-Tracer
   c. config file
   Now you are ready to run Aladdin by:
     
   ```
   cd $ALADDIN_HOME/SHOC/triad/example
   $ALADDIN_HOME/aladdin triad $ALADDIN_HOME/SHOC/triad/dynamic_trace config_example
   ```

   Aladdin will print out the different stages of optimizations and scheduling as
   it runs. In the end, Aladdin prints out the performance, power and area
   estimates for this design, which is also saved at <bench_name>_summary
   (triad_summary) in this case. 

   Aladdin will generate some files during its execution. One file you might
   be interested is 
    `<bench_name>_stats`
   which profiles the dynamic activities as accelerator is running. Its format:
     
   ```
   line1: cycles,<cycle count>,<# of nodes in the trace>
   line2: <cycle count>,<function-name-mul>,<function-name-add>,<each-partitioned-array>,....
   line3: <cycle 0>,<# of mul happend from functiona-name at cycle 0>,..
   line4: ...
   ```
     
    A corresponding dynamic power trace is 
      `<bench_name>_stats_power`
  
============================================
Sophia Shao,

shao@eecs.harvard.edu

Harvard University, 2014
