---
title: Run a Large Language model (LLM) chatbot with rtp-llm on Arm servers
weight: 3

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Before you begin
The instructions in this Learning Path are for any Arm Neoverse N2 or Neoverse V2 based server running Ubuntu 22.04 LTS. You need an Arm server instance with at least four cores and 16GB of RAM to run this example. Configure disk storage up to at least 32 GB. The instructions have been tested on an Alibaba Cloud g8y.8xlarge instance and an AWS Graviton4 r8g.8xlarge instance.

## Overview

Arm CPUs are widely used in traditional ML and AI use cases. In this Learning Path, you will learn how to run generative AI inference-based use case like a LLM chatbot on Arm-based CPUs. You do this by deploying the [Qwen2-0.5B-Instruct model](https://huggingface.co/Qwen/Qwen2-0.5B-Instruct) on your Arm-based CPU using `rtp-llm`.

[rtp-llm](https://github.com/alibaba/rtp-llm) is an open source C/C++ project developed by Alibaba that enables efficient LLM inference on a variety of hardware. 
 
## Install dependencies 

Install `micromamba` to setup python 3.10 at path `/opt/conda310`, required by `rtp-llm` build system:

```bash
"${SHELL}" <(curl -L micro.mamba.pm/install.sh)
source ~/.bashrc
sudo ${HOME}/.local/bin/micromamba -r /opt/conda310 install python=3.10
micromamba -r /opt/conda310 shell
```

Install `bazelisk` to build `rtp-llm`:

```bash
wget https://github.com/bazelbuild/bazelisk/releases/download/v1.22.1/bazelisk-linux-arm64
chmod +x bazelisk-linux-arm64
sudo mv bazelisk-linux-arm64 /usr/bin/bazelisk
```

Install `git/gcc/g++` on your machine:

```bash
sudo apt install git -y
sudo apt install build-essential -y
```

Install `openblas` developmwnt package and fix the header paths:

```bash
sudo apt install libopenblas-dev
sudo mkdir -p /usr/include/openblas
sudo ln -sf /usr/include/aarch64-linux-gnu/cblas.h /usr/include/openblas/cblas.h
```

## Download and build rtp-llm

You are now ready to start building `rtp-llm`. 

Clone the source repository for rtp-llm:

```bash
git clone https://github.com/alibaba/rtp-llm
cd rtp-llm
git checkout 4656265
```

Comment out the lines 7-10 in `deps/requirements_lock_torch_arm.txt` as some hosts are not accessible from the Internet.

```bash
sed -i '7,10 s/^/#/' deps/requirements_lock_torch_arm.txt
```

By default, `rtp-llm` builds for GPU only on Linux. You need to provide extra config `--config=arm` to build it for the Arm CPU that you will run it on:

Configure and build:

```bash
bazelisk build --config=arm //maga_transformer:maga_transformer_aarch64
```
The output from your build should look like:

```output
INFO: 10094 processes: 8717 internal, 1377 local.
INFO: Build completed successfully, 10094 total actions
```

Install the built wheel package:

```bash
pip install bazel-bin/maga_transformer/maga_transformer-0.2.0-cp310-cp310-linux_aarch64.whl
```

Create a file named `python-test.py` in your `/tmp` directory with the contents below: 

```python
from maga_transformer.pipeline import Pipeline
from maga_transformer.model_factory import ModelFactory
from maga_transformer.openai.openai_endpoint import OpenaiEndopoint
from maga_transformer.openai.api_datatype import ChatCompletionRequest, ChatMessage, RoleEnum
from maga_transformer.distribute.worker_info import update_master_info

import asyncio
import json
import os

async def main():
    update_master_info('127.0.0.1', 42345)
    os.environ["MODEL_TYPE"] = os.environ.get("MODEL_TYPE", "qwen2")
    os.environ["CHECKPOINT_PATH"] = os.environ.get("CHECKPOINT_PATH", "Qwen/Qwen2-0.5B-Instruct")
    os.environ["RESERVER_RUNTIME_MEM_MB"] = "0"
    os.environ["DEVICE_RESERVE_MEMORY_BYTES"] = f"{128 * 1024 ** 2}"
    model_config = ModelFactory.create_normal_model_config()
    model = ModelFactory.from_huggingface(model_config.ckpt_path, model_config=model_config)
    pipeline = Pipeline(model, model.tokenizer)

    # usual request
    for res in pipeline("<|im_start|>user\nhello, what's your name<|im_end|>\n<|im_start|>assistant\n", max_new_tokens = 100):
        print(res.generate_texts)

    # openai request
    openai_endpoint = OpenaiEndopoint(model)
    messages = [
        ChatMessage(**{
            "role": RoleEnum.user,
            "content": "Who are you？",
        }),
    ]
    request = ChatCompletionRequest(messages=messages, stream=False)
    response = openai_endpoint.chat_completion(request_id=0, chat_request=request, raw_request=None)
    async for res in response:
        pass
    print((await response.gen_complete_response_once()).model_dump_json(indent=4))

    pipeline.stop()

if __name__ == '__main__':
    asyncio.run(main())
```

Now run this file:

```bash
python /tmp/python-test.py
```

If `rtp-llm` has built correctly on your machine, you will see the LLM model response for the prompt input. A snippet of the output is shown below:

```output
['I am a large language model created by Alibaba Cloud. My name is Qwen.']
{
    "id": "chat-",
    "object": "chat.completion",
    "created": 1730272196,
    "model": "AsyncModel",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "I am a large language model created by Alibaba Cloud. I am called Qwen.",
                "function_call": null,
                "tool_calls": null
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 23,
        "total_tokens": 40,
        "completion_tokens": 17,
        "completion_tokens_details": null,
        "prompt_tokens_details": null
    },
    "debug_info": null,
    "aux_info": null
}
```


You have successfully run a LLM chatbot with Arm optimizations, all running on your Arm AArch64 CPU on your server. You can continue experimenting and trying out the model with different prompts.

