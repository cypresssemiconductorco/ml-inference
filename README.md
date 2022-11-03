# ModusToolbox™ Machine Learning Inference Engine Library

## Overview

The ModusToolbox™ ML Inference Engine is a library which deploys an ML model and performs complete inference in different numerical formats on PSoC™ 6 embedded devices.

## Requirements

* [ModusToolbox™ software](https://www.cypress.com/products/modustoolbox-software-environment) v2.4
* Board Support Package (BSP) minimum required version: 2.3.0
* Programming Language: C
* Associated Parts: See "Supported Kits" section below.

## Supported Toolchains (make variable 'TOOLCHAIN')

* GNU Arm® Embedded Compiler v10.3.1 (`GCC_ARM`)
* Arm compiler v6.16 (`ARM`)
* IAR C/C++ compiler v9.30.1 (`IAR`)

## Supported Device Family

* [PSoC™ 6](https://www.infineon.com/cms/en/product/microcontroller/32-bit-psoc-arm-cortex-microcontroller/psoc-6-32-bit-arm-cortex-m4-mcu/)

## Features

Supports the following typical ML models:
* MLP - Dense feed-forward network
* CONV1D
* CONV2D
* RNNs - GRU

Supports floating-point and fixed-point variants:
* 32-bit floating-point
* 16-bit fixed-point input
* 8-bit fixed-point input
* 16-bit fixed-point weight
* 8-bit fixed-point weight
* Support model, layer, and frame profiling

## Quick Start

### Adding the library

You can add a dependency file (MTB format) under the deps folder or use the Library Manager to add it in your project. It is available under Library > MCU Middleware > ml.

In the Makefile of the project, you need to define the quantization to be deployed. Note that you can only choose one the type of quantization. In the COMPONENTS parameter, add one of the following:
* ML_FLOAT32: use 32-bit floating-point for the weights and input data
* ML_INT16x16: use 16-bit fixed-point for the weights and input data
* ML_INT16x8: use 16-bit fixed-point for the input data and 8-bit for the weights
* ML_INT8x8: use 8-bit fixed-point for the weights and input data

### Using the library

There are four steps to use the ML-inference engine library.

#### Step 1: Get required memory for the inference engine

One of the data arrays generated by the ML Configurator Tool contains the model parameters. If using binary file, it is placed in the *NN_model_prms.bin*. If using C array header, it is placed in the *NN_model_all.h* > NN_model_prms_bin[].

Use the `Cy_ML_Model_Parse()` function to extract information from the data. It tells how much memory is required by the persistent and scratch memory.

#### Step 2: Allocate Memory

Once you know how much memory is required by the persistent and scratch memory, allocate it in your application. Here is an example in C on how to perform Step 1 and 2.

```c
#include NN_model_all.h
...
cy_stc_ml_model_info_t model_xx_info;

count = Cy_ML_Model_Parse(NN_model_prms_bin, &model_xx_info);
if (count > 0) /* Model parsing successful */
{
    persistent_mem = (char*) malloc(model_xx_info.persistent_mem*sizeof(char));
    scratch_mem = (char*) malloc(model_xx_info.scratch_mem*sizeof(char));
}
```

#### Step 3: Initialize Model and get Model Container/Object

After allocating memory, you can initialize the interference engine by providing pointers to the memory and the model weights. Here is an example in C using float quantization:

```c
#include NN_model_all.h
...
void *model_xx_obj;

result = Cy_ML_Model_Init(&model_xx_obj,      // NN model data container pointer
                          &NN_model_flt_bin,  // NN model parameter buffer pointer
                          persistent_mem,     // Pointer to allocated persistent mem
                          scratch_mem,        // Pointer to allocated scratch mem
                          &model_xx_info)     // Pointer to model info structure
```

#### Step 4: Run the inference engine

The last step is to run the inference engine. In this step, the input data can come from sensors or from the regression data. Here is an example in C using float quantization.

```c
result = Cy_ML_Model_Inference(model_xx_obj,  // NN model data container pointer
                               in_buffer,     // Input buffer
                               out_buffer,    // Output buffer
                               NULL);         // Not used in floating-point
```

Note that the last argument of the function above is only used in fixed-point. it is the pointer to input data and output data fixed-point Q factor.

#### Optional Step: Reset state memory in Recurrent NN model

This is optional and can be used to reset inference engine state memory in Recurrent NN model. If there is a need to do the state memory reset, here is API function. This API can be called after initilization.
If periodic reset is desired, call this API with desired window size, if one-time reset is desired, call this routine
twice, first to reset (1) and then clear with this routine after Cy_ML_Model_Inference() call.
```c
result = Cy_ML_Rnn_State_Control(model_xx_obj   // NN model data container pointer
                               rnn_status       // Recurrent NN reset (1) or clear (0)
                               rnn_reset_win);  // Recurrent NN reset window size
```

### Verifying the inference

The ML-inference engine library integrates a profiling mechanism to obtain the number of cycles the library takes to execute. To make this work, the application needs to implement the following function:

```c
int Cy_ML_Profile_Get_Tsc(uint32_t *val)
```

This function needs to return a counter value that increments on every CPU cycle. One way to implement this is to run a timer continuously that triggers an interrupt every second to increment a variable to represent how many seconds passed. On calling `Cy_ML_Profile_Get_Tsc`, it gets the number of seconds passed and combine with the current counter. Then it translates the result in CPU cycles.

To actually enable the profiling from the ML-inference engine library, a few functions need to be called. Here is the
flow:

1. Call `Cy_ML_Profile_Init()` to initialize the profiling ond setup callback function to handle the profile log.
2. Call `Cy_ML_Profile_Control()` to change the profile configuration during inference
3. Call `Cy_ML_Profile_Print()` to process profile log after inference

An example using the regression data generated by the ML Configurator tool and the profiling integrated in the ML-inference engine library is available in this link:
https://github.com/cypresssemiconductorco/mtb-example-ml-profiler

There are few options of profiling information printed when enabled, as shown in the following list:

| Profile setting                        | Description |
| :---                                   | :----       |
| `CY_ML_PROFILE_DISABLE`                | Disable profiling. |
| `CY_ML_PROFILE_ENABLE_LAYER`           | Enable layer profiling. It prints general information of each layer of the NN model, such as average cycle, peak cycle and peak frame. |
| `CY_ML_PROFILE_ENABLE_LAYER_PER_FRAME` | Enable layer profiling for each frame. On top of layer profiling, it also prints the cycles for each frame at each layer. |
| `CY_ML_PROFILE_ENABLE_MODEL`           | Enable the NN model profiling. It prints general information about the NN model, such as average cycle, peak cycle and peak frame. |
| `CY_ML_PROFILE_ENABLE_MODEL_PER_FRAME` | Enable the per frame profiling. On top of model profiling, it also prints the cycles for each frame. |
| `CY_ML_LOG_ENABLE_MODEL_LOG`           | Enable the model output log. It prints the output values generated by the inference engine. |
| `CY_ML_LOG_ENABLE_LAYER_LOG`           | Enable the layer output log. It prints the output values of each layer in the NN model. |

## More information
The following resources contain more information:
* [ModusToolbox™ Machine Learning Design Support](https://www.infineon.com/cms/en/design-support/tools/sdk/modustoolbox-software/modustoolbox-machine-learning/)
* [ModusToolbox™  ML Inference Engine RELEASE.md](./RELEASE.md)
* [ModusToolbox™  ML Inference Engine API Reference Guide](https://cypresssemiconductorco.github.io/ml-inference/html/index.html)
* [ModusToolbox™ Software Environment, Quick Start Guide, Documentation, and Videos](https://www.cypress.com/products/modustoolbox-software-environment)
* [Cypress Semiconductor](http://www.cypress.com)

---
© 2021-2022, Cypress Semiconductor Corporation (an Infineon company) or an affiliate of Cypress Semiconductor Corporation.
