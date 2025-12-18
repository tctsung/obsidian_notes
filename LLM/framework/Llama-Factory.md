---
created: 2025-09-30T22:05
updated: 2025-12-18T00:15
---
### Intro
- easy to use LLM fine-tuning framework 
- link to [repo](https://github.com/hiyouga/LLaMA-Factory.git)
	- ref version: [v0.9.3](https://github.com/hiyouga/LLaMA-Factory/releases/tag/v0.9.3)
## Installation

```
# create venv
conda create --name finetune python=3.11 -y
conda activate finetune

# clone llama-factory
git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory

# install transformers:
git clone depth==1 -b main https://github.com/huggingface/transformers.git
cd transformers
pip install .
echo 'export DISABLE_VERSION_CHECK=1' >> ~/.bashrc  # skip version check

# install llama-factory
pip install -e ".[torch,metrics]" --no-build-isolation

# install vllm (compatible to torch 2.6)
pip install vllm==0.8.0 --no-build-isolation
```
## Workflow
### data prep
1. Prepare data in the proper format based on the fine-tune method
	- supports two format, see example data str: [LLaMA-Factory/data](https://github.com/hiyouga/LLaMA-Factory/tree/2b27283ba0566eda9ec7ac335642807189c87e70/data)
2. save the fine-tuned dataset into repo
	- location: `LLaMA-Factory/data/<MY_FINETUNED_DATA.json>`
3. append the new data into LLaMA-Factory/data/**dataset_info.json**
	- `"ranking": true` is DPO specific
4. validation file: put a sep file & define it in dataset_info.json as well
	- use `"split": "train"` for both train & validation data
```python
# eg. DPO dataset
"<MY_FINETUNED_DATA>": {
    "file_name": "<MY_FINETUNED_DATA>.json",
    "formatting": "sharegpt",
    "ranking": true,
    "columns": {
      "messages": "conversations",
      "chosen": "chosen",
      "rejected": "rejected"
    }
  }
  
# for validation set, still use split: train, it will be specified in yaml
"dpo_cn_len_val": {
    "file_name": "dpo_valdation.json",
    "formatting": "sharegpt",
    "ranking": true,
    "split": "train",
    "columns": {
      "messages": "conversations",
      "chosen": "chosen",
      "rejected": "rejected"
    }
  }
  
```

### model training: yaml file
1. copy & paste a proper yaml file in `LLaMA-Factory/examples/train_xxx`
	- do <span style="color:rgb(255, 0, 0)">NOT</span> leave a blank space in yaml file name (eg. xxx copy.yaml $\rightarrow$ bad)
	- choose the yaml file based on the <span style="color:rgb(255, 0, 0)">finetune methodology</span>, NOT the model
	- eg. LoRA + DPO $\rightarrow$ `LLaMA-Factory/examples/train_lora/llama3_lora_dpo.yaml`
		- choose this even if NOT using llama3
2. update the copied YAML file
	- must change:
		- `model_name_or_path`: Huggingface API
			- might use local model
		- `dataset`: change to the new dataset name from added key in `dataset_info.json`
		- `template`: update based on the model you choose; see [readme](https://github.com/hiyouga/LLaMA-Factory/tree/2b27283ba0566eda9ec7ac335642807189c87e70?tab=readme-ov-file#supported-models)
		- `output_dir`: something that make sense
	- optional:
		- cutoff_len
		- max_samples (update if finetuned data size bigger than default)
		- learning_rate
		- num_train_epochs
		- lora_rank
		- val_size
		- save_steps
		- save_strategy: "epoch" ([ref](https://huggingface.co/docs/transformers/main_classes/trainer#transformers.TrainingArguments.save_strategy)
```yaml
### model 
model_name_or_path: Qwen/Qwen2.5-7B-Instruct 
trust_remote_code: true

### method
stage: dpo
do_train: true
finetuning_type: lora
lora_rank: 8
lora_target: all
pref_beta: 0.1
pref_loss: sigmoid  # choices: [sigmoid (dpo), orpo, simpo]

### dataset
dataset: dpo_en_len
template: qwen
cutoff_len: 2048   # be aware of OOM
max_samples: 2000
overwrite_cache: true
preprocessing_num_workers: 16
dataloader_num_workers: 4

### output
output_dir: saves/qwen2.5-7B/lora/dpo
logging_steps: 10
save_steps: 500
plot_loss: true
overwrite_output_dir: true
save_only_model: false
report_to: tensorboard  # choices: [none, wandb, tensorboard, swanlab, mlflow]

### train
per_device_train_batch_size: 1
gradient_accumulation_steps: 8
learning_rate: 5.0e-6
num_train_epochs: 3.0
lr_scheduler_type: cosine
warmup_ratio: 0.1
bf16: true
ddp_timeout: 180000000
resume_from_checkpoint: null

### validation set
eval_dataset: dpo_cn_len_val
per_device_eval_batch_size: 1
eval_strategy: epoch
# eval_steps: 200
```

3. train the model
```bash
llamafactory-cli train examples/train_XXX/<my training file>.yaml
```
#### Validation set
- [ref](https://github.com/hiyouga/LLaMA-Factory/pull/4691)
- setÂ `split: â€œtrainâ€ `Â for both datasets
- removeÂ `val_size`Â argument from yaml doc
- set `eval_dataset` using the validation set
#### Monitor- tensorboard

- in training yaml file, set: `report_to: tensorboard`
	- if `overwrite_output_dir: true` : the file will be overwritten
- limitation: **validation result will be atÂ `trainer_state.json`**Â of the save directory after the run, but not visible during monitoring
```bash
# connect to dashboard:
tensorboard --logdir ./<output_dir>`
```

### Inference
1. create a new YAML file in `LLaMA-Factory/examples/inference/`
2. update the following
	- model_name_or_path, template: same as training
	- adapter_name_or_path: match training `output_dir`
		- <span style="color:rgb(255, 0, 0)">remove this line if want base model</span>
```YAML
model_name_or_path: Qwen/Qwen2.5-7B-Instruct 
adapter_name_or_path: saves/qwen2.5-7B/lora/dpo  # rm for base model (ctrl set)
template: qwen
infer_backend: huggingface  # choices: [huggingface, vllm, sglang]
trust_remote_code: true
```
3. CLI inference:
```bash
llamafactory-cli chat examples/inference/<my new file>.yaml
```

### API call/deployment
- to use the LLM for inference in other program
1. launch an API
	- default API port: 8000
	- this is an OpenAI compatible API endpoint
```
API_PORT=<port no.> llamafactory-cli api examples/inference/<my file>.yaml
```

2. get model response in python
```python
from openai import OpenAI

# client interface:
client = OpenAI(
	api_key="",
	base_url="http://localhost:8080/v1"
	)
# example message
messages=[
	{"role": "system", "content": "You are a helpful assistant."},
	{"role": "user", "content": "Tell me a short story."}
	]

# API call:	
result = client.chat.completions.create(
	messages=messages, 
	model="my_finetuned_model")


# clean model response:
llm_output_str = result.choices[0].message.content
```