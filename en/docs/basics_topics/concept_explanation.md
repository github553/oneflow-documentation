# Term/concept explanation

In this article, we will make a general explanation of some common terms and concepts in OneFlow.The main content is divided for algorithm engineer and framework developers:

-  **Algorithm development**
-  **Framework development**

In algorithms development part, we will explain some common terms and concepts may use in the process of deep learning algorithms development and model training. But in framework developing, we will focus on the inner design concepts of OneFlow and some relevant basic level concepts.



## Algorithms developing

### 1.Placeholder

Placeholder is **data Placeholder**, this concept is for define the shape of input or output data. It do not have real data in it.

For example:

```python
import oneflow.typing as oft
def test_job(images:oft.Numpy.Placeholder((32, 1, 28, 28), dtype=flow.float32),
             labels:.oft.Numpy.Placeholder((32, ), dtype=flow.int32)):
```

It define the input of a job function. The shape of image input is (32, 1, 28, 28) and the data type is flow.float32. The shape of labels is (32,) and data type is flow.int32.



### 2.Tensor and Blob

Tensor is the common concept in other framework. Such as Tensor in pytorch, it contain the data and data type, grad, storing device and other attribute. Tensor can use to structure and the description of the process of the forward/reverse calculation chart.

In OneFlow, the basics level also use the concept of Tensor. But there are some difference between the Tensor in OneFlow and Tensor in pytorch/tensorflow. In order to provide sufficient support for distributed and parallelism, Tensor in OneFlow is more complex and have more types and attributes. Such as logical, physical, devices and attributes of distribution. But a Tensor unified on logic level, could be divided to different devices on real time operations. So in order to simplified description, OneFlow hide the specific type of Tensor, all the things is define by a higher level concept Blob.



Blob in OneFlow have corresponding base class  `BlobDef`. You can print the attribute of  `Blob` when build network. Such as the following script can print  `conv1`, `shape` and `dtype`:

```python
print(conv1.shape, conv1.dtype)
```

Blob can only be Placeholder, but also can be the unit to pack the function's parameters.



### 3.Job Function

In OneFlow, we called training, evaluations, predictions and ratiocination tasks as job function. Job function connect logic of the users and  computing resources management of OneFlow.

In OneFlow, all functions need use decorator `@oneflow.global_function` to define as a job functions. By this decorator, we not only can define the model structure, optimize method and other task logic but also can pass the parameters need by job when running the job function. (like: get_train_config()). That can make OneFlow manage our memory and device resources more conveniently.



 **Why use global_function rather than function?**

The beginning of the design of OneFlow is to solve the task of multiple device in distribution training. In this situation, set global_function means make global configuration on devices.



### 4.Layer and Operator

#### Layer

The layer concept in OneFlow is basically same as the layer in tensorflow, pytorch and other popular deep learning framework. It is use to describe a layer in neural network. Like: conv2d convolution layer, batch_normalization layer, dense fully connected layer and layer_norm regularization layer.Layer can simplify the process of build the neural networks. For example you can use just few line of code to build Lenet:

```python
def lenet(data, train=False):
    initializer = flow.truncated_normal(0.1)
    conv1 = flow.layers.conv2d(data, 32, 5, padding='SAME', activation=flow.nn.relu, name='conv1',
                               kernel_initializer=initializer)
    pool1 = flow.nn.max_pool2d(conv1, ksize=2, strides=2, padding='SAME', name='pool1', data_format='NCHW')
    conv2 = flow.layers.conv2d(pool1, 64, 5, padding='SAME', activation=flow.nn.relu, name='conv2',
                               kernel_initializer=initializer)
    pool2 = flow.nn.max_pool2d(conv2, ksize=2, strides=2, padding='SAME', name='pool2', data_format='NCHW')
    reshape = flow.reshape(pool2, [pool2.shape[0], -1])
    hidden = flow.layers.dense(reshape, 512, activation=flow.nn.relu, kernel_initializer=initializer, name='dense1')
    if train: hidden = flow.nn.dropout(hidden, rate=0.5, name="dropout")
    return flow.layers.dense(hidden, 10, kernel_initializer=initializer, name='dense2')
```

The bottom of layer is composed by many operators. Like: `layers.conv2d` is composed by  `conv2d` and `variable`.

#### Operator

Operator is the **basics calculation unit** in OneFlow.The calculation in above example is complete by overlay of operator.flow.nn.max_pool2d and flow.reshape() all both operators.



### 5.Consistent/Mirrored View

OneFlow use two type of point of view:  **Mirrored View** and **Consistent View** is for describ distribution of data and model under distributed system. Different view is for different parallelism strategies.

Mirrored View is come from mirrors strategy of MPI distributed calculation. It is for describing model mirrored to multiple devices when use data parallelism.

Consistent View is trade multi devices and multi devices as one object in distribution environment. When use this strategy, OneFlow will hiding the detail process for user and chosse parallelism method in the optimal strategy(can be data/model/hybrid parallelism).

Basically:

When set the mirrored view(flow.scope.mirrored_view), it means only can use **data parallelism**.Such as set four solo device nodes in job function. Thus the model will be copy and paste to all devices but data will be cut in to four part and send to each device.

When set consistent view(flow.scope.consistent_view), It means no limit. OneFlow **can choose use data parallelism, model parallelism or hybrid parallelism.**



## Frame developing

### 1.Boxing

The model responsible for the logic switching different between different parallelism properties of tensor transformation mechanism/functionfor. We called it  **Boxing**.

Such as: When the upstream and downstream of the op has different parallelism characteristics (such as parallelism number different). OneFlow will use Boxing automatic processing all kinds of data conversion and transfer process.



### 2.SBP

In fact, all forwards or bcakward operations in neural network can calculate by matrixes. Matrixes calculation always have process which according to the axis cut broadcast.OneFlow also have same step, we call it SBP. Of course, the SBP in OneFlow is not only matrixes calculations. It also corresponding to divided data in different device and broadcast.

SBP means Split、broadcast、Partial sum.

#### Split

When parallelism operations, tensor is divided in to many sub-ensor.Different operators allow tensor to divided on different axis.Boxing will automatically process one tensor is divided in multiple axis.

#### Broadcast

When parallelism operator calculation, one tensor will be broadcast to many device. Make the same tensor amount in each device.

#### Partial Sum

If a operator have distributive attribute, tensor will sum part of dimension according to attribute of tensor.

