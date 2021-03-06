This is very fast stacktraces resolver for x86-64(possibly working on i386 architecture as well, but hasn't been tested on it).
This is a new major release that fixes all outstanding issues with the older implementation. It comes with a new design and implements new heuristics for even faster stacktraces generation.

There are no dependencies on any non-standard, external libraries (e.g it does not depend on binutils libBFD, which is what most other such implementations rely on). LibBFD is very powerful, but it always allocates memory and is not very fast.

For example, if you are interested in the first 4 frames and those frames belong to the same DSO, it can take as little as 400us to generate the stacktrace.

## Features Include
- High performance stacktraces generation
- Accurate resolution: inline functions source code resolution works as expected
- Very memory efficient: it does not allocate any memory on the heap, which makes it ideal for memory-constrained environments and execution contexts(see below)
- Simple to use: it is implemented in a single module(stacktraces.cpp), and all you need to use is #include "stacktraces.h" and link against stacktraces.cpp

## Memory Contraints
While the resolver does not allocate memory on the heap, if you are going to use backtrace(), you should know that that it always allocates memory.
There are alternative means to walking the stack (I mean to provide one such function soon -- it's a simple matter of looking into EBP, etc), but for now
know that if you plan to use the resolver in the context of a signal handlerr that may not allocate memory, backtrace() may be unsuitable for that purpose.
Furthermore, `abi::__cxa_demangle()` also allocates memory; an alternative will be implemented soon.

## Performance
The resolver is designed to do as little as possible. It will access as few ELF files as possible in order to return as many stack frames requested, it will use `.debug_aranges` section information if available( you should compile with ` -gdwarf-aranges` so that the compiler will emit the respective DWARF section ),
and it will group operations on a per ELF sections chunks in order to minimize CPU overhead.
You should compile stacktraces.cpp with optimizations enabled (e.g with `-Ofast`). When compiled with optimizations enabled, and if `.debug_ranges` is present in the ELF file, you can expect sub-ms resolution times even if multiple compilation units are included in the ELF sections.


## Installation
1. You need libdward include files for the various definitions/macros.

```bash
apt-get install -y libdwarf-dev
```

2. Copy stacktraces.{h,cpp} somewhere so that you can access it later. You may also want to build a library out of it(e.g `ar rcs libstacktraces.a stacktraces.o`)


## Using the Resolver
```cpp
#include "stacktraces.h"
#include <cstdio>

void fun_with_stacks() {
        // provide a buffer allocated on the stack for stacktrace()
        // you could also use e.g
        // auto stacktrace_buf = reinterpret_cast<uint8_t *>(alloca(128 * 1024));
        uint8_t             stacktrace_buf[128 * 1024];
        Switch::stack_frame frames[4];
        const auto          frames_cnt = Switch::stacktrace(frames, sizeof(frames) / sizeof(frames[0]), stacktrace_buf, sizeof(stacktrace_buf));

        if (frames_cnt < 0) {
                // check Switch::StackResolverErrors
        } else {
                for (int i{0}; i < frames_cnt; ++i) {
                        const auto &frame = frames[i];

                        if (frame.filename) {
                                // we were able to resolve the filename (or the dso/dll library path)
                                if (frame.func) {
                                        // we were able to resolve the function name
                                        if (frame.line) {
                                                // line number was also resolved
                                                std::printf(R"(%.*s at %.*s:%zu)" "\n", static_cast<int>(frame.func.size()), frame.func.data(), static_cast<int>(frame.filename.size()), frame.filename.data(), frame.line);
                                        } else {
                                                std::printf(R"(%.*s at %.*s:??)" "\n", static_cast<int>(frame.func.size()), frame.func.data(), static_cast<int>(frame.filename.size()), frame.filename.data());
                                        }
                                } else {
                                        std::printf(R"(?? at %.*s:??)" "\n", static_cast<int>(frame.filename.size()), frame.filename.data());
                                }
                        } else {
                                std::printf(R"(??)" "\n");
                        }
                }
        }
}

int main(int argc, char *argv[]) {
        fun_with_stacks();
        return 0;
}
```

```bash
clang++ -std=c++1z example.cpp stacktraces.cpp -ldl
```


Here is another example of how you could set exception handlers so that when an exception is raised and not caught, you will get a chance to display a message about it. 
```cpp
#include <cstdio>
#include <cstdlib>
#include <stdexcept>
#include <stacktraces.h>
#include <cxxabi.h>

std::terminate_handler  orig_terminate;
std::unexpected_handler orig_unexpected;

void except_handler() {
        int                 status;
        const auto          tp        = abi::__cxa_current_exception_type();
        auto                demangled = abi::__cxa_demangle(tp->name(), 0, 0, &status);
        Switch::stack_frame frames[8];
        uint8_t             stackframe_buf[32 * 1024];
        const auto frames_cnt = Switch::stacktrace(frames, sizeof(frames) / sizeof(frames[0]), 
		stackframe_buf, sizeof(stackframe_buf));

	printf("Exception of type %s thrown\n", demangled);
	std::free(demangled);

	for (int i{0}; i < frames_cnt; ++i)  {
		if (frames[i].func) {
			printf("%d: at %.*s:%zu\n", 
				i, static_cast<int>(frames[i].func.size()), frames[i].func.data(), frames[i].line);
		} else {
			printf("%d: at %.*s\n", 
				i, static_cast<int>(frames[i].filename.size()), frames[i].filename.data());
		}
	}
}

void term_handler() {
        except_handler();
        orig_terminate();
}

void unexpected_handler() {
        except_handler();
        orig_unexpected();
}

int main(int argc, char *argv[]) {
        orig_terminate  = std::set_terminate(term_handler);
        orig_unexpected = std::set_unexpected(unexpected_handler);


	// throw an exception - except_handler() will get to act on it
	// before the default handler is invoked
        throw std::range_error("Out of Range");
        return 0;
}
```

## API
There are currently two methods implemented in Switch namespace. The only difference between the overloaded `stacktrace()` is that one of the methods accepts an array of frames (that you should obtain from e.g `backtrace()`), whereas the other will invoke it for you internally. The arguments are self-explanatory, but you should make sure that storage provided is at least 32K in size(although depending on the depth of the stack, smaller buffers may suffice). You can use `alloca()` to allocate memory on the stack, if you are e.g in the execution context of a signal handler, or just allocate using e.g `uint8_t buf[128 * 1024];`, otherwise you may allocate on the heap using e.g `malloc()`.
Note that the fewer frames you request (via `stack_frames_capacity`) the faster the resolution will be, because it will likely need to access fewer DSOs and lookup the required information in those files.





