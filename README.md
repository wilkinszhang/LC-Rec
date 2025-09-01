# LC-Rec

This is the official PyTorch implementation for the paper:

> [Adapting Large Language Models by Integrating Collaborative Semantics for Recommendation](https://ieeexplore.ieee.org/abstract/document/10597986/)

## Overview

We propose **LC-Rec**, a new approach to integrate **L**anguage and **C**ollaborative semantics for improving LLMs in **Rec**ommender systems. To tackle the large gap between the language semantics modeled by LLMs and collaborative semantics implied by recommender systems, we make two major contributions in two aspects. For item indexing, we design a learning-based vector quantization method with uniform semantic mapping, which can assign meaningful and non-conflicting IDs (called item indices) for items. For alignment tuning, we propose a series of specially designed tuning tasks to enhance the integration of collaborative semantics in LLMs. Our fine-tuning tasks enforce LLMs to deeply integrate language and collaborative semantics (characterized by the learned item indices), so as to achieve an effective adaptation to recommender systems.

![model](./asset/model.jpg)

## Requirements

```
torch==1.13.1+cu117
accelerate
bitsandbytes
deepspeed
evaluate
peft
sentencepiece
tqdm
transformers

# 新环境（Python3.9）
conda create -n lcrec python=3.9 -y
conda activate lcrec

# 安装 pytorch 2.0.0 + cu118
conda install -y pytorch==2.0.0 torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia

# 其它依赖
pip install accelerate evaluate peft sentencepiece tqdm transformers

# 安装 bitsandbytes（>=0.43）
pip install bitsandbytes==0.45.0

# deepspeed
pip install deepspeed

```

## Model Checkpoint

The delta weights on the three datasets can be downloaded from huggingface hub ([Instruments](https://huggingface.co/bwzheng0324/lc-rec-instruments-delta), [Arts](https://huggingface.co/bwzheng0324/lc-rec-arts-delta), [Games](https://huggingface.co/bwzheng0324/lc-rec-games-delta)). After downloading, you can add our deltas to the original LLaMA weights to obtain LC-Rec weights:

1. Get the original [LLaMA](https://huggingface.co/huggyllama/llama-7b) weights.
2. Use the following scripts to get LC-Rec weights by applying our delta.

```shell
python convert/merge_delta.py \
    --base-model-path llama-7b \
    --target-model-path /mnt/d/LC-Rec/ckpt/Games \
    --delta-path Games-delta
    
C:\Users\RTX\AppData\Local\Temp\C0CB8EDE-BAD1-445A-8A13-C10A8D19F41C\swap.vhdx
```
模型存储到ckpt中：
```shell
(lcrec) root@DESKTOP-IG5PJ98:/mnt/d/LC-Rec# cd ckpt/
(lcrec) root@DESKTOP-IG5PJ98:/mnt/d/LC-Rec/ckpt# ls
Games
```


## Dataset

We use three datasets in our paper, all of which have been uploaded to [Google Drive](https://drive.google.com/drive/folders/1RcJ2M1l5zWPHYuGd9l5Gibcs5w5aI3y6?usp=sharing) 

数据集下载到data目录中:
```shell
(lcrec) root@DESKTOP-IG5PJ98:/mnt/d/LC-Rec# cd data
(lcrec) root@DESKTOP-IG5PJ98:/mnt/d/LC-Rec/data# ls
Arts  Games  Instruments
```
## Train

The detailed scripts for all three datasets are in `run.sh`:

```shell
DATASET=Games
BASE_MODEL=huggyllama/llama-7b
DATA_PATH=./data
OUTPUT_DIR=./ckpt/$DATASET/

torchrun --nproc_per_node=8 --master_port=23324 finetune.py \
    --base_model $BASE_MODEL \
    --output_dir $OUTPUT_DIR \
    --dataset $DATASET \
    --data_path $DATA_PATH \
    --per_device_batch_size 8 \
    --gradient_accumulation_steps 2 \
    --learning_rate 5e-5 \
    --epochs 4 \
    --weight_decay 0.01 \
    --save_and_eval_strategy epoch \
    --deepspeed ./config/ds_z3_bf16.json \
    --bf16 \
    --only_train_response \
    --tasks seqrec,item2index,index2item,fusionseqrec,itemsearch,preferenceobtain \
    --train_prompt_sample_num 1,1,1,1,1,1 \
    --train_data_sample_num 0,0,0,100000,0,0 \
    --index_file .index.json


cd convert
nohup ./convert.sh $OUTPUT_DIR >convert.log 2>&1 &
cd ..
```

## Test

Test with a single GPU:

```shell
#DATASET=Games
#DATA_PATH=Games
#CKPT_PATH="/mnt/d/LC-Rec/Games"
#RESULTS_FILE="/mnt/d/LC-Rec/Games/result.json"
DATASET=Games
DATA_PATH=./data
CKPT_PATH=./ckpt/$DATASET/
RESULTS_FILE=./results/$DATASET/result.json

python test.py \
    --gpu_id 0 \
    --ckpt_path $CKPT_PATH \
    --dataset $DATASET \
    --data_path $DATA_PATH \
    --results_file $RESULTS_FILE \
    --test_batch_size 1 \
    --num_beams 20 \
    --test_prompt_ids all \
    --index_file .index.json
```

Test with multiple GPUs:

```shell
#DATASET=Games
#DATA_PATH=./data
#CKPT_PATH=./ckpt/$DATASET/
#RESULTS_FILE=./results/$DATASET/result.json
DATASET=Games
DATA_PATH=./data
CKPT_PATH=./ckpt/$DATASET/
RESULTS_FILE=./results/$DATASET/result.json

torchrun --nproc_per_node=8 --master_port=23324 test_ddp.py \
    --ckpt_path $CKPT_PATH \
    --dataset $DATASET \
    --data_path $DATA_PATH \
    --results_file $RESULTS_FILE \
    --test_batch_size 1 \
    --num_beams 20 \
    --test_prompt_ids all \
    --index_file .index.json
```

## Acknowledgement

The implementation is based on [HuggingFace](https://github.com/huggingface/transformers).

Please cite the following paper as the reference if you use our codes or the processed datasets.
```bigquery
@inproceedings{zheng2024adapting,
  title={Adapting large language models by integrating collaborative semantics for recommendation},
  author={Zheng, Bowen and Hou, Yupeng and Lu, Hongyu and Chen, Yu and Zhao, Wayne Xin and Chen, Ming and Wen, Ji-Rong},
  booktitle={2024 IEEE 40th International Conference on Data Engineering (ICDE)},
  pages={1435--1448},
  year={2024},
  organization={IEEE}
}
```
