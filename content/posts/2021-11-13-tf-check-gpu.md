---
title: "How to check if your deep learning library is actually using the GPU"
date: 2021-11-13T20:54:39Z
draft: true
tags = ["cheatsheet", "tensorflow", "pytorch", "cuda"]
---

If you've got a GPU on your system that you want to run your deep learning model on, you'd probably want to check that the library is able to access the GPU. Installation issues/ incorrect setups etc can mean that it's actually inaccessible. I have googled far too many times 'is tensorflow/pytorch accessing the GPU', so putting this down here so I don't have to go through the same stackoverflow posts again and again :sweat_smile:

### Tensorflow (as of version 2.7)

```python
import tensorflow as tf

assert len(tf.config.list_physical_devices('GPU')) > 0
# if your system has a GPU, tf.config.list_physical_devices('GPU') will return a list like so [PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]

# get cuda version-- if you have installed cudatoolkit using conda in your env, rather than using a global cuda, you can
# check this version is what you expect
sys_details = tf.sysconfig.get_build_info()
cuda_version = sys_details["cuda_version"]
print(cuda_version)

# get cudnn version-- if you have installed cudnn using conda in your env, rather than using a global cuda, you can
# check this version is what you expect
cudnn_version = sys_details["cudnn_version"]  
print(cudnn_version)
```

### Pytorch (as of version 1.10)

```python
import torch

# this should print True if the GPU is accessible
print(torch.cuda.is_available())

```

Of course, one of the best way to avoid installation issues with a gpu setup and tensorflow/pytorch etc is using conda. Unfortunately tf is not 'officially' available on conda, and you might not get latest version, and installing extra tf related libraries that are only available through pip such as tensorflow-io could mean the conda installed version of tensorflow that is linked to the cudatoolkit is uninstalled... 
