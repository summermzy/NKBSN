# Joint embedding VQA model based on dynamic word vector

It is the code reprository for "Joint embedding VQA model based on dynamic word vector", in which the model used a pre-trained ELMO model to replace the static word embedding commonly used by other VQA models.

![basic structure of N-KBSN model](misc/N-KBSN.jpg)

## Table of Contents
0. [Install](#Install)
0. [Method](#Method)
0. [Experiment setup](#Experiment-setup)
0. [Training](#Training)
0. [Validation and Testing](#Validation-and-Testing)
0. [Pretrained models](#Pretrained-models)

## Install

#### Software and Hardware Requirements

You may need a machine with at least **1 GPU (>= 8GB)**, **20GB memory** and **50GB free disk space**.  We strongly recommend to use a SSD drive to guarantee high-speed I/O.

You should first install some necessary packages.

1. Install [Python](https://www.python.org/downloads/) >= 3.5
2. Install [Cuda](https://developer.nvidia.com/cuda-toolkit) >= 9.0 and [cuDNN](https://developer.nvidia.com/cudnn)
3. Install all required packages as following: 
	```bash
	$ pip install -r requirements.txt
	```


#### Setup 
  The image features are extracted using the [bottom-up-attention](https://github.com/peteanderson80/bottom-up-attention) strategy, with each image being represented as an dynamic number (from 10 to 100) of 2048-D features. We store the features for each image in a `.npz` file. You can prepare the visual features by yourself or download the extracted features from [OneDrive](https://awma1-my.sharepoint.com/:f:/g/personal/yuz_l0_tn/EsfBlbmK1QZFhCOFpr4c5HUBzUV0aH2h1McnPG1jWAxytQ?e=2BZl8O) or [BaiduYun](https://pan.baidu.com/s/1C7jIWgM3hFPv-YXJexItgw#list/path=%2F). The downloaded files contains three files: **train2014.tar.gz, val2014.tar.gz, and test2015.tar.gz**, corresponding to the features of the train/val/test images for *VQA-v2*, respectively. You should place them as follows:

```angular2html
|-- datasets
	|-- coco_extract
	|  |-- train2014.tar.gz
	|  |-- val2014.tar.gz
	|  |-- test2015.tar.gz
 ```

Besides, we use the VQA samples from the [visual genome dataset](http://visualgenome.org/) to expand the training samples. Similar to existing strategies, we preprocessed the samples by two rules:

1. Select the QA pairs with the corresponding images appear in the MSCOCO train and *val* splits.
2. Select the QA pairs with the answer appear in the processed answer list (occurs more than 8 times in whole *VQA-v2* answers).

For convenience, we provide our processed vg questions and annotations files, you can download them from [OneDrive](https://awma1-my.sharepoint.com/:f:/g/personal/yuz_l0_tn/EmVHVeGdck1IifPczGmXoaMBFiSvsegA6tf_PqxL3HXclw) or [BaiduYun](https://pan.baidu.com/s/1QCOtSxJGQA01DnhUg7FFtQ#list/path=%2F), and place them as follow:


```angular2html
|-- datasets
	|-- vqa
	|  |-- VG_questions.json
	|  |-- VG_annotations.json
```

We use a pre-trained [ELMO](https://allennlp.org/elmo) model to get language features. [weights][w_elmo] and [options][opt_elmo] should be placed as follow:

```angular2html
|-- utils
	|-- elmo
	|  |-- elmo_2x4096_512_2048cnn_2xhighway_options.json
	|  |-- elmo_2x4096_512_2048cnn_2xhighway_weights.hdf5
```
[w_elmo]: https://s3-us-west-2.amazonaws.com/allennlp/models/elmo/2x4096_512_2048cnn_2xhighway/elmo_2x4096_512_2048cnn_2xhighway_weights.hdf5 "weights of elmo"
[opt_elmo]: https://s3-us-west-2.amazonaws.com/allennlp/models/elmo/2x4096_512_2048cnn_2xhighway/elmo_2x4096_512_2048cnn_2xhighway_options.json "options of elmo"

After that, you can run the following script to setup all the needed configurations for the experiments

```bash
$ sh setup.sh
```

Running the script will: 

1. Download the QA files for [VQA-v2](https://visualqa.org/download.html).
2. Unzip the bottom-up features

Finally, the `datasets` folders will have the following structure:

```angular2html
|-- datasets
	|-- coco_extract
	|  |-- train2014
	|  |  |-- COCO_train2014_...jpg.npz
	|  |  |-- ...
	|  |-- val2014
	|  |  |-- COCO_val2014_...jpg.npz
	|  |  |-- ...
	|  |-- test2015
	|  |  |-- COCO_test2015_...jpg.npz
	|  |  |-- ...
	|-- vqa
	|  |-- v2_OpenEnded_mscoco_train2014_questions.json
	|  |-- v2_OpenEnded_mscoco_val2014_questions.json
	|  |-- v2_OpenEnded_mscoco_test2015_questions.json
	|  |-- v2_OpenEnded_mscoco_test-dev2015_questions.json
	|  |-- v2_mscoco_train2014_annotations.json
	|  |-- v2_mscoco_val2014_annotations.json
	|  |-- VG_questions.json
	|  |-- VG_annotations.json

```

## Training

The following script will start training with the default hyperparameters:

```bash
$ python3 run.py --RUN='train'
```
All checkpoint files will be saved to:

```
ckpts/ckpt_<VERSION>/epoch<EPOCH_NUMBER>.pkl
```

and the training log file will be placed at:

```
results/log/log_run_<VERSION>.txt
```

To add：

1. ```--VERSION=str```, e.g.```--VERSION='small_model'``` to assign a name for your this model.

2. ```--GPU=str```, e.g.```--GPU='2'``` to train the model on specified GPU device.

3. ```--NW=int```, e.g.```--NW=8``` to accelerate I/O speed.

4. ```--MODEL={'small', 'large'}```  ( Warning: The large model will consume more GPU memory, maybe [Multi-GPU Training and Gradient Accumulation](#Multi-GPU-Training-and-Gradient-Accumulation) can help if you want to train the model with limited GPU memory.)

5. ```--SPLIT={'train', 'train+val', 'train+val+vg'}``` can combine the training datasets as you want. The default training split is ```'train+val+vg'```.  Setting ```--SPLIT='train'```  will trigger the evaluation script to run the validation score after every epoch automatically.

6. ```--RESUME=True``` to start training with saved checkpoint parameters. In this stage, you should assign the checkpoint version```--CKPT_V=str``` and the resumed epoch number ```CKPT_E=int```.

7. ```--MAX_EPOCH=int``` to stop training at a specified epoch number.

8. ```--PRELOAD=True``` to pre-load all the image features into memory during the initialization stage (Warning: needs extra 25~30GB memory and 30min loading time from an HDD drive).


####  Multi-GPU Training and Gradient Accumulation

We recommend to use the GPU with at least 8 GB memory, but if you don't have such device, don't worry, we provide two ways to deal with it:

1. _Multi-GPU Training_: 

    If you want to accelerate training or train the model on a device with limited GPU memory, you can use more than one GPUs:

	Add ```--GPU='0, 1, 2, 3...'```

    The batch size on each GPU will be adjusted to `BATCH_SIZE`/#GPUs automatically.

2. _Gradient Accumulation_: 

    If you only have one GPU less than 8GB, an alternative strategy is provided to use the gradient accumulation during training:
	
	Add ```--ACCU=n```  
	
    This makes the optimizer accumulate gradients for`n` small batches and update the model weights at once. It is worth noting that  `BATCH_SIZE` must be divided by ```n``` to run this mode correctly. 


## Validation and Testing

**Warning**: If you train the model use ```--MODEL``` args or multi-gpu training, it should be also set in evaluation.


Offline evaluation only support the VQA 2.0 *val* split. If you want to evaluate on the VQA 2.0 *test-dev* or *test-std* split, please see [Online Evaluation](#Online-Evaluation).

There are two ways to start:

(Recommend)

```bash
$ python3 run.py --RUN='val' --CKPT_V=str --CKPT_E=int
```

or use the absolute path instead:

```bash
$ python3 run.py --RUN='val' --CKPT_PATH=str
```

The evaluations of both the VQA 2.0 *test-dev* and *test-std* splits are run as follows:

```bash
$ python3 run.py --RUN='test' --CKPT_V=str --CKPT_E=int
```

Result files are stored in ```results/result_test/result_run_<'PATH+random number' or 'VERSION+EPOCH'>.json```

## Results
In our expriments, we compare six models: `baseline(random)`, `baseline (w2v)`, `baseline (glove)`, `N — KBSN(s)`, `N — KBSN(m)`, `N — KBSN(l)`.

 In order to further explore the best Elmo parameters, three different pre-trained Elmo models with different parameters are selected in this experiment, which are _N — KBSN(s)_/_N — KBSN(m)_/_N — KBSN(l)_. The parameters, hidden layer size, output size and Elmo size of LSTM are shown in the table.

_Model_ | Parameters (M) | LSTM Size | Output Size | ELMO Size
:-: | :-: | :-: | :-: | :-:
 _ELMO(s)_|13.6|1024|128|256|
 _ELMO(m)_|28.0|2048|256|512|
 _ELMO(l)_|93.6|4096|512|1024|

 Statistics of several word vectors are shown in the table.
 _name_ | Pre-training corpus (size) | Word vector dimension | Number of word vectors 
:-: | :-: | :-: | :-: 
 _word2vec_|Google News (100 billion words)|300|3 million|
 _Glove_|Wikipedia 2014 + Gigaword 5 (Six billion words)|300|400 thousand|
 _ELMO(s)_||256||
 _ELMO(m)_|WMT 2011 (800 million words)|512||
 _ELMO(l)_||1024||

The performance of the two models on the *Val set*  is reported as follows:
_Model_ | Overall | Yes/No | Number | Other
:-: | :-: | :-: | :-: | :-:
_baseline(random)_ | 62.34 | 78.77 | 41.92 | 55.27| 
_baseline (w2v)_ | 64.37| 81.89 | 44.51 | 56.31|
_baseline (glove)_|66.73|84.56|49.52|57.72|
_N — KBSN(s)_|67.27|84.76|49.31|58.73|
_N — KBSN(m)_|67.55|85.03|49.62|59.01|
_N — KBSN(l)_|**67.72**|**85.22**|**49.63**|**59.20**|

## Acknowlegement
This project get a lot helps from the open-source MCAN-VQA project. If you are interested in the GloVe version of VQA, you can find a good example from [here.](https://github.com/MILVLG/mcan-vqa)