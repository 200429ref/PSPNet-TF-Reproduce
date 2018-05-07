## Unofficial Reproduce of PSPNet.

Pyramid Scene Parsing Network [paper](https://arxiv.org/abs/1612.01105) and [official github](https://github.com/hszhao/PSPNet).

Here is an unofficial re-implementation of PSPNet (from training to test) in pure Tensorflow library, where the most interesting point is the implementation of ***sync batch norm*** across multiple gpus (see `./model/utils_mg.py`). Tested on TF1.1 and TF1.4.

### Sync Batch Norm
When concerning image segmentation, batch size is usually limited. Small batch size will harm the performance. Using multi-GPU to increase the batch size does not help because the statistics are computed with each GPU (by default in all DL libraries). More discussion can be found [here](https://github.com/tensorflow/tensorflow/issues/7439) and [here](https://github.com/torch/nn/issues/1071).

This repo resolves this problem in pure python and pure Tensorflow and the main idea is located in [model/utils_mg.py](https://github.com/holyseven/PSPNet-TF-Reproduce/blob/master/model/utils_mg.py)

_I do not know if this is the first implementation of sync batch norm in Tensorflow, but there is already an implementation in [PyTorch](http://hangzh.com/PyTorch-Encoding/syncbn.html) and [some applications](https://github.com/CSAILVision/semantic-segmentation-pytorch)._

### L2-SP regularization
L2-SP regularization is a variant of L2 regularization and sets the pre-trained model as reference, instead of the origin like L2 does. More details can be found in the [paper](https://arxiv.org/abs/1802.01483) and [code](https://github.com/holyseven/TransferLearningClassification).

### Numerical Results

Train on extra+train (20000+2975 images) then on train set (2975), test on val set (500):

- w|o ms: 80.1
- w|  ms: 81.2

test on [test set](https://www.cityscapes-dataset.com/method-details/?submissionID=1148) (1525):
- w| ms: 80.3

### Qualitative Results

![](https://github.com/holyseven/PSPNet-TF-Reproduce/blob/master/results_examples/berlin_000000_000019_leftImg8bit.png)  |  ![](https://github.com/holyseven/PSPNet-TF-Reproduce/blob/master/results_examples/berlin_000000_000019_30k-extra-wd1-0_coloring.png)

### Training

(This is the process for reproducing PSPNet on Cityscapes and Pascal VOC. For fine-tuning on other databases, some additional changes for reading images and labels in `reader_segmentation.py` may be needed.)

1. Before training, it would be better to change the path of database in ./database/reader_segmentation.py, function `find_data_path`.

2. Download resnet_v1_101.ckpt. The script in `./z_pretrained_weights/` can help do it.

3. Run this training script under `./run_pspmg/` (4*2=8 examples in a batch, L2-SP regularization) for Cityscapes which follows an evaluation without multi-scale test. (details of all hyperparameters)

`CUDA_VISIBLE_DEVICES=0,1,2,3 python ./train.py --subsets_for_training 'train' --ema_decay 0.9 --gpu_num 4 --network 'pspnet' --structure_in_paper 0 --train_like_in_paper 0 --initializer 'he' --color_switch 0 --poly_lr 1 --data_type 32 --lrn_rate 0.01 --weight_decay_mode 1 --weight_decay_rate 0.0001 --weight_decay_rate2 0.0001 --batch_size 2 --train_max_iter 50000 --snapshot 25000 --momentum 0.9 --random_scale 1 --scale_min 0.5 --scale_max 2.0 --random_rotate 0 --database 'Cityscapes' --server $s --fine_tune_filename '../z_pretrained_weights/resnet_v1_101.ckpt' --train_image_size 864 --test_image_size 864 --optimizer 'mom' --data_type 32 --log_dir only-resnet`

4. An example of training script for Pascal VOC (details of all hyperparameters):

`CUDA_VISIBLE_DEVICES=0,1,2,3 python ./train.py --subsets_for_training 'train' --ema_decay 0.9 --gpu_num 4 --network 'pspnet' --structure_in_paper 0 --train_like_in_paper 0 --initializer 'he' --color_switch 0 --poly_lr 1 --data_type 32 --lrn_rate 0.01 --weight_decay_mode 1 --weight_decay_rate 0.001 --weight_decay_rate2 0.0001 --batch_size 4 --train_max_iter 30000 --snapshot 15000 --momentum 0.9 --random_scale 1 --scale_min 0.5 --scale_max 2.0 --random_rotate 1 --database 'SBD' --server $s --fine_tune_filename '../z_pretrained_weights/resnet_v1_101.ckpt' --train_image_size 480 --test_image_size 480 --optimizer 'mom' --data_type 32 --log_dir only-resnet`

5. Under `./run_pspmg/`, run the script for an evaluation with ms test:

`CUDA_VISIBLE_DEVICES=0 python ./predict.py --server $s --database 'SBD' --structure_in_paper 0 --save_prediction 1 --color_switch 0 --test_image_size 480 --mode 'test' --weights_ckpt './log/SBD/only-resnet-1/snapshot/model.ckpt-30000' --coloring 0 --mirror 1 --ms 1`

6. Infer one image will be added soon.