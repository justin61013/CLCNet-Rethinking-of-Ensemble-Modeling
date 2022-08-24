# **CLCNet: Rethinking of Ensemble Modeling with Classification Confidence Network**
![](https://img.shields.io/badge/license-MIT-brightgreen) 
![](https://img.shields.io/badge/python-3.9-blue)
![](https://img.shields.io/badge/torch-1.10.0-yellowgreen)
[![](https://img.shields.io/badge/timm-0.5.4-orange)](https://github.com/rwightman/pytorch-image-models)

This repository is the official implementation of [CLCNet](https://arxiv.org/abs/2205.09612). 

### **Architecture Diagram**
![Architecture](https://github.com/yaoching0/CLCNet/blob/main/images/fig1-1.jpg)

## **Summary**
[1. Requirements](#1)

[2. Download ImageNet-1k validation set](#2)

[3. Generate dataset for training CLCNet](#3)

[4. ImageNet-1k cross validation](#4)

[5. (optional) Training and evaluation with custom data](#5)

[6. (optional) Tutorials](#6)

[7. Results](#7)

[8. Acknowledgements](#8)

## <a name="1"></a>**1. Requirements**

To install requirements:

```setup
pip install -r requirements.txt
```

Please make sure you have at least 8GB of free GPU memory.


## <a name="2"></a>**2. Download ImageNet-1k validation set**

We divided ImageNet-1k val set into five equal parts for subsequent cross validation: [Google Drive](https://drive.google.com/file/d/1dnFNH0LZfs_UpzDms5FBznupKph38pBV/view?usp=sharing)

Please unzip it and place it in the following path:

```ImageNet
[root of repository]\data\imagenet_splits_5\...
```

## <a name="3"></a>**3. Generate dataset for training CLCNet**
You can directly download the dataset already generated by EfficientNet-B4 (used in table1&2 in the paper) for CLCNet training: [Google Drive](https://drive.google.com/file/d/1RGDtI0Y78muoAK6H7CcD2S1qElfDlplN/view?usp=sharing), and put it in `[root of repository]\data\(named)effb4_output_ds_for_CLCNet.csv`.


### (optional) Or you can also generate by your own:
By default, we use [timm](https://github.com/rwightman/pytorch-image-models) pre-trained model (automatic download) to classify ImageNet-1k val and save the results for subsequent CLCNet training, and the classification results will be saved in the root directory of the repository:

```generate CLCNet dataset
python generate_data_for_CLCNet.py --model tf_efficientnet_b4
```

You can generate the classification results of the current shallow model and deep model separately, and then merge them as the training set of CLCNet for better performance:

```generate CLCNet dataset
python merge.py --path_of_csv1 <path_of_shallow/deep_model_result_csv> --path_of_csv2 <path_of_shallow/deep_model_result_csv> --output_path <path_of_merged_csv_output>
``` 

We support all classification models in the [timm](https://github.com/rwightman/pytorch-image-models) (see [model list](https://github.com/rwightman/pytorch-image-models/blob/master/results/results-imagenet.csv)), and you can also use your own weights trained with timm and generate training data for CLCNet on your own dataset.


```generate customized CLCNet dataset
python generate_data_for_CLCNet.py --data-path <path_of_custom_data> --model <timm_model_name> --model-weight <path_of_model_weight> --num_classes <num_cls> --output <path_of_output_result>
```


### The generated dataset will be a csv file in the following format:

```dataset format
[filename][sorted n-dim(ImageNet:n=1000) classifcation probilities][label (1 or 0)] 
[filename][sorted n-dim(ImageNet:n=1000) classifcation probilities][label (1 or 0)] 
...
```

## <a name="4"></a>**4. ImageNet-1k cross validation**

### Training
The path `[root of repository]\weights` already contains the CLCNet weights we trained (on dataset generated by EfficientNet-b4) as follow:

#### (optional) Training process
First, we use the dataset generated in the previous step to train CLCNet. We divide ImageNet-1k val into two parts of 40,000 samples and 10,000 samples each time, we will train and validate CLCNet using only the classification results generated by the part of 40,000 samples, repeat five times to make the part of 10,000 samples cover the entire ImageNet val, and generate five CLCNet weights. The weights will be saved in ```[root of repository]\weights``` by default:

```train
python train-imagenet-5-fold-cv.py --imagenet-split <path_of_step2_downloaded_ImageNet> --cls-output <path_of_previous_generated_csv>
```

Of course, you can also specify training hyperparameters:

```train
python train-imagenet-5-fold-cv.py --max-epochs 200 --batch-size 256 --imagenet-split <path_of_step2_downloaded_ImageNet> --cls-output <path_of_previous_generated_csv>
```

For more details, please see the parameter description in the py file.

### Evaluation

To evaluate cascade structure system on ImageNet-1k val, run:


```eval
python eval-imagenet-5-fold-cv.py --threshold 0.5 --shallow-model tf_efficientnet_b4 --shallow-model-FLOPs 4.2 --deep-model tf_efficientnet_b7 --deep-model-FLOPs 37
```

The system will automatically use the five CLCNet weights (`[root of repository]\weights`) we trained in the previous step and the weights pre-trained by timm (automatic download, we support all models in timm, see [model list](https://github.com/rwightman/pytorch-image-models/blob/master/results/results-imagenet.csv)) on ImageNet (`[root of repository]\data\imagenet_splits_5\...`) to calculate the accuracy and FLOPs under the _threshold_, and save them as a csv file in `[root of repository]\performance-result.csv`.

You can also use `--threshold-searching` to automatically calculate the accuracy and FLOPs under different thresholds between [0.05, 1.01].


#### (Important) Single model accuracy
To compare with single model (like EfficientNet-b4) in the system, the values (accuracy) in the [model list](https://github.com/rwightman/pytorch-image-models/blob/master/results/results-imagenet.csv) will be different under different hardware environments, please test separately.


You can make the model to be tested as the shallow model in the system, and set the threshold to a very small value. So the deep model will not be used, and the accuracy of the system is the accuracy of the shallow model:

```eval
python eval-imagenet-5-fold-cv.py --threshold -999 --shallow-model tf_efficientnet_b4
```

## <a name="5"></a>5. (optional) Training and evaluation with custom data

You can use the dataset obtained in step 3 to train CLCNet without cross validation (so you will only get one weight, and saved in the `[root of repository]\weights`):

```train
python train-custom-data.py --cls-output <path_of_previous_generated_csv>
```

You can also generate specific dataset of classification results of any dimension to train CLCNet using the above command, which will achieve better performance on your own tasks. You only need to make sure that the generated dataset is a csv file in the following format:
```
[filename][sorted n-dim classifcation probilities][label (1 or 0)] 
[filename][sorted n-dim classifcation probilities][label (1 or 0)] 
...
```

Use the above trained CLCNet weight to test performance of cascade structure system on custom dataset, and the shallow and deep models can be replaced by any timm [models](https://github.com/rwightman/pytorch-image-models/blob/master/results/results-imagenet.csv) trained by custom dataset:

```eval
python eval-custom-data.py --custom-data <custom_dataset_for_inference> --threshold 0.5 --num_classes <any_num_cls> --shallow-model tf_efficientnet_b4 --shallow-model-FLOPs 4.2 --shallow-model-weight <path_of_model_weight> --deep-model tf_efficientnet_b7 --deep-model-FLOPs 37 --deep-model-weight <path_of_model_weight> 
```

The system will infer each image in `<custom_dataset_for_inference>` and save the classification results in `[root of repository]\custom-data-result.csv`.


## <a name="6"></a>**6. (optional) Tutorials**

If you just want to use CLCNet to predict whether the classification results are correct, please check this tutorial:
- [Tutorial for CLCNet](https://github.com/yaoching0/CLCNet-Rethinking-of-Ensemble-Modeling/blob/main/Tutorial_for_CLCNet.ipynb)

## <a name="7"></a>**7. Results**

- Use the variant of EfficientNet as shallow model and deep model of cascade structure system, and compare with the original EfficientNet on ImageNet-1k:


| Model name         | Top 1 Accuracy  |   Threshold    | FLOPs per image |
---|:--:|:--:|:--:|
| EfficientNet-B0    |     75.40%  |       --       |      0.39B      |
| EfficientNet-B1    |     77.64%  |       --       |      0.7B      |
| CLCNet (S:B0+D:B4)    |   **77.74%**   |       0.19      |    **0.74B**       |
| EfficientNet-B2    |     78.73%  |       --       |      1.0B      |
| CLCNet (S:B0+D:B4)    |   79.06%   |      0.27       |     **0.996B**       |
| CLCNet (S:B0+D:B4)    |    **78.71%**  |       0.25      |     0.933B       |
| EfficientNet-B3    |     80.52%  |       --       |      1.8B      |
| CLCNet (S:B0+D:B4)    |    81.19%  |       0.43      |    **1.77B**        |
| CLCNet (S:B0+D:B4)    |    **80.50%**  |     0.39        |    1.42B        |
| EfficientNet-B4    |     82.00%  |       --       |      4.2B      |
| CLCNet (S:B4+D:B7)    |   **82.02%**   |      0.05       |    **4.27B**       |
| EfficientNet-B5    |     82.72%  |       --       |      9.9B      |
| CLCNet (S:B4+D:B7)   |  83.59%   |      0.45       |     **9.94B**       |
| CLCNet (S:B4+D:B7)   |    **82.75%**  |      0.27       |    6.1B        |
| EfficientNet-B6    |     83.30%  |       --       |      19B      |
| CLCNet (S:B4+D:B7)    |   83.88%   |      0.83       |    **18.58B**        |
| CLCNet (S:B4+D:B7)   |    **83.42%**  |      0.39       |    8.95B        |
| EfficientNet-B7    |     83.80%  |       --       |      37B      |
| CLCNet (S:B4+D:B7)    |    **83.88%**  |     0.83        |   18.58B         |

> Bold indicates the value of the metric is close to EfficientNet variant, S and D stand for shallow model and deep model respectively, and we use CLCNet to denote the cascade structure system using CLCNet.

- Use completely different models as shallow model and deep model, and compare with general ensemble modeling (GEM):


| Model name         | Top 1 Accuracy  |   Threshold    | FLOPs per image |
---|:--:|:--:|:--:|
| EfficientNet-B0    |     75.40%  |       --       |      0.39B      |
| EfficientNet-B1    |     77.64%  |       --       |      0.7B      |
| CLCNet (S:B0+D:B4)    |   **77.74%**   |       0.19      |    **0.74B**       |
| EfficientNet-B2    |     78.73%  |       --       |      1.0B      |
| CLCNet (S:B0+D:B4)    |   79.06%   |      0.27       |     **0.996B**       |
| CLCNet (S:B0+D:B4)    |    **78.71%**  |       0.25      |     0.933B       |
| EfficientNet-B3    |     80.52%  |       --       |      1.8B      |
| CLCNet (S:B0+D:B4)    |    81.19%  |       0.43      |    **1.77B**        |
| CLCNet (S:B0+D:B4)    |    **80.50%**  |     0.39        |    1.42B        |
| EfficientNet-B4    |     82.00%  |       --       |      4.2B      |
| CLCNet (S:B4+D:B7)    |   **82.02%**   |      0.05       |    **4.27B**       |
| EfficientNet-B5    |     82.72%  |       --       |      9.9B      |
| CLCNet (S:B4+D:B7)   |  83.59%   |      0.45       |     **9.94B**       |
| CLCNet (S:B4+D:B7)   |    **82.75%**  |      0.27       |    6.1B        |
| EfficientNet-B6    |     83.30%  |       --       |      19B      |
| CLCNet (S:B4+D:B7)    |   83.88%   |      0.83       |    **18.58B**        |
| CLCNet (S:B4+D:B7)   |    **83.42%**  |      0.39       |    8.95B        |
| EfficientNet-B7    |     83.80%  |       --       |      37B      |
| CLCNet (S:B4+D:B7)    |    **83.88%**  |     0.83        |   18.58B         |

## <a name="8"></a>**8. Acknowledgements**
Our implementation uses the source code from the following repositories:

- [PyTorch Image Models](https://github.com/rwightman/pytorch-image-models)

- [TabNet : Attentive Interpretable Tabular Learning](https://github.com/dreamquark-ai/tabnet)
