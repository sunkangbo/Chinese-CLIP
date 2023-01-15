[**中文说明**](deployment.md) | **English**

# Chinese-CLIP模型部署：ONNX & TensorRT格式转换

最新的Chinese-CLIP代码，已经支持将各规模的Pytorch模型，转换为[ONNX](https://onnx.ai/)或[TensorRT](https://developer.nvidia.com/tensorrt)格式，从而相比原始Pytorch模型**提升特征计算的推理速度**，同时不影响特征提取的下游任务效果。下面我们给出在GPU上，准备ONNX和TensorRT格式的FP16 Chinese-CLIP部署模型的整个流程，同时给出模型效果和推理速度的对比，方便大家上手利用ONNX和TensorRT库在推理性能上的优势。

## 环境准备

+ **GPU硬件要求**：请准备**Volta架构及以上**的Nvidia GPU显卡（配备FP16 Tensor Core），Nvidia各架构对应显卡型号请参见[此文档表格](https://en.wikipedia.org/wiki/CUDA#GPUs_supported)。本文我们以T4显卡为例
+ **CUDA**：推荐[CUDA](https://developer.nvidia.com/cuda-11-6-0-download-archive)版本11.6及以上，本文以11.6为例
+ **CUDNN**：推荐[CUDNN](https://developer.nvidia.com/rdp/cudnn-archive) 8.6.0及以上，本文以8.6.0为例。请注意TensorRT和CUDNN有版本match关系，如TensorRT 8.5.x必须使用CUDNN 8.6.0，详见TensorRT的版本要求
+ **ONNX**：请安装`pip install onnx onnxruntime-gpu onnxmltools`。注意我们转换TensorRT模型时，将沿着Pytorch → ONNX → TensorRT的步骤，所以准备TensorRT模型也需要先安装ONNX库。本文以onnx版本1.13.0，onnxruntime-gpu版本1.13.1，onnxmltools版本1.11.1为例
+ **TensorRT**：推荐[TensorRT](https://docs.nvidia.com/deeplearning/tensorrt/archives/index.html#trt_8)版本8.5.x，本文以8.5.2.2为例，使用pip即可安装`pip install tensorrt==8.5.2.2`。TensorRT各版本对应的CUDNN匹配版本，请从[文档页面](https://docs.nvidia.com/deeplearning/tensorrt/archives/index.html#trt_8)，查阅此TensorRT版本的"NVIDIA TensorRT Support Matrix"
+ **Pytorch**：推荐1.12.1及以上，本文以1.12.1为例（建议直接pip安装1.12.1+cu116，环境尽量不要再使用conda安装cudatoolkit，避免环境CUDNN版本变化，导致TensorRT报错）
+ [requirements.txt](requirements.txt)要求的其他依赖项


## 转换和运行ONNX模型

### 转换模型

将Pytorch模型checkpoint转换为ONNX格式的代码，请参见`cn_clip/deploy/pytorch_to_onnx.py`。我们以转换ViT-B-16规模的Chinese-CLIP预训练模型为例，具体的代码运行方式如下（请参考Readme[代码组织部分](https://github.com/OFA-Sys/Chinese-CLIP#代码组织)建好`${DATAPATH}`并替换下面的脚本内容）：

```bash
cd Chinese-CLIP/
export CUDA_VISIBLE_DEVICES=0
export PYTHONPATH=${PYTHONPATH}:`pwd`/cn_clip

# ${DATAPATH}的指定，请参考Readme"代码组织"部分创建好目录：https://github.com/OFA-Sys/Chinese-CLIP#代码组织
checkpoint_path=${DATAPATH}/pretrained_weights/clip_cn_vit-b-16.pt # 指定要转换的ckpt完整路径
mkdir -p ${DATAPATH}/deploy/ # 创建ONNX模型的输出文件夹

python cn_clip/deploy/pytorch_to_onnx.py \
       --model-arch ViT-B-16 \
       --pytorch-ckpt-path ${checkpoint_path} \
       --save-onnx-path ${DATAPATH}/deploy/vit-b-16 \
       --convert-text --convert-vision
```

其中各配置项定义如下：
+ `model-arch`: 模型规模，选项包括`["RN50", "ViT-B-16", "ViT-L-14", "ViT-L-14-336", "ViT-H-14"]`，各规模细节详见[Readme](https://github.com/OFA-Sys/Chinese-CLIP#模型规模--下载链接)
+ `pytorch-ckpt-path`: 指定Pytorch模型ckpt路径，上面的代码示例中我们指定为预训练的ckpt路径，也可以指定为用户finetune ckpt的位置。ckpt中的参数需要与`model-arch`指定的模型规模对应
+ `save-onnx-path`: 指定输出ONNX格式模型的路径（前缀）。完成转换后，代码将分别输出文本侧和图像侧的ONNX格式编码模型文件，FP32与FP16各一版，该参数即指定了以上输出文件的路径前缀
+ `convert-text`和`convert-vision`: 指定是否转换文本侧和图像侧模型
+ `context-length`（可选）: 指定文本侧ONNX模型，接收输入的序列长度，默认为我们预训练ckpt所使用的52
+ `download-root`（可选）: 如果不指定`pytorch-ckpt-path`，代码将根据`model-arch`自动下载Chinese-CLIP官方预训练ckpt用于转换，存放于`download-root`指定的目录

运行此代码转换完成后，将得到以下的log输出：
```
Finished PyTorch to ONNX conversion...
>>> The text FP32 ONNX model is saved at ${DATAPATH}/deploy/vit-b-16.txt.fp32.onnx
>>> The text FP16 ONNX model is saved at ${DATAPATH}/deploy/vit-b-16.txt.fp16.onnx with extra file ${DATAPATH}/deploy/vit-b-16.txt.fp16.onnx.extra_file
>>> The vision FP32 ONNX model is saved at ${DATAPATH}/deploy/vit-b-16.img.fp32.onnx
>>> The vision FP16 ONNX model is saved at ${DATAPATH}/deploy/vit-b-16.img.fp16.onnx with extra file ${DATAPATH}/deploy/vit-b-16.img.fp16.onnx.extra_file
```

上面示例代码执行结束后，我们得到了ViT-B-16规模，Chinese-CLIP文本侧和图像侧的ONNX模型，可以分别用于提取图文特征。输出ONNX模型的路径均以运行脚本时的`save-onnx-path`为前缀，后面依次拼上`.img`/`.txt`、`.fp16`/`.fp32`、`.onnx`。我们后续将主要使用FP16格式的ONNX模型`vit-b-16.txt.fp16.onnx`和`vit-b-16.img.fp16.onnx`。注意到部分ONNX模型文件还附带有一个extra_file，其也是对应ONNX模型的一部分。在使用这些ONNX模型时，请保留extra_file与ONNX文件放在一起，才能保证正常的推理。

### 运行模型

#### 提取图像侧特征
我们在`Chinese-CLIP/`目录下，使用以下的示例代码，读取刚刚转换好的ViT-B-16规模ONNX图像侧模型`vit-b-16.img.fp16.onnx`，并为Readme中示例的[皮卡丘图片](examples/pokemon.jpeg)提取图像侧特征。注意转换好的ONNX模型只接受batch大小为1的输入，即一次调用只处理一张输入图片

```python
# 完成必要的import（下文省略）
import onnxruntime
from PIL import Image
import numpy as np
import torch
import argparse
import cn_clip.clip as clip
from clip import load_from_name, available_models
from clip.utils import _MODELS, _MODEL_INFO, _download, available_models, create_model, image_transform

# 载入ONNX图像侧模型（**请替换${DATAPATH}为实际的路径**）
img_sess_options = onnxruntime.SessionOptions()
img_run_options = onnxruntime.RunOptions()
img_run_options.log_severity_level = 2
img_onnx_model_path="${DATAPATH}/deploy/vit-b-16.img.fp16.onnx"
img_session = onnxruntime.InferenceSession(img_onnx_model_path,
                                        sess_options=img_sess_options,
                                        providers=["CUDAExecutionProvider"])

# 预处理图片
model_arch = "ViT-B-16" # 这里我们使用的是ViT-B-16规模，其他规模请对应修改
preprocess = image_transform(_MODEL_INFO[model_arch]['input_resolution'])
# 示例皮卡丘图片，预处理后得到[1, 3, 分辨率, 分辨率]尺寸的Torch Tensor
image = preprocess(Image.open("examples/pokemon.jpeg")).unsqueeze(0)

# 用ONNX模型计算图像侧特征
image_features = img_session.run(["unnorm_image_features"], {"image": image.cpu().numpy()})[0] # 未归一化的图像特征
image_features = torch.tensor(image_features)
image_features /= image_features.norm(dim=-1, keepdim=True) # 归一化后的Chinese-CLIP图像特征，用于下游任务
print(image_features.shape) # Torch Tensor shape: [1, 特征向量维度]
```

#### 提取文本侧特征

类似地，我们用如下代码完成文本侧ONNX模型的载入与特征计算，与图像侧相同，文本侧ONNX部署模型只接受batch大小为1的输入，即一次调用只处理一条输入文本。我们为4条候选文本依次计算ViT-B-16规模模型的文本特征。import相关代码与上文相同，这里省略：

```python
# 载入ONNX文本侧模型（**请替换${DATAPATH}为实际的路径**）
txt_sess_options = onnxruntime.SessionOptions()
txt_run_options = onnxruntime.RunOptions()
txt_run_options.log_severity_level = 2
txt_onnx_model_path="${DATAPATH}/deploy/vit-b-16.txt.fp16.onnx"
txt_session = onnxruntime.InferenceSession(txt_onnx_model_path,
                                        sess_options=txt_sess_options,
                                        providers=["CUDAExecutionProvider"])

# 为4条输入文本进行分词。序列长度指定为52，需要和转换ONNX模型时保持一致（参见转换时的context-length参数）
text = clip.tokenize(["杰尼龟", "妙蛙种子", "小火龙", "皮卡丘"], context_length=52) 

# 用ONNX模型依次计算文本侧特征
text_features = []
for i in range(len(text)):
    one_text = np.expand_dims(text[i].cpu().numpy(),axis=0)
    text_feature = txt_session.run(["unnorm_text_features"], {"text":one_text})[0] # 未归一化的文本特征
    text_feature = torch.tensor(text_feature)
    text_features.append(text_feature)
text_features = torch.squeeze(torch.stack(text_features),dim=1) # 4个特征向量stack到一起
text_features = text_features / text_features.norm(dim=1, keepdim=True) # 归一化后的Chinese-CLIP文本特征，用于下游任务
print(text_features.shape) # Torch Tensor shape: [4, 特征向量维度]
```

#### 计算图文相似度

ONNX模型产出的归一化图文特征，内积后softmax即可计算相似度（需要考虑`logit_scale`），与原始Pytorch模型操作相同，参见以下代码。import相关代码与上文相同，这里省略：

```python
# 内积后softmax
# 注意在内积计算时，由于对比学习训练时有temperature的概念
# 需要乘上模型logit_scale.exp()，我们的预训练模型logit_scale均为4.6052，所以这里乘以100
# 对于用户自己的ckpt，请使用torch.load载入后，查看ckpt['state_dict']['module.logit_scale']或ckpt['state_dict']['logit_scale']
logits_per_image = 100 * image_features @ text_features.t()
print(logits_per_image.softmax(dim=-1)) # 图文相似概率: [[1.2252e-03, 5.2874e-02, 6.7116e-04, 9.4523e-01]]
```

可以看到，给出的图文相似概率，和[Readme中快速使用部分](https://github.com/OFA-Sys/Chinese-CLIP#api快速上手)，基于Pytorch同一个模型计算的结果基本一致，证明了ONNX模型特征计算的正确性，而**ONNX模型的特征计算速度相比Pytorch更有优势**（详见下文）。

## 转换和运行TensorRT模型

### 转换模型（请先阅读ONNX模型转换部分）

相比ONNX模型，TensorRT模型具有更快的推理速度。如前文所说，我们准备Chinese-CLIP的TensorRT格式模型，是用刚刚得到的ONNX格式模型进一步转化而来。将ONNX转换为TensorRT格式的代码，请参见`cn_clip/deploy/onnx_to_tensorrt.py`。仍然以ViT-B-16规模为例，利用刚刚得到的ONNX模型`vit-b-16.txt.fp16.onnx`和`vit-b-16.img.fp16.onnx`，在`Chinese-CLIP/`下运行如下代码：

```bash
export PYTHONPATH=${PYTHONPATH}:`pwd`/cn_clip
# 如前文，${DATAPATH}请根据实际情况替换
python cn_clip/deploy/onnx_to_tensorrt.py \
       --model-arch ViT-B-16 \
       --convert-text \
       --text-onnx-path ${DATAPATH}/deploy/vit-b-16.txt.fp16.onnx \
       --convert-vision \
       --vision-onnx-path ${DATAPATH}/deploy/vit-b-16.img.fp16.onnx \
       --save-tensorrt-path ${DATAPATH}/deploy/vit-b-16 \
       --fp16
```

其中各配置项定义如下：
+ `model-arch`: 模型规模，选项包括`["RN50", "ViT-B-16", "ViT-L-14", "ViT-L-14-336", "ViT-H-14"]`，各规模细节详见[Readme](https://github.com/OFA-Sys/Chinese-CLIP#模型规模--下载链接)
+ `convert-text`和`convert-vision`: 指定是否转换文本侧和图像侧模型
+ `text-onnx-path`: 指定文本侧ONNX模型文件路径，需要与`model-arch`指定的模型规模对应
+ `vision-onnx-path`: 指定图像侧ONNX模型文件路径，需要与`model-arch`指定的模型规模对应
+ `save-tensorrt-path`: 指定输出TensorRT格式模型的路径（前缀）。完成转换后，代码将分别输出文本侧和图像侧的TensorRT格式编码模型文件，该参数即指定了以上输出文件的路径前缀
+ `fp16`: 指定转换FP16精度的TensorRT格式模型

整个过程根据模型规模不同，耗时几分钟到十几分钟不等。运行此代码转换完成后，将得到以下的log输出：
```
Finished ONNX to TensorRT conversion...
>>> The text FP16 TensorRT model is saved at ${DATAPATH}/deploy/vit-b-16.txt.fp16.trt
>>> The vision FP16 TensorRT model is saved at ${DATAPATH}/deploy/vit-b-16.img.fp16.trt
```

上面示例代码执行结束后，我们使用ONNX模型，得到了ViT-B-16规模，Chinese-CLIP文本侧和图像侧的TensorRT格式模型，可以用于提取图文特征。输出TensorRT模型的路径以运行脚本时的`save-tensorrt-path`为前缀，后面依次拼上`.img`/`.txt`、`.fp16`、`.trt`。我们使用两个输出文件`vit-b-16.txt.fp16.trt`和`vit-b-16.img.fp16.trt`。

### 运行模型

在运行TensorRT模型时，如果转换和运行不是在同一个环境下，请注意运行模型的环境TensorRT库版本与转换保持一致，避免报错。

#### 提取图像侧特征

类似于ONNX模型运行的流程，我们在`Chinese-CLIP/`目录下，使用以下的示例代码，读取刚刚转换好的ViT-B-16规模TensorRT图像侧模型`vit-b-16.img.fp16.trt`，并为Readme中示例的[皮卡丘图片](examples/pokemon.jpeg)提取图像侧特征。和ONNX模型一样，这里转换好的TensorRT模型也只接受batch大小为1的输入，即一次调用只处理一张输入图片

```python
# 完成必要的import（下文省略）
from cn_clip.deploy.tensorrt_utils import TensorRTModel
from PIL import Image
import numpy as np
import torch
import argparse
import cn_clip.clip as clip
from clip import load_from_name, available_models
from clip.utils import _MODELS, _MODEL_INFO, _download, available_models, create_model, image_transform

# 载入TensorRT图像侧模型（**请替换${DATAPATH}为实际的路径**）
img_trt_model_path="${DATAPATH}/deploy/vit-b-16.img.fp16.trt"
img_trt_model = TensorRTModel(img_trt_model_path)

# 预处理图片
model_arch = "ViT-B-16" # 这里我们使用的是ViT-B-16规模，其他规模请对应修改
preprocess = image_transform(_MODEL_INFO[model_arch]['input_resolution'])
# 示例皮卡丘图片，预处理后得到[1, 3, 分辨率, 分辨率]尺寸的Torch Tensor
image = preprocess(Image.open("examples/pokemon.jpeg")).unsqueeze(0).cuda()

# 用TensorRT模型计算图像侧特征
image_features = img_trt_model(inputs={'image': image})['unnorm_image_features'] # 未归一化的图像特征
image_features /= image_features.norm(dim=-1, keepdim=True) # 归一化后的Chinese-CLIP图像特征，用于下游任务
print(image_features.shape) # Torch Tensor shape: [1, 特征向量维度]
```

#### 提取文本侧特征

与图像侧类似，我们用如下代码完成文本侧TensorRT模型的载入与特征计算，与图像侧相同，文本侧ONNX部署模型只接受batch大小为1的输入，即一次调用只处理一条输入文本。TensorRT接受的文本序列长度和用于转换的ONNX模型一致，请参见ONNX转换时的context-length参数。我们为4条候选文本依次计算ViT-B-16规模模型的文本特征。import相关代码与上文相同，这里省略：

```python
# 载入TensorRT文本侧模型（**请替换${DATAPATH}为实际的路径**）
txt_trt_model_path="${DATAPATH}/deploy/vit-b-16.txt.fp16.trt"
txt_trt_model = TensorRTModel(txt_trt_model_path)

# 为4条输入文本进行分词。序列长度指定为52，需要和转换ONNX模型时保持一致（参见ONNX转换时的context-length参数）
text = clip.tokenize(["杰尼龟", "妙蛙种子", "小火龙", "皮卡丘"], context_length=52).cuda()

# 用TensorRT模型依次计算文本侧特征
text_features = []
for i in range(len(text)):
    # 未归一化的文本特征
    text_feature = txt_trt_model(inputs={'text': torch.unsqueeze(text[i], dim=0)})['unnorm_text_features']
    text_features.append(text_feature)
text_features = torch.squeeze(torch.stack(text_features), dim=1) # 4个特征向量stack到一起
text_features = text_features / text_features.norm(dim=1, keepdim=True) # 归一化后的Chinese-CLIP文本特征，用于下游任务
print(text_features.shape) # Torch Tensor shape: [4, 特征向量维度]
```

#### 计算图文相似度

TensorRT模型产出的归一化图文特征，内积后softmax即可计算相似度（需要考虑`logit_scale`），与原始Pytorch和ONNX模型的操作均相同，参见以下代码。import相关代码与上文相同，这里省略：
```python
# 内积后softmax
# 注意在内积计算时，由于对比学习训练时有temperature的概念
# 需要乘上模型logit_scale.exp()，我们的预训练模型logit_scale均为4.6052，所以这里乘以100
# 对于用户自己的ckpt，请使用torch.load载入后，查看ckpt['state_dict']['module.logit_scale']或ckpt['state_dict']['logit_scale']
logits_per_image = 100 * image_features @ text_features.t()
print(logits_per_image.softmax(dim=-1)) # 图文相似概率: [[1.2475e-03, 5.3037e-02, 6.7583e-04, 9.4504e-01]]
```

可以看到，TensorRT模型给出的图文相似概率，和[Readme中快速使用部分](https://github.com/OFA-Sys/Chinese-CLIP#api快速上手)基于Pytorch的同一份模型、以及上文ONNX模型计算的结果都基本一致，证明了TensorRT模型特征计算的正确性，而TensorRT模型的特征计算速度，**相比前两者都更胜一筹**（详见下文）。