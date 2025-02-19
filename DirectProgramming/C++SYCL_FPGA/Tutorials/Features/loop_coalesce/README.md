
# Coalescing Nested Loops
This FPGA tutorial demonstrates applying the `loop_coalesce` attribute to a nested loop in a task kernel to reduce the area overhead.

| Optimized for                     | Description
---                                 |---
| OS                                | Linux* Ubuntu* 18.04/20.04 <br> RHEL*/CentOS* 8 <br> SUSE* 15 <br> Windows* 10
| Hardware                          | Intel® Agilex® 7, Arria® 10, and Stratix® 10 FPGAs
| Software                          | Intel® oneAPI DPC++/C++ Compiler
| What you will learn               |  What the `loop_coalesce` attribute does <br> How `loop_coalesce` attribute affects resource usage and loop throughput <br> How to apply the `loop_coalesce` attribute to loops in your program <br> Which loops make good candidates for coalescing
| Time to complete                  | 15 minutes

> **Note**: Even though the Intel DPC++/C++ OneAPI compiler is enough to compile for emulation, generating reports and generating RTL, there are extra software requirements for the simulation flow and FPGA compiles.
>
> For using the simulator flow, Intel® Quartus® Prime Pro Edition and one of the following simulators must be installed and accessible through your PATH:
> - Questa*-Intel® FPGA Edition
> - Questa*-Intel® FPGA Starter Edition
> - ModelSim® SE
>
> When using the hardware compile flow, Intel® Quartus® Prime Pro Edition must be installed and accessible through your PATH.
>
> :warning: Make sure you add the device files associated with the FPGA that you are targeting to your Intel® Quartus® Prime installation.

## Prerequisites

This sample is part of the FPGA code samples.
It is categorized as a Tier 2 sample that demonstrates a compiler feature.

```mermaid
flowchart LR
   tier1("Tier 1: Get Started")
   tier2("Tier 2: Explore the Fundamentals")
   tier3("Tier 3: Explore the Advanced Techniques")
   tier4("Tier 4: Explore the Reference Designs")

   tier1 --> tier2 --> tier3 --> tier4

   style tier1 fill:#0071c1,stroke:#0071c1,stroke-width:1px,color:#fff
   style tier2 fill:#f96,stroke:#333,stroke-width:1px,color:#fff
   style tier3 fill:#0071c1,stroke:#0071c1,stroke-width:1px,color:#fff
   style tier4 fill:#0071c1,stroke:#0071c1,stroke-width:1px,color:#fff
```

Find more information about how to navigate this part of the code samples in the [FPGA top-level README.md](/DirectProgramming/C++SYCL_FPGA/README.md).
You can also find more information about [troubleshooting build errors](/DirectProgramming/C++SYCL_FPGA/README.md#troubleshooting), [running the sample on the Intel® DevCloud](/DirectProgramming/C++SYCL_FPGA/README.md#build-and-run-the-samples-on-intel-devcloud-optional), [using Visual Studio Code with the code samples](/DirectProgramming/C++SYCL_FPGA/README.md#use-visual-studio-code-vs-code-optional), [links to selected documentation](/DirectProgramming/C++SYCL_FPGA/README.md#documentation), etc.

## Purpose
The `loop_coalesce` attribute enables you to direct the compiler to combine nested loops into a single loop. The attribute `[[intel::loop_coalesce(N)]]` takes an integer argument `N`, that specifies how many nested loop levels that you want the compiler to attempt to coalesce.

**NOTE**: If you specify `[[intel::loop_coalesce(1)]]` on nested loops, the compiler does not attempt to coalesce any of the nested loops.
### Example: Coalescing Two Loops

```
[[intel::loop_coalesce(2)]]
for (int i = 0; i < N; i++)
  for (int j = 0; j < M; j++)
    sum[i][j] += i+j;
```
The compiler coalesces the two loops together so that they execute as if they were a single loop written as follows:

```
int i = 0;
int j = 0;
while(i < N){
  sum[i][j] += i+j;
  j++;
  if (j == M){
    j = 0;
    i++;
  }
}
```

### Identifying Which Loops to Coalesce
Generally, coalescing loops can help reduce area usage by reducing the overhead needed for loop control. However, in some circumstances, coalescing loops can reduce kernel throughput. Scenarios where the `loop_coalesce` attribute can be applied to save area without a loss of throughput, are those where:

   1. The loops being coalesced have the same initiation interval (II).
   2. The exit condition computation for the resulting coalesced look is not complicated.

If the innermost coalesced loop has a very small trip count, `loop_coalesce` might actually improve throughput.

## Key Concepts
* Description of the `loop_coalesce` attribute.
* How `loop_coalesce` attribute affects resource usage and loop throughput.
* How to apply the `loop_coalesce` attribute to loops in your program.
* Determining which loops make good candidates for coalescing.


## Building the `loop_coalesce` Tutorial

> **Note**: When working with the command-line interface (CLI), you should configure the oneAPI toolkits using environment variables.
> Set up your CLI environment by sourcing the `setvars` script located in the root of your oneAPI installation every time you open a new terminal window.
> This practice ensures that your compiler, libraries, and tools are ready for development.
>
> Linux*:
> - For system wide installations: `. /opt/intel/oneapi/setvars.sh`
> - For private installations: ` . ~/intel/oneapi/setvars.sh`
> - For non-POSIX shells, like csh, use the following command: `bash -c 'source <install-dir>/setvars.sh ; exec csh'`
>
> Windows*:
> - `C:\Program Files(x86)\Intel\oneAPI\setvars.bat`
> - Windows PowerShell*, use the following command: `cmd.exe "/K" '"C:\Program Files (x86)\Intel\oneAPI\setvars.bat" && powershell'`
>
> For more information on configuring environment variables, see [Use the setvars Script with Linux* or macOS*](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-linux-or-macos.html) or [Use the setvars Script with Windows*](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-programming-guide/top/oneapi-development-environment-setup/use-the-setvars-script-with-windows.html).

### On a Linux* System

1. Generate the `Makefile` by running `cmake`.
  ```
  mkdir build
  cd build
  ```
  To compile for the default target (the Agilex® 7 device family), run `cmake` using the command:
  ```
  cmake ..
  ```

  > **Note**: You can change the default target by using the command:
  >  ```
  >  cmake .. -DFPGA_DEVICE=<FPGA device family or FPGA part number>
  >  ```
  >
  > Alternatively, you can target an explicit FPGA board variant and BSP by using the following command:
  >  ```
  >  cmake .. -DFPGA_DEVICE=<board-support-package>:<board-variant>
  >  ```
  >
  > You will only be able to run an executable on the FPGA if you specified a BSP.

2. Compile the design through the generated `Makefile`. The following build targets are provided, matching the recommended development flow:

   * Compile for emulation (fast compile time, targets emulated FPGA device):
      ```
      make fpga_emu
      ```
   * Generate the optimization report:
     ```
     make report
     ```
   * Compile for simulation (fast compile time, targets simulated FPGA device):
     ```
     make fpga_sim
     ```
   * Compile for FPGA hardware (longer compile time, targets FPGA device):
     ```
     make fpga
     ```

### On a Windows* System

1. Generate the `Makefile` by running `cmake`.
  ```
  mkdir build
  cd build
  ```
  To compile for the default target (the Agilex® 7 device family), run `cmake` using the command:
  ```
  cmake -G "NMake Makefiles" ..
  ```
  > **Note**: You can change the default target by using the command:
  >  ```
  >  cmake -G "NMake Makefiles" .. -DFPGA_DEVICE=<FPGA device family or FPGA part number>
  >  ```
  >
  > Alternatively, you can target an explicit FPGA board variant and BSP by using the following command:
  >  ```
  >  cmake -G "NMake Makefiles" .. -DFPGA_DEVICE=<board-support-package>:<board-variant>
  >  ```
  >
  > You will only be able to run an executable on the FPGA if you specified a BSP.

2. Compile the design through the generated `Makefile`. The following build targets are provided, matching the recommended development flow:

   * Compile for emulation (fast compile time, targets emulated FPGA device):
     ```
     nmake fpga_emu
     ```
   * Generate the optimization report:
     ```
     nmake report
     ```
   * Compile for simulation (fast compile time, targets simulated FPGA device):
     ```
     nmake fpga_sim
     ```
   * Compile for FPGA hardware (longer compile time, targets FPGA device):
     ```
     nmake fpga
     ```

> **Note**: If you encounter any issues with long paths when compiling under Windows*, you may have to create your ‘build’ directory in a shorter path, for example c:\samples\build.  You can then run cmake from that directory, and provide cmake with the full path to your sample directory.

## Examining the Reports
Locate `report.html` in the `loop_coalesce_report.prj/reports/` directory. Open the report in Chrome*, Firefox*, Edge*, or Internet Explorer*.

On the main report page, scroll down to the section titled `Compile Estimated Kernel Resource Utilization Summary`. Each kernel name ends in the loop_coalesce attribute argument used for that kernel, e.g., KernelCompute<2> uses a loop_coalesce argument of 2. You can verify that the number of registers, MLABs and DSPs used for each kernel decreases after nested loops are coalesced.

## Running the Sample

1. Run the sample on the FPGA emulator (the kernel executes on the CPU):
  ```bash
  ./loop_coalesce.fpga_emu     (Linux)
  loop_coalesce.fpga_emu.exe   (Windows)
  ```
2. Run the sample on the FPGA simulator device:
  * On Linux
    ```bash
    CL_CONTEXT_MPSIM_DEVICE_INTELFPGA=1 ./loop_coalesce.fpga_sim
    ```
  * On Windows
    ```bash
    set CL_CONTEXT_MPSIM_DEVICE_INTELFPGA=1
    loop_coalesce.fpga_sim.exe
    set CL_CONTEXT_MPSIM_DEVICE_INTELFPGA=
    ```
3. Run the sample on the FPGA device (only if you ran `cmake` with `-DFPGA_DEVICE=<board-support-package>:<board-variant>`):
  ```bash
  ./loop_coalesce.fpga         (Linux)
  loop_coalesce.fpga.exe       (Windows)
  ```

### Example of Output

```
Loop Coalesce: 1 -- kernel time : 156 microseconds
Throughput for kernel with coalesce_factor 1: 6550KB/S
Loop Coalesce: 2 -- kernel time : 113 microseconds
Throughput for kernel with coalesce_factor 2: 9064KB/S
PASSED: The results are correct

```

### Discussion of Results
The execution time and throughput for each kernel is displayed. Applying the `loop_coalesce` attribute in this example reduced the kernel execution time by a factor of ~1.5. Note that you will only see this result when executing on FPGA hardware. The emulator will generally not reflect performance differences.

## License

Code samples are licensed under the MIT license. See [License.txt](https://github.com/oneapi-src/oneAPI-samples/blob/master/License.txt) for details.

Third-party program Licenses can be found here: [third-party-programs.txt](https://github.com/oneapi-src/oneAPI-samples/blob/master/third-party-programs.txt).
