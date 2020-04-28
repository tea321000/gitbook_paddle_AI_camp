# Day4-《青春有你2》选手识别

###  一、任务简介

图像分类是计算机视觉的重要领域，它的目标是将图像分类到预定义的标签。近期，许多研究者提出很多不同种类的神经网络，并且极大的提升了分类算法的性能。本文以自己创建的数据集：青春有你2中选手识别为例子，介绍如何使用PaddleHub进行图像分类任务。

![](https://ai-studio-static-online.cdn.bcebos.com/dbaf5ba718f749f0836c342fa67f6d7954cb89f3796b4c8b981a03a1635f2fbe)

这个题目可以说是对新人来说比较有难度的题目，尤其是之前没有炼过丹的同学。一言蔽之，这个任务是使用自定义输入（自己定义的数据集）加上使用其他数据集已经训练好的预训练模型进行参数调优`fine tune`的过程。

这样做的好处是当自定义的数据集与先前训练的数据分布相差不大时（比如先前使用大量人脸训练的预训练模型来调优新的人脸数据），模型的参数通过先前的训练已经很接近真实的数据分布参数，而且训练出的底层滤波器特征可以套用到新的数据集中，使得收敛的速度更快，在更短的时间内使用更少的算力达到更优的性能。

太长不看版：训练的过程就是使用已知数据分布（已有的数据集）推广未知数据分布（模型推断时的图片输入）的过程，因此当模型固定时，你的数据输入决定了模型推断时的性能。

接下来进行代码的讲解。用户基本只需要对数据集部分的代码进行更改即可。

```bash
#CPU环境启动请务必执行该指令
# %set_env CPU_NUM=1 
```

```text
#安装paddlehub
# !pip install paddlehub==1.6.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 二、任务实践

#### Step1、基础工作

加载数据文件

导入python包

```bash
# !unzip dataset/pics.zip -d dataset/pics
```

```python
import paddlehub as hub
!hub install xception71_imagenet==1.0.0
```

#### Step2、加载预训练模型

在这里我没有选择resnet50的模型，而是使用了Google inception的预训练模型[xception71\_imagenet](https://www.paddlepaddle.org.cn/hubdetail?name=xception71_imagenet&en_category=ImageClassification)模型：

```python
module = hub.Module(name="xception71_imagenet")
```

#### Step3、数据准备

接着需要加载图片数据集。我们使用自定义的数据进行体验，请查看[适配自定义数据](https://github.com/PaddlePaddle/PaddleHub/wiki/PaddleHub%E9%80%82%E9%85%8D%E8%87%AA%E5%AE%9A%E4%B9%89%E6%95%B0%E6%8D%AE%E5%AE%8C%E6%88%90FineTune)In\[23\]

```python
from paddlehub.dataset.base_cv_dataset import BaseCVDataset
import os
import random


folder_base="pics"
text_base="dataset"
folder_name=['yushuxin','xujiaqi','zhaoxiaotang','anqi','wangchengxuan']
def write_list():
    train_folder=[]
    validate_folder=[]
    for folder in folder_name:
        pics=os.listdir(os.path.join(text_base,folder_base,folder))
        # 打乱图片
        random.shuffle(pics)
        
        # 每类取20张作为验证集
        train_folder.append(pics[:-20])
        validate_folder.append(pics[-20:])

    # 生成训练集 验证集配置文件
    with open(os.path.join(text_base,'train_list.txt'),'w') as f:
        for index,pics in enumerate(train_folder):
            for pic in pics:
                f.write(os.path.join(folder_base,folder_name[index],pic)+' '+str(index)+'\n')
        f.close()

    with open(os.path.join(text_base,'validate_list.txt'),'w') as f:
        for index,pics in enumerate(validate_folder):
            for pic in pics:
                f.write(os.path.join(folder_base,folder_name[index],pic)+' '+str(index)+'\n')
        f.close()

    with open(os.path.join(text_base,'test_list.txt'),'w') as f:
        # 将test的相对路径先改变 不然训练会出错
        f.write('test/yushuxin.jpg 0\n');
        f.write('test/xujiaqi.jpg 1\n');
        f.write('test/zhaoxiaotang.jpg 2\n');
        f.write('test/anqi.jpg 3\n');
        f.write('test/wangchengxuan.jpg 4');
        f.close()
        
class DemoDataset(BaseCVDataset):	
   def __init__(self):	
       # 数据集存放位置
       
       self.dataset_dir = "dataset"
       super(DemoDataset, self).__init__(
           base_path=self.dataset_dir,
           train_list_file="train_list.txt",
           validate_list_file="validate_list.txt",
           test_list_file="test_list.txt",
           label_list_file="label_list.txt",
           )
write_list()
dataset = DemoDataset()
```

在这里我导入了os库使用文件写入的方式随机打乱我的数据集并每类取20张作为验证集进行模型调优，而我的每类只有7 80张的图片（每类手动收集20~30张，通过数据增强做到7 80张）。之所以要取这么多张做验证集，是因为这些预训练模型往往都带有`batch normalized`层（bn层），这个层的本质就是将数据偏移`shift`到一个合适的分布中进行训练，使得模型能更快收敛，但对于小样本（比如我们这里只有几十张图片）时很容易出现过拟合。因此为了不让模型太快的过拟合，需要将验证集取的大一些。

#### Step4、生成数据读取器

接着生成一个图像分类的reader，reader负责将dataset的数据进行预处理，接着以特定格式组织并输入给模型进行训练。

当我们生成一个图像分类的reader时，需要指定输入图片的大小In\[24\]

```python
data_reader = hub.reader.ImageClassificationReader(
    image_width=module.get_expected_image_width(),
    image_height=module.get_expected_image_height(),
    images_mean=module.get_pretrained_images_mean(),
    images_std=module.get_pretrained_images_std(),
    dataset=dataset)
```

```bash
[2020-04-26 09:03:13,639] [    INFO] - Dataset label map = {'虞书欣': 0, '许佳琪': 1, '赵小棠': 2, '安崎': 3, '王承渲': 4}
```

#### Step5、配置策略

在进行Finetune前，我们可以设置一些运行时的配置，例如如下代码中的配置，表示：

* `use_cuda`：设置为False表示使用CPU进行训练。如果您本机支持GPU，且安装的是GPU版本的PaddlePaddle，我们建议您将这个选项设置为True；
* `epoch`：迭代轮数；
* `batch_size`：每次训练的时候，给模型输入的每批数据大小为32，模型训练时能够并行处理批数据，因此batch\_size越大，训练的效率越高，但是同时带来了内存的负荷，过大的batch\_size可能导致内存不足而无法训练，因此选择一个合适的batch\_size是很重要的一步；
* `log_interval`：每隔10 step打印一次训练日志；
* `eval_interval`：每隔50 step在验证集上进行一次性能评估；
* `checkpoint_dir`：将训练的参数和数据保存到cv\_finetune\_turtorial\_demo目录中；
* `strategy`：使用DefaultFinetuneStrategy策略进行finetune；

更多运行配置，请查看[RunConfig](https://github.com/PaddlePaddle/PaddleHub/wiki/PaddleHub-API:-RunConfig)

同时PaddleHub提供了许多优化策略，如`AdamWeightDecayStrategy`、`ULMFiTStrategy`、`DefaultFinetuneStrategy`等，详细信息参见[策略](https://github.com/PaddlePaddle/PaddleHub/wiki/PaddleHub-API:-Strategy)In\[25\]

```python
config = hub.RunConfig(
    use_cuda=True,                              #是否使用GPU训练，默认为False；
    num_epoch=5,                                #Fine-tune的轮数；
    checkpoint_dir="cv_finetune_turtorial_demo",#模型checkpoint保存路径, 若用户没有指定，程序会自动生成；
    batch_size=40,                              #训练的批大小，如果使用GPU，请根据实际情况调整batch_size；
    eval_interval=10,                           #模型评估的间隔，默认每100个step评估一次验证集；
    strategy=hub.finetune.strategy.DefaultFinetuneStrategy())  #Fine-tune优化策略；
```

```bash
[2020-04-26 09:03:13,644] [    INFO] - Checkpoint dir: cv_finetune_turtorial_demo
```

#### Step6、组建Finetune Task

有了合适的预训练模型和准备要迁移的数据集后，我们开始组建一个Task。

由于该数据设置是一个二分类的任务，而我们下载的分类module是在ImageNet数据集上训练的千分类模型，所以我们需要对模型进行简单的微调，把模型改造为一个二分类模型：

1. 获取module的上下文环境，包括输入和输出的变量，以及Paddle Program；
2. 从输出变量中找到特征图提取层feature\_map；
3. 在feature\_map后面接入一个全连接层，生成Task；

In\[26\]

```python
input_dict, output_dict, program = module.context(trainable=True)
img = input_dict["image"]
feature_map = output_dict["feature_map"]
feed_list = [img.name]

task = hub.ImageClassifierTask(
    data_reader=data_reader,
    feed_list=feed_list,
    feature=feature_map,
    num_classes=dataset.num_labels,
    config=config)
```

```bash
[2020-04-26 09:03:13,786] [    INFO] - 638 pretrained paramaters loaded by PaddleHub
```

#### Step5、开始Finetune

我们选择`finetune_and_eval`接口来进行模型训练，这个接口在finetune的过程中，会周期性的进行模型效果的评估，以便我们了解整个训练过程的性能变化。In\[27\]

```python
run_states = task.finetune_and_eval()
```

```bash
2020-04-26 09:03:14,531-WARNING: paddle.fluid.layers.py_reader() may be deprecated in the near future. Please use paddle.fluid.io.DataLoader.from_generator() instead.
[2020-04-26 09:03:14,850] [    INFO] - Strategy with scheduler: {'warmup': 0.0, 'linear_decay': {'start_point': 1.0, 'end_learning_rate': 0.0}, 'noam_decay': False, 'discriminative': {'blocks': 0, 'factor': 2.6}, 'gradual_unfreeze': 0, 'slanted_triangle': {'cut_fraction': 0.0, 'ratio': 32}}, regularization: {'L2': 0.001, 'L2SP': 0.0, 'weight_decay': 0.0} and clip: {'GlobalNorm': 0.0, 'Norm': 0.0}
[2020-04-26 09:03:35,033] [    INFO] - Try loading checkpoint from cv_finetune_turtorial_demo/ckpt.meta
[2020-04-26 09:03:35,035] [    INFO] - PaddleHub model checkpoint not found, start from scratch...
[2020-04-26 09:03:35,150] [    INFO] - PaddleHub finetune start
[2020-04-26 09:03:47,510] [   TRAIN] - step 10 / 42: loss=0.24024 acc=0.97500 [step/sec: 0.57]
[2020-04-26 09:03:47,511] [    INFO] - Evaluation on dev dataset start
2020-04-26 09:03:48,333-WARNING: paddle.fluid.layers.py_reader() may be deprecated in the near future. Please use paddle.fluid.io.DataLoader.from_generator() instead.
share_vars_from is set, scope is ignored.
[2020-04-26 09:03:52,060] [    EVAL] - [dev dataset evaluation result] loss=0.78752 acc=0.72500 [step/sec: 0.90]
[2020-04-26 09:03:52,061] [    EVAL] - best model saved to cv_finetune_turtorial_demo/best_model [best acc=0.72500]
2020-04-26 09:03:53,383-WARNING: paddle.fluid.layers.py_reader() may be deprecated in the near future. Please use paddle.fluid.io.DataLoader.from_generator() instead.
[2020-04-26 09:04:00,589] [   TRAIN] - step 20 / 42: loss=0.03348 acc=1.00000 [step/sec: 0.64]
[2020-04-26 09:04:00,590] [    INFO] - Evaluation on dev dataset start
[2020-04-26 09:04:03,909] [    EVAL] - [dev dataset evaluation result] loss=0.43007 acc=0.84167 [step/sec: 0.91]
[2020-04-26 09:04:03,910] [    EVAL] - best model saved to cv_finetune_turtorial_demo/best_model [best acc=0.84167]
[2020-04-26 09:04:12,277] [   TRAIN] - step 30 / 42: loss=0.00609 acc=1.00000 [step/sec: 0.70]
[2020-04-26 09:04:12,278] [    INFO] - Evaluation on dev dataset start
[2020-04-26 09:04:15,549] [    EVAL] - [dev dataset evaluation result] loss=0.27297 acc=0.88333 [step/sec: 0.92]
[2020-04-26 09:04:15,550] [    EVAL] - best model saved to cv_finetune_turtorial_demo/best_model [best acc=0.88333]
[2020-04-26 09:04:23,595] [   TRAIN] - step 40 / 42: loss=0.00310 acc=1.00000 [step/sec: 0.79]
[2020-04-26 09:04:23,596] [    INFO] - Evaluation on dev dataset start
[2020-04-26 09:04:26,920] [    EVAL] - [dev dataset evaluation result] loss=0.20050 acc=0.90000 [step/sec: 0.90]
[2020-04-26 09:04:26,922] [    EVAL] - best model saved to cv_finetune_turtorial_demo/best_model [best acc=0.90000]
[2020-04-26 09:04:29,548] [    INFO] - Evaluation on dev dataset start
[2020-04-26 09:04:32,778] [    EVAL] - [dev dataset evaluation result] loss=0.19163 acc=0.89167 [step/sec: 0.93]
[2020-04-26 09:04:32,779] [    INFO] - Load the best model from cv_finetune_turtorial_demo/best_model
[2020-04-26 09:04:33,288] [    INFO] - Evaluation on test dataset start
[2020-04-26 09:04:33,583] [    EVAL] - [test dataset evaluation result] loss=0.36848 acc=1.00000 [step/sec: 3.43]
[2020-04-26 09:04:33,585] [    INFO] - Saving model checkpoint to cv_finetune_turtorial_demo/step_45
[2020-04-26 09:04:34,922] [    INFO] - PaddleHub finetune finished.
```

这里虽然我最后的准确率没有达到百分之百，但这反而会使得模型不那么过拟合从而使推断性能好一点。

#### Step6、预测

当Finetune完成后，我们使用模型来进行预测，先通过以下命令来获取测试的图片

```python
import numpy as np
import matplotlib.pyplot as plt 
import matplotlib.image as mpimg

with open(os.path.join(text_base,'test_list.txt'),'w') as f:
        # 将test的相对路径再改回来
        f.write('dataset/test/yushuxin.jpg 0\n');
        f.write('dataset/test/xujiaqi.jpg 1\n');
        f.write('dataset/test/zhaoxiaotang.jpg 2\n');
        f.write('dataset/test/anqi.jpg 3\n');
        f.write('dataset/test/wangchengxuan.jpg 4');
        f.close()

with open("dataset/test_list.txt","r") as f:
    filepath = f.readlines()

data = [filepath[0].split(" ")[0],filepath[1].split(" ")[0],filepath[2].split(" ")[0],filepath[3].split(" ")[0],filepath[4].split(" ")[0]]

label_map = dataset.label_dict()
index = 0
run_states = task.predict(data=data)
results = [run_state.run_results for run_state in run_states]

for batch_result in results:
    print(batch_result)
    batch_result = np.argmax(batch_result, axis=2)[0]
    print(batch_result)
    for result in batch_result:
        index += 1
        result = label_map[result]
        print("input %i is %s, and the predict result is %s" %
              (index, data[index - 1], result))
```

```bash
[2020-04-26 09:04:34,934] [    INFO] - The best model has been loaded
[2020-04-26 09:04:34,935] [    INFO] - PaddleHub predict start
share_vars_from is set, scope is ignored.
[2020-04-26 09:04:35,332] [    INFO] - PaddleHub predict finished.
```

```bash
[array([[9.9538785e-01, 1.4281583e-03, 1.4344796e-03, 1.5812222e-03,
        1.6829165e-04],
       [1.5182982e-01, 4.8975992e-01, 3.1340811e-02, 1.5656298e-01,
        1.7050646e-01],
       [1.7682713e-01, 2.9195112e-01, 3.3537161e-01, 1.8833089e-01,
        7.5192782e-03],
       [2.2442464e-03, 1.8785913e-02, 1.1669159e-03, 9.7122192e-01,
        6.5810294e-03],
       [3.4868010e-04, 1.0628232e-03, 1.2720286e-04, 6.7610515e-04,
        9.9778521e-01]], dtype=float32)]
[0 1 2 3 4]
input 1 is dataset/test/yushuxin.jpg, and the predict result is 虞书欣
input 2 is dataset/test/xujiaqi.jpg, and the predict result is 许佳琪
input 3 is dataset/test/zhaoxiaotang.jpg, and the predict result is 赵小棠
input 4 is dataset/test/anqi.jpg, and the predict result is 安崎
input 5 is dataset/test/wangchengxuan.jpg, and the predict result is 王承渲
```

可以看到最终的测试结果都正确。但也可以看到这个模型还是明显有缺陷的，徐佳琪的概率明显大于其他四人，可以看到赵小棠的概率只比她高了一点，而我之前一开始训练的时候徐佳琪的概率也是很容易大于其他的类，其本质原因是

