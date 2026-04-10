---
layout: post
title: "Simple DLL Injection via the Windows API"
---
All of the code in this article can be found [here](https://github.com/fdh911/ez_dll_injector).

I've recently started learning more about the Windows API and gained a surface level understanding of virtual memory, threads, modules, etc. As such, I wanted to apply this newfound knowledge by building a basic DLL injector, which makes use of some simple WinAPI functionality.

Prerequisites: You're expected to be familiar with the basics of programming in C++ and have a vague idea of how the WinAPI works.

# The final goal

We want to be able to load a DLL of our choice into any process (Not accounting for security measures right now); a reason why we might want to do this is to read and modify a game's memory at runtime.

# Brainstorming

Since we have access to the Windows API, the idea is as follows: We're going to make the target process call the LoadLibrary function on our behalf.

To be more specific: we will be creating a remote thread that runs inside of the process' address space. The thread will have to call the following function on startup:

```cpp
LoadLibraryA("c:/path/to/dll");
```

Notice the string literal in the parameters. Since that's a `const char*`,this hints to us that we'll have to point to an array of bytes that's inside of the target's address space.

Which is to say, we will be allocating some memory within that process, copying our DLL's path into that space, and then freeing it once we're done with the injection.

We'll also need to find the address where LoadLibraryA resides within the target process; therefore, we will need to find kernel32.dll's base address as well.. and that's about it! Now we can go to the next section:

# The plan

- Get a handle to the process;
- Allocate some memory in the target;
- Write our DLL path in that memory;
- Find kernel32.dll and the LoadLibraryA address;
- Create a remote thread that then calls LoadLibraryA;
- Free up the memory, close the handles etc.

# Writing the code

To avoid adding unnecessary complexity to this simple example, we'll directly request the process ID from the user:

```cpp
DWORD pid = 0;
std::cout << "Input process ID:\n";
std::cin >> pid;
```

If you want to automatically find the process by name, you might use an API such as Toolhelp32, in order to go through the list of processes and filter them by exeName. I might make an article on this in the future.

Using this information, we can now request a handle to the target process:

```cpp
HANDLE proc_handle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
assert(proc_handle != INVALID_HANDLE_VALUE);
```

The first parameter indicates what level of permissions we require. This tells the OS whether we want to read virtual memory, write to it, duplicate a handle, and so on. For the sake of this example, we'll be asking for all the permissions.

Notice the assert after the OpenProcess call. It's easy to write code that ends up failing for whatever reason, so it's important to know when and where the failure happens, to make our lives easier when it comes to debugging. (Of course, you should be using a better logging & error handling mechanism than this, but this is sufficient for our purposes).

Reading the DLL path is straightforward:

```cpp
std::string dll_name;
std::cout << "Input the full DLL path:\n";
std::cin >> dll_name;
const std::size_t dll_name_len = dll_name.length() + 1;
```

The purpose of `dll_name_len` is to make sure that we'll also be copying the null terminator of the string once we place it in our target's virtual memory.

Let's allocate some memory:

```cpp
LPVOID memory = VirtualAllocEx(
    proc_handle, 
    NULL, 
    dll_name_len,
    MEM_COMMIT | MEM_RESERVE,
    PAGE_READWRITE
);
assert(memory != NULL);
```

This will commit a piece of memory of size `dll_name_len` in the target process, that we can now write the DLL path to:

```cpp
std::size_t bytes_written = 0;
BOOL write_status = WriteProcessMemory(
    proc_handle,
    memory,
    dll_name.data(),
    dll_name_len,
    &bytes_written
);
assert(write_status == TRUE && bytes_written == dll_name_len);
```

The code is relatively self-explanatory, but we're writing `dll_name` in the process, including the null terminator (so that it's a valid C string), at the address stored in `memory`. We're checking if the function returned successfully, as well as if we actually wrote all the bytes in the process.

Before we can finally create the remote thread, we need to find the address of LoadLibraryA:

```cpp
HMODULE kernel32_handle = GetModuleHandleA("Kernel32.dll");
assert(kernel32_handle != NULL);

FARPROC loadlibrary_address = GetProcAddress(kernel32_handle, "LoadLibraryA");
assert(loadlibrary_address != NULL);
```

This piece of code actually relies on a querk related to how Windows loads known system DLL's. You might have noticed that we're actually finding the address of LoadLibrary within our own process, not within the target process. It turns out that this is perfectly okay: since Kernel32.dll is considered a "KnownDLL", it's actually only loaded once by the system, and all processes that want to reference it use the same base module address. This is a very cool topic that I might cover in a future article.

Now we have everything that we need in order to perform the injection itself:

```cpp
HANDLE thread_handle = CreateRemoteThread(
    proc_handle,
    NULL,
    0,
    (LPTHREAD_START_ROUTINE)loadlibrary_address,
    memory,
    0,
    NULL
);
assert(thread_handle != NULL);

WaitForSingleObject(thread_handle, INFINITE);

std::cout << "Successfully injected\n";
std::cin.get();
```

If you are curious what all of the function's parameters do, I recommend you check the Windows API documentation.

After creating the thread, we wait for it to finish its execution before moving on and declaring the DLL injection as successful.

Once the thread concludes its execution, we don't need the memory anymore, nor the handles that we've opened, so let's clean everything up:

```cpp
VirtualFreeEx(
    proc_handle,
    memory,
    0,
    MEM_RELEASE
);
CloseHandle(kernel32_handle);
CloseHandle(thread_handle);
CloseHandle(proc_handle);

return 0;
```

# Testing it out

To verify whether this actually works, we'll write a very simple DLL which creates a console window that will periodically display a message until we close it. You can find the code [here](https://github.com/fdh911/ez_dll_injector/blob/master/example_dll/src/Main.cc).

![The code in action]({{ "/assets/dll_injection1.gif" | relative_url }})
