---
created: 2025-09-30T22:05
updated: 2025-09-30T22:12
---
### Intro
- easy to use fine-tuning LLM framework
- link to [repo](https://github.com/hiyouga/LLaMA-Factory.git)
 - ref version: [v0.9.3](https://github.com/hiyouga/LLaMA-Factory/releases/tag/v0.9.3)

## Workflow
### data prep
1. Prepare data in the proper format based on the fine-tune method
 - supports two format, see example data str: [LLaMA-Factory/data](https://github.com/hiyouga/LLaMA-Factory/tree/2b27283ba0566eda9ec7ac335642807189c87e70/data)
2. save the fine-tuned dataset into repo
 - location: LLaMA-Factory/data/<MY_FINETUNED_DATA.json>
3. append the new data into LLaMA-Factory/data/**dataset_info.json**
 - "ranking": true is DPO specific
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
```
### model training: yaml file
1. copy & paste a proper yaml file in LLaMA-Factory/examples/train_xxx
 - do <span style="color:rgb(255, 0, 0)">NOT</span> leave a blank space in yaml file name (eg. xxx copy.yaml $\rightarrow$ bad)
 - choose the yaml file based on the <span style="color:rgb(255, 0, 0)">finetune methodology</span>, NOT the model
 - eg. LoRA + DPO $\rightarrow$ LLaMA-Factory/examples/train_lora/llama3_lora_dpo.yaml
  - choose this even if NOT using llama3
2. update the copied YAML file
 - must change:
	 - `model_name_or_path`: Huggingface API
	 - might use local model
	 - `dataset`: change to the new dataset name from added key in dataset_info.json
	 - `template`: update based on the model you choose; see [readme](https://github.com/hiyouga/LLaMA-Factory/tree/2b27283ba0566eda9ec7ac335642807189c87e70?tab=readme-ov-file#supported-models)
	 - `output_dir`: something that make sense
 - optional:
	 - cutoff_len (be aware of OOM issue)
	 - max_samples (update if finetuned data size is bigger than default)
	 - learning_rate
	 - num_train_epochs
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
report_to: none  # choices: [none, wandb, tensorboard, swanlab, mlflow]

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
```
3. train the model
```bash
llamafactory-cli train examples/train_XXX/<my training file>.yaml
```
### Inference
1. create a new YAML file in LLaMA-Factory/examples/inference/
2. update the following
 - model_name_or_path, template: same as training
 - adapter_name_or_path: match training output_dir
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
```bash
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