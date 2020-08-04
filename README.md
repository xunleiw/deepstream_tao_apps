# Integrate TLT model with DeepStream SDK

   * [Integrate TLT model with DeepStream SDK](#integrate-tlt-model-with-deepstream-sdk)
      * [Description](#description)
      * [Prerequisites](#prerequisites)
      * [Download](#download)
         * [1. Install <a href="https://github.com/git-lfs/git-lfs/wiki/Installation">git-lfs</a> (git &gt;= 1.8.2)](#1-install-git-lfs-git--182)
         * [2. Download Source Code with SSH or HTTPS](#2-download-source-code-with-ssh-or-https)
      * [Build](#build)
         * [1. Build TRT OSS Plugin](#1-build-trt-oss-plugin)
         * [2. Build Sample Application](#2-build-sample-application)
      * [Run](#run)
      * [Information for Customization](#information-for-customization)
         * [TLT Models](#tlt-models)
         * [Label Files](#label-files)
         * [DeepStream configuration file](#deepstream-configuration-file)
         * [Model Outputs](#model-outputs)
            * [1. Yolov3](#1-yolov3)
            * [2. Detectnet_v2](#2-detectnet_v2)
            * [3~5. RetinaNet / DSSD / SSD](#35-retinanet--dssd--ssd)
            * [6. FasterRCNN](#6-fasterrcnn)
         * [TRT Plugins Requirements](#trt-plugins-requirements)
      * [FAQ](#faq)
         * [Measure The Inference Perf](#measure-the-inference-perf)
      * [Known issues](#known-issues)

## Description

This repository provides a DeepStream sample application based on [NVIDIA DeepStream SDK](https://developer.nvidia.com/deepstream-sdk) to run six TLT models (**DetectNet_v2** / **Faster-RCNN** / **YoloV3** / **SSD** / **DSSD** / **RetinaNet**/ **MaskRCNN**) with below files:

- **deepstream_custom.c**: sample application main file
- **pgie_$(MODEL)_tlt_config.txt**: DeepStream nvinfer configure file
- **nvdsinfer_customparser_$(MODEL)_tlt**: include inference postprocessor and label file for the model
- **models**:  To download sample models that are trained by  trained by [NVIDIA Transfer Learning Toolkit(TLT) SDK](https://developer.nvidia.com/transfer-learning-toolkit) run `wget https://nvidia.box.com/shared/static/8k0zpe9gq837wsr0acoy4oh3fdf476gq.zip -O models.zip`
- **TRT-OSS**: TRT(TensorRT) OSS libs for some platforms/systems (refer to the README to build lib for other platforms/systems)

The pipeline of this sample is:
>
>
>   H264/JPEG-->decoder-->tee -->| -- (batch size) ------>|-->streammux--> nvinfer-->nvosd --> |---> encode --->filesink (save the output in local dir) / 
>                                                                                              |---> display
>

## Prerequisites

* [Deepstream SDK 5.0](https://developer.nvidia.com/deepstream-sdk)   

   Make sure deepstream-test1 sample can run successful to verify your installation

* [TensorRT OSS (release/7.x branch)](https://github.com/NVIDIA/TensorRT/tree/release/7.0)

  This is **ONLY** needed when running *SSD*, *DSSD*, *RetinaNet* and *YOLOV3* models because BatchTilePlugin required by these models is not supported by TensorRT7.x native package.

## Download

### 1. Install [git-lfs](https://github.com/git-lfs/git-lfs/wiki/Installation) (git >= 1.8.2)

*Need git-lfs to support downloading the >5MB model files.*  

```
curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
sudo apt-get install git-lfs
git lfs install
```

### 2. Download Source Code with SSH or HTTPS

```
// SSH
git clone git@github.com:NVIDIA-AI-IOT/deepstream_tlt_apps.git
// or HTTPS
git clone https://github.com/NVIDIA-AI-IOT/deepstream_tlt_apps.git
```

## Build

### 1. Build TRT OSS Plugin

Refer to below README to update libnvinfer_plugin.so* if want to run *SSD*, *DSSD*, *RetinaNet* or *YOLOV3*.

```
TRT-OSS/Jetson/README.md              // for Jetson platform
TRT-OSS/x86/README.md                 // for x86 platform
```

### 2. Build Sample Application

```
export DS_SRC_PATH="Your deepstream sdk source path"      // e.g. /opt/nvidia/deepstream/deepstream
export CUDA_VER=xy.z                                      // xy.z is CUDA version, e.g. 10.2
make
```
## Run

```
./deepstream-custom -c pgie_config_file -i <H264 or JPEG filename> [-b BATCH] [-d]
    -h: print help info
    -c: pgie config file, e.g. pgie_frcnn_tlt_config.txt
    -i: H264 or JPEG input file
    -b: batch size, this will override the value of "baitch-size" in pgie config file
    -d: enable display, otherwise dump to output H264 or JPEG file
 
 e.g.
 ./deepstream-custom  -c pgie_frcnn_tlt_config.txt -i $DS_SRC_PATH/samples/streams/sample_720p.h264
```

## Information for Customization

If you want to do some customization, such as training your own TLT model, running the model in other DeepStream pipeline, you should read below sections.  
### TLT Models

To download the sample models that we have trained with [NVIDIA Transfer Learning Toolkit(TLT) SDK](https://developer.nvidia.com/transfer-learning-toolkit) , run `wget https://nvidia.box.com/shared/static/8k0zpe9gq837wsr0acoy4oh3fdf476gq.zip -O models.zip`

After training finishes, run `tlt-export` to generate an `.etlt` model. This .etlt model can be deployed into DeepStream for fast inference as this sample shows.  
This DeepStream sample app also supports the TensorRT engine(plan) file generated by running the `tlt-converter` tool on the `.etlt` model.  
The TensorRT engine file is hardware dependent, while the `.etlt` model is not. You may specify either a TensorRT engine file or a `.etlt` model in the DeepStream configuration file.

### Label Files

The label file includes the list of class names for a model, which content varies for different models.  
User can find the detailed label information for the MODEL in the README.md and the label file under *nvdsinfer_customparser_$(MODEL)_tlt/*, e.g. ssd label informantion under *nvdsinfer_customparser_ssd_tlt/*  
  
Note, for FasterRCNN, DON'T forget to include "background" lable and change num-detected-classes in pgie configure file accordingly 

### DeepStream configuration file

The DeepStream configuration file includes some runtime parameters for DeepStream **nvinfer** plugin, such as model path, label file path, TensorRT inference precision, input and output node names, input dimensions and so on.  
In this sample, each model has its own DeepStream configuration file, e.g. pgie_dssd_tlt_config.txt for DSSD model.
Please refer to [DeepStream Development Guide](https://docs.nvidia.com/metropolis/deepstream/dev-guide/index.html#page/DeepStream_Development_Guide%2Fdeepstream_app_config.3.2.html) for detailed explanations of those parameters.

### Model Outputs

#### 1. Yolov3

The model has the following four outputs:

- **num_detections**: A [batch_size] tensor containing the INT32 scalar indicating the number of valid detections per batch item. It can be less than keepTopK. Only the top num_detections[i] entries in nmsed_boxes[i], nmsed_scores[i] and nmsed_classes[i] are valid
- **nmsed_boxes**: A [batch_size, keepTopK, 4] float32 tensor containing the coordinates of non-max suppressed boxes
- **nmsed_scores**: A [batch_size, keepTopK] float32 tensor containing the scores for the boxes
- **nmsed_classes**: A [batch_size, keepTopK] float32 tensor containing the classes for the boxes

#### 2. Detectnet_v2

The model has the following two outputs:

- **output_cov/Sigmoid**: A [batchSize, Class_Num, gridcell_h, gridcell_w] tensor contains the number of gridcells that are covered by an object
- **output_bbox/BiasAdd**: a [batchSize, Class_Num, 4] contains the normalized image coordinates of the object (x1, y1) top left and (x2, y2) bottom right with respect to the grid cell

#### 3~5. RetinaNet / DSSD / SSD

These three models have the same output layer named NMS which implementation can refer to TRT OSS [nmsPlugin](https://github.com/NVIDIA/TensorRT/tree/master/plugin/nmsPlugin):

* an output of shape [batchSize, 1, keepTopK, 7] which contains nmsed box class IDs(1 value), nmsed box scores(1 value) and nmsed box locations(4 value)
* another output of shape [batchSize, 1, 1, 1] which contains the output nmsed box count.

#### 6. FasterRCNN

The model has the following three outputs:

- **dense_regress_td/BiasAdd**: A [batchSize, R, C*4] tensor containing the bounding box regression
- **dense_class_td/Softmax**:  A [batchSize, R, C+1] tensor containing the class id and scores
- **proposal**:  A [batchSize, R, 4] tensor containing the rois  
  R = post NMS top N (usually 300)  
  C = class numbers (+1 means background)  

### 7. MaskRCNN

The model has the following two outputs:

- **generate_detections**: A [batchSize, keepTopK, C*6] tensor containing the bounding box, class id, score
- **mask_head/mask_fcn_logits/BiasAdd**:  A [batchSize, keepTopK, C+1, 28*28] tensor containing the masks

### TRT Plugins Requirements

> **FasterRCNN**: cropAndResizePlugin,  proposalPlugin    
> **SSD/DSSD/RetinaNet**:  batchTilePlugin, nmsPlugin   
> **YOLOV3**:  batchTilePlugin, resizeNearestPlugin, batchedNMSPlugin

> **MaskRCNN**:  generateDetectionPlugin, MultilevelCropAndResize, MultilevelProposeROI

## FAQ

### Measure The Inference Perf

```CQL
# 1.  Build TensorRT Engine through this smample, for example, build YoloV3 with batch_size=2
./deepstream-custom -c pgie_yolov3_tlt_config.txt -i /opt/nvidia/deepstream/deepstream/samples/streams/sample_720p.h264 -b 2
## after this is done, it will generate the TRT engine file under models/$(MODEL), e.g. models/yolov3/ for above command.
# 2. Measure the Inference Perf with trtexec, following above example
cd models/yolov3/
trtexec --batch=2 --useSpinWait --loadEngine=yolo_resnet18.etlt_b2_gpu0_fp16.engine
## then you can find the per *BATCH* inference time in the trtexec output log
```

## Known issues


* LFS bandwidth issue

    * `wget https://nvidia.box.com/shared/static/8k0zpe9gq837wsr0acoy4oh3fdf476gq.zip -O models.zip` to get the models
    * for prebuilt lib issues, you can refer the README under ./TRT-OSS


