# Learning Diverse Attacks on Large Language Models for Robust Red-teaming and Safety Tuning

This repository contains code for Red-teaming with GFlowNet, as described in the ICLR 2025 paper.

**Note**: We are actively working on testing and cleaning up the code for release. If you encounter issues before that please let us know!


**Learning Diverse Attacks on Large Language Models for Robust Red-teaming and Safety Tuning**<br />
Seanie Lee, Minsu Kim, Lynn Cherif, David Dobre, Juho Lee, Sung Ju Hwang, Kenji Kawaguchi, Gauthier Gidel, Yoshua Bengio, Esmeralda S. Whitammer, Moksh Jain <br/>
Paper: https://arxiv.org/abs/2405.18540
<details>
<summary>
BibTeX
</summary>
  
```bibtex
@article{
lee2025learning,
title={Learning Diverse Attacks on Large Language Models for Robust Red-Teaming and Safety Tuning},
author={Seanie Lee and Minsu Kim and Lynn Cherif and David Dobre and Juho Lee and Sung Ju Hwang and Kenji Kawaguchi and Gauthier Gidel and Yoshua Bengio and Esmeralda S. Whitammer and Moksh Jain},
journal={International Conference on Learning Representations (ICLR)},
year={2025}
}
```
</details>


## Installation of Dependencies
```bash
conda env create -n redteam python=3.10
conda activate redteam
pip install -r requirements.txt
```

## SFT
You can download the checkpoint from [link](https://drive.google.com/drive/folders/1yG9RPnnL83nrVJ7tiuPYHEHNiN_cDIAo?usp=sharing).
and save it under the directory ```./save```

<details>
<summary>
Command-line script
</summary>
```bash
python main.py \
--mode sft \
--lr 3e-5 \
--train_steps 100 \
--grad_acc_steps 32 \
--batch_size 1024 \
--prompt_file ./prompts/alpaca.jsonl \
--few_shot_file ./prompts/sft_dataset.json \
--exp_name gpt2-sft-position-final
```
</details>


## GFN fine-tuning with **GPT2** target model
You can try three different victim models: ```["vicgalle/gpt2-alpaca", "meta-llama/Llama-2-7b-chat-hf", "databricks/dolly-v2-7b"]```.

<details>
<summary>
Command-line script
</summary>
```bash
python main.py \
--exp_name gpt2-gfn \
--sim_tolerance 0.25 \
--victim_model vicgalle/gpt2-alpaca \
--lr 1e-4 \
--max_len 20 \
--reward_sched_horizon 500 \
--train_steps 5000 \
--seed 42 \
--temp_low 0.5 \
--temp_high 2.0 \
--buffer_size 1000 \
--lm_sched_end 1.0 \
--beta 0.1 \
--sim_tolerance 0.25
```
and the command for evaluation:
```
python eval.py \
--ckpt save/gpt2-gfn/latest \
--output_file gpt2-gfn \
--victim_model gpt2
```
</details>

## GFN fine-tuning with **Dolly** target model

<details>
<summary>
Command-line script
</summary>
```bash
python main.py \
--exp_name dolly-gfn \
--sim_tolerance 0.25 \
--victim_model databricks/dolly-v2-7b \
--lr 1e-4 \
--max_len 20 \
--reward_sched_horizon 500 \
--train_steps 5000 \
--seed 42 \
--temp_low 0.5 \
--temp_high 2.0 \
--lm_sched_horizon 2000 \
--lm_sched_end 1.0 \
--buffer_size 1000 \
--compare reward \
--beta 0.1
```

```bash
python eval.py \
--ckpt save/dolly-gfn/latest \
--output_file dolly-gfn \
--victim_model dolly
```
</details>

## GFN fine-tuning with **Gemma-2b-it** target model

<details>
<summary>
Command-line script
</summary>
```bash
python main.py \
--exp_name gemma-gfn \
--sim_tolerance 0.3 \
--victim_model google/gemma-2b-it \
--lr 1e-4 \
--reward_sched_horizon 1000 \
--train_steps 5000 \
--buffer_size 5000 \
--seed 42 \
--max_len 20 \
--temp_low 0.7 \
--temp_high 2.0 \
--lm_sched_end 1.2 \
--lm_sched_horizon 2000 \
--compare c_reward \
--prioritization c_reward \
--beta 0.1 \
--metric cosine
```

```bash
python eval.py \
--ckpt /save/gemma-gfn/latest \
--output_file gemma-gfn \
--victim_model gemma
```
</details>


## GFN fine-tuning with **Llama-2-7b** target model
<details>
<summary>
Command-line script
</summary>
```bash
python main.py \
--exp_name llama-gfn \
--sim_tolerance 0.3 \
--victim_model meta-llama/Llama-2-7b-chat-hf \
--lr 1e-4 \
--reward_sched_horizon 1000 \
--train_steps 5000 \
--buffer_size 5000 \
--seed 42 \
--max_len 20 \
--temp_low 0.7 \
--temp_high 2.0 \
--lm_sched_end 1.2 \
--lm_sched_horizon 2000 \
--sft_ckpt $SFT_CKPT \
--compare c_reward \
--prioritization c_reward \
--beta 0.1 \
--metric cosine
```

```bash
python eval.py \
--ckpt save/llama-gfn/latest \
--output_file llama-gfn \
--victim_model llama
```
</details>

## MLE-smoothing
For gpt2, dolly target model, we set to `train_steps` for 2000 and `batch_size` for 2048. For the other models, we use 1000 and 1024 for train_steps and batch_size, respectively. Note that $\text{target\\_model} \in \\{\text{gpt2}, \text{dolly}, \text{gemma}, \text{llama} \\}$.


Collect offline samples based on reward:
```bash
python collect_samples.py --exp_name "{target_model}"_gfn
```

Run MLE smoothing:
<details>
<summary>
Command-line script
</summary>
```bash
python main.py \
--mode distillation \
--exp_name "{target_model}"_mle \
--lr 1e-4 \
--seed 42 \
--batch_size "batch_size" \
--train_steps "train_steps" \
--grad_acc_steps 8 \
--model_name gpt2-sft-position-final \
--few_shot_file offline_dataset/"{target_model}"_gfn
```

For evaluation:
```bash
python eval.py \
--ckpt save/"{target_model}"_gfn/latest \
--output_file "{target_model}"_mle \
--victim_model "{target_model}" \
--no_lora
```
</details>

## Transfer experiments
First train an attacker model with GFlowNet and MLE for Gemma and you can transfer to attack new target models from $\\{\text{llama, mistral, gemma, starling} \\}$

<details>
<summary>
Command-line script
</summary>
```bash
python run_transfer.py \
--victim_model gemma \
--exp_name gemma_mle \
--target_model llama
```
</details>

## Safety Fine-tuning
For safety fine-tuning, train gflownet + MLE to red-team Gemma-2b-it and generate attack prompts.


<details>
<summary>
Command-line script
</summary>
```bash
python safety_datset/create_safety_response.py \
--input_file results/gemma/gemma_mle.json
--output_file safety_dataset/gemma_mle.json
```


```bash
python main.py \
--model_name google/gemma-2b-it \
--save_dir /network/scratch/s/seanie.lee/redteam/save \
--mode safety \
--lr 5e-5 \
--weight_decay 0.0 \
--batch_size 32 \
--num_warmup_steps 0 \
--epoch 2 \
--prompt_file safety_dataset/gemma_mle.json \
--exp_name gfn-safety-tuned
```
</details>
