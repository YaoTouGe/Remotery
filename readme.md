Remotery
--------

[![Build](https://github.com/Celtoys/Remotery/actions/workflows/build.yml/badge.svg?branch=main)](https://github.com/Celtoys/Remotery/actions/workflows/build.yml)

A realtime CPU/GPU profiler hosted in a single C file with a viewer that runs in a web browser.

![RemoteryNew](https://github.com/Celtoys/Remotery/assets/1532903/bc5117f6-0f1e-438c-a096-67c8ff7747e7)


Features:

* Lightweight instrumentation of multiple threads running on the CPU and GPU.
* Web viewer that runs in Chrome, Firefox and Safari; on Desktops, Mobiles or Tablets.
* GPU UI rendering, bypassing the DOM completely, for real-time 60hz viewer updates at 10,000x the performance.
* Automatic thread sampler that tells you what processor cores your threads are running on without requiring Administrator privileges.
* Drop saved traces onto the Remotery window to load historical runs for inspection.
* Console output for logging text.
* Console input for sending commands to your game.
* A Property API for recording named/typed values over time, alongside samples.
* Profiles itself and shows how it's performing in the viewer.

Supported Profiling Platforms:

* Windows 7/8/10/11/UWP (Hololens), Linux, OSX, iOS, Android, Xbox One/Series, Free BSD.

Supported GPU Profiling APIS:

* D3D 11/12, OpenGL, CUDA, Metal.

Compiling
---------

* Windows (MSVC) - add lib/Remotery.c and lib/Remotery.h to your program. Set include
  directories to add Remotery/lib path. The required libraries (ws2_32.lib and winmm.lib) should be picked
  up through the use of the `#pragma comment` directives in Remotery.c.

* Windows (MINGW-64) - add lib/Remotery.c and lib/Remotery.h to your program. Set include
  directories to add Remotery/lib path. You will need to link libws2_32.a and libwinmm.a yourself through your build system, as GCC (and therefore MINGW-64) do not support `#pragma comment` directives

* Mac OS X (XCode) - simply add lib/Remotery.c, lib/Remotery.h and lib/Remotery.mm to your program.

* Linux (GCC) - add the source in lib folder. Compilation of the code requires -pthreads for
  library linkage. For example to compile the same run: cc lib/Remotery.c sample/sample.c
  -I lib -pthread -lm

* FreeBSD - the easiest way is to take a look at the official port
  ([devel/remotery](https://www.freshports.org/devel/remotery/)) and modify the port's
  Makefile if needed. There is also a package available via `pkg install remotery`.

You can define some extra macros to modify what features are compiled into Remotery:

    Macro               Default     Description

    RMT_ENABLED         1           Disable this to not include any bits of Remotery in your build
    RMT_USE_TINYCRT     0           Used by the Celtoys TinyCRT library (not released yet)
    RMT_USE_CUDA        0           Assuming CUDA headers/libs are setup, allow CUDA profiling
    RMT_USE_D3D11       0           Assuming Direct3D 11 headers/libs are setup, allow D3D11 GPU profiling
    RMT_USE_D3D12       0           Allow D3D12 GPU profiling
    RMT_USE_OPENGL      0           Allow OpenGL GPU profiling (dynamically links OpenGL libraries on available platforms)
    RMT_USE_METAL       0           Allow Metal profiling of command buffers


Basic Use
---------

See the sample directory for further examples. A quick example:

    int main()
    {
        // Create the main instance of Remotery.
        // You need only do this once per program.
        Remotery* rmt;
        rmt_CreateGlobalInstance(&rmt);

        // Explicit begin/end for C
        {
            rmt_BeginCPUSample(LogText, 0);
            rmt_LogText("Time me, please!");
            rmt_EndCPUSample();
        }

        // Scoped begin/end for C++
        {
            rmt_ScopedCPUSample(LogText, 0);
            rmt_LogText("Time me, too!");
        }

        // Destroy the main instance of Remotery.
        rmt_DestroyGlobalInstance(rmt);
    }


Running the Viewer
------------------

Double-click or launch `vis/index.html` from the browser.


Sampling CUDA GPU activity
--------------------------

Remotery allows for profiling multiple threads of CUDA execution using different asynchronous streams
that must all share the same context. After initialising both Remotery and CUDA you need to bind the
two together using the call:

    rmtCUDABind bind;
    bind.context = m_Context;
    bind.CtxSetCurrent = &cuCtxSetCurrent;
    bind.CtxGetCurrent = &cuCtxGetCurrent;
    bind.EventCreate = &cuEventCreate;
    bind.EventDestroy = &cuEventDestroy;
    bind.EventRecord = &cuEventRecord;
    bind.EventQuery = &cuEventQuery;
    bind.EventElapsedTime = &cuEventElapsedTime;
    rmt_BindCUDA(&bind);

Explicitly pointing to the CUDA interface allows Remotery to be included anywhere in your project without
need for you to link with the required CUDA libraries. After the bind completes you can safely sample any
CUDA activity:

    CUstream stream;

    // Explicit begin/end for C
    {
        rmt_BeginCUDASample(UnscopedSample, stream);
        // ... CUDA code ...
        rmt_EndCUDASample(stream);
    }

    // Scoped begin/end for C++
    {
        rmt_ScopedCUDASample(ScopedSample, stream);
        // ... CUDA code ...
    }

Remotery supports only one context for all threads and will use cuCtxGetCurrent and cuCtxSetCurrent to
ensure the current thread has the context you specify in rmtCUDABind.context.


Sampling Direct3D 11 GPU activity
---------------------------------

Remotery allows sampling of D3D11 GPU activity on multiple devices on multiple threads. After initialising Remotery, you need to bind it to D3D11 with a single call from the thread that owns the device context:

    // Parameters are ID3D11Device* and ID3D11DeviceContext*
    rmt_BindD3D11(d3d11_device, d3d11_context);

Sampling is then a simple case of:

    // Explicit begin/end for C
    {
        rmt_BeginD3D11Sample(UnscopedSample);
        // ... D3D code ...
        rmt_EndD3D11Sample();
    }

    // Scoped begin/end for C++
    {
        rmt_ScopedD3D11Sample(ScopedSample);
        // ... D3D code ...
    }

Subsequent sampling calls from the same thread will use that device/context combination. When you shutdown your D3D11 device and context, ensure you notify Remotery before shutting down Remotery itself:

    rmt_UnbindD3D11();


Sampling OpenGL GPU activity
----------------------------

Remotery allows sampling of GPU activity on your main OpenGL context. After initialising Remotery, you need
to bind it to OpenGL with the single call:

    rmt_BindOpenGL();

Sampling is then a simple case of:

    // Explicit begin/end for C
    {
        rmt_BeginOpenGLSample(UnscopedSample);
        // ... OpenGL code ...
        rmt_EndOpenGLSample();
    }

    // Scoped begin/end for C++
    {
        rmt_ScopedOpenGLSample(ScopedSample);
        // ... OpenGL code ...
    }

Support for multiple contexts can be added pretty easily if there is demand for the feature. When you shutdown
your OpenGL device and context, ensure you notify Remotery before shutting down Remotery itself:

    rmt_UnbindOpenGL();


Sampling Metal GPU activity
---------------------------

Remotery can sample Metal command buffers issued to the GPU from multiple threads. As the Metal API does not
support finer grained profiling, samples will return only the timing of the bound command buffer, irrespective
of how many you issue. As such, make sure you bind and sample the command buffer for each call site:

    rmt_BindMetal(mtl_command_buffer);
    rmt_ScopedMetalSample(command_buffer_name);

The C API supports begin/end also:

    rmt_BindMetal(mtl_command_buffer);
    rmt_BeginMetalSample(command_buffer_name);
    ...
    rmt_EndMetalSample();


Applying Configuration Settings
-------------------------------

Before creating your Remotery instance, you can configure its behaviour by retrieving its settings object:

    rmtSettings* settings = rmt_Settings();

Some important settings are:

    // Redirect any Remotery allocations to your own malloc/free, with an additional context pointer
    // that gets passed to your callbacks.
    settings->malloc;
    settings->free;
    settings->mm_context;

    // Specify an input handler that receives text input from the Remotery console, with an additional
    // context pointer that gets passed to your callback.
    // The handler will be called from the Remotery thread so synchronization with a mutex or atomics
    // might be needed to avoid race conditions with your threads.
    settings->input_handler;
    settings->input_handler_context;
