# Overview

* minimalist
* clean code
* KISS (Keep It Simple Stupid)


## coding style

 * UNIX-like, formatted by clang-format (TODO)
 * indent formatted by [.editorconfig][1]


## fatcore.lib

Minimalist foundation library

* allocator (allocator.h, temp_allocator.h)
* api_registry
* data structures (carray.inl, slab.inl, ...)
* log
* os (platform-specific: file_io, file_system, threading, ...)
* test framework (unit_test.h, integration_test.h)


About FAT_LINKS_CORE

* use it when linking .exe

```
#if FAT_LINKS_CORE
extern struct fat_allocator_api* fat_allocator_api;
#endif

cl.exe /DFAT_LINKS_CORE=1
```


## including rule of headers

* .h - only include "api_types.h"
* .inl - can include other .


## platform macros

OS Macros

* before

```C
/* api_types.h */
#if defined(_WIN64) || defined(_WIN32)
#  define FAT_OS_WINDOWS
#elif defined(__ANDROID__)
#  define FAT_OS_ANDROID
#elif defined(__linux__)
#  define FAT_OS_LINUX
#else
#  error Unsupported Operating System
#endif
```

* after, OS macros defined by Makefile(emake/vs proj/gradle)

Compiler Macros

* before

```C
#if defined(_WIN64) || defined(_WIN32)
#  define FAT_COMPILER_MSVC
#elif defined(__clang__)
#  define FAT_COMPILER_CLANG
#elif defined(__GNUC__)
#  define FAT_COMPILER_GCC
#else
#  error Unsupported Compiler
#endif
```

* after, we use _MSC_VER directly, make macros.h doesn't depends on api_types.h

```C
/* fatcore/macros.h */
#if defined(_MSC_VER)
#  define fat_or(a, b) ((a) ? (a) : (b))
#else
#  define fat_or(a, b) ((a) ?: (b))
#endif
```


## allocator

### normal allocator

* fatcore/allocator.h
* extremely simple interface (only realloc())

```C
typedef struct fat_allocator_i
{
    ...
    void *(*realloc)(struct fat_allocator_i* a, void* ptr, uint64_t old_size, uint64_t new_size,
        const char* file, uint32_t line);
} fat_allocator_i;

// Convenience macro for allocating memory using `fat_allocator_i`.
#define fat_alloc(a, sz)                  (a)->realloc(a, 0, 0, sz, __FILE__, __LINE__)
#define fat_alloc_at(a, sz, file, line)   (a)->realloc(a, 0, 0, sz, file, line)

// Convenience macro for freeing memory using `fat_allocator_i`.
#define fat_free(a, p, sz)                (a)->realloc(a, p, sz, 0, __FILE__, __LINE__)

// Convenience macro for reallocating memory using `fat_allocator_i`.
#define fat_realloc(a, p, old_sz, new_sz) (a)->realloc(a, p, old_sz, new_sz, __FILE__, __LINE__)
```

Usage

```C
fat_allocator_i *a = fat_allocator_api->system;

uint8_t* text = fat_alloc(a, size);
fat_free(a, text, size);
```

### temp allocator

* allocate 1024 bytes from stack (fast allocate & free)
* using backing_allocator when exceeding 1024 bytes

```C
{
    FAT_INIT_TEMP_ALLOCATOR(ta);

    void* p = fat_temp_alloc(ta, 16);
    memset(p, 1, 16);

    char* msg = fat_temp_allocator_api->printf(ta, "%s.%s.%d\n", "fat", "hello", 10);
    fat_logger_api->print(FAT_LOG_TYPE_INFO, msg);

    FAT_SHUTDOWN_TEMP_ALLOCATOR(ta);
}
```

## api registry

* exe(host) <==> .dll(plugins / business-logic)

```
+---------------------+ - - - - - - - - - 
| simple_3d.dll       | business logic
+---------------------+ - - - - - - - - -
| fat_os_windows.dll  |
| fat_renderer.dll    | plugins
| ...                 |
+---------------------+ - - - - - - - - - 
| simple_3d.exe       |
| - - - - - - - - - - | host 
| fatcore.lib         |
|   * allocator       |
|   * logger          |
|   * ...             |
+---------------------+ - - - - - - - - -
```

```C
host.c
host.inl
simple_3d.c
simple_3d.h
```

EXE

```
/* host.c */
const char *main_dll = "simple-3d-dll";
#include "host.inl"

/* host.inl */
int run(int argc, char *argv[])
{
    ...

    fat_allocator_i allocator = fat_allocator_api->create_child(fat_allocator_api->system, "host");
    fat_init_global_api_registry(&allocator);
    fat_register_all_core_apis(fat_global_api_registry);

    ...

    fat_shutdown_global_api_registry(&allocator);
    fat_allocator_api->destroy_child(&allocator);

    fat_memory_tracker_api->check_for_leaked_scopes();

    return 0;
}
```

DLL

```C
/* fatcore/allocator.h */
#define FAT_ALLOCATOR_API_NAME "fat_allocator_api"

/* fatcore/buffer_format.h */
#define FAT_BUFFER_FORMAT_API_NAME "fat_buffer_format_api"

/* simple_3d.c */
struct fat_api_registry_api *fat_global_api_registry;

struct fat_allocator_api *fat_allocator_api;
struct fat_buffer_format_api *fat_buffer_format_api;

FAT_DLL_EXPORT void fat_load_plugin(struct fat_api_registry_api *reg, bool load)
{
    fat_global_api_registry = reg;

    // Get core apis
    fat_allocator_api = reg->get(FAT_ALLOCATOR_API_NAME);
    fat_buffer_format_api = reg->get(FAT_BUFFER_FORMAT_API_NAME);
    ...
}
```

## api & interface

 * api - singleton
 * interface - multiple instances

```C
/* fatcore/unit_test.h */
#define FAT_UNIT_TEST_INTERFACE_NAME "fat_unit_test_i"

typedef struct fat_unit_test_i
{
    void (*test)();
} fat_unit_test_i;

/* unit_test.c */
int main(int argc, char* argv[])
{
    ...

    uint32_t count;
    /* array */ fat_unit_test_i* unittests = fat_global_api_registry->implementations(
        FAT_UNIT_TEST_INTERFACE_NAME, &count);
    for (uint32_t i = 0; i < FAT_ARRAY_END(unittests); ++i)
        unittests[i]->test();

    ...
}
```

[1]:https://editorconfig.org/
