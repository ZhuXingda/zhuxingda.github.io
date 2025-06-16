---
title: "【LLM】VLLM 单卡部署 Qwen3-4B"
date: "2025-05-23T13:44:34+08:00"
categories:
  - "LLM"
tags:
  - "VLLM"
  - "Qwen3"
draf: true
---
尝试在个人显卡上部署一个 Qwen-4B 模型
<!--more-->
## 1. 部署
#### 1.1 软硬件版本
- 显卡：RTX4070
- 操作系统：Ubuntu 23.10
- Python：3.12
- VLLM：0.8.5.post1
- CUDA：12.2
#### 1.2 部署环境配置
```shell
# 下载模型文件
$ git clone https://www.modelscope.cn/Qwen/Qwen3-4B.git
# 创建 python 环境
$ conda create --name llm_hello_world python=3.12
$ conda activate llm_hello_world
# 安装 vllm
$ pip install -U "vllm"
# Docker 拉 Open-WebUI 镜像
$ docker pull ghcr.dockerproxy.com/open-webui/open-webui:main
```
#### 1.3 部署测试
启动 VLLM Serve
```shell
$ vllm serve /home/zhuxingda/Projets/Qwen3-4B --quantization fp8 --gpu-memory-utilization 0.6 --max-num-seq 1 --max-model-len 4K --api-key 123456
```
部署完之后请求接口确认成功
```shell
$ curl http://localhost:8000/v1/models -H "Authorization: Bearer 123456" | jq
{
  "object": "list",
  "data": [
    {
      "id": "/home/zhuxingda/Projets/Qwen3-4B",
      "object": "model",
      "created": 1748317788,
      "owned_by": "vllm",
      "root": "/home/zhuxingda/Projets/Qwen3-4B",
      "parent": null,
      "max_model_len": 4096,
      "permission": [
       ...
      ]
    }
  ]
}
```
启动 Open-WebUI 镜像
```shell
$ sudo docker run -d \                                                       
   --restart unless-stopped \
   --name open-webui \
   -p 0.0.0.0:8080:8080 \
   -v $(pwd)/data:/app/backend/data \
   -e WEBUI_SECRET_KEY=123456 \
   -e HF_ENDPOINT=https://hf-mirror.com \
   ghcr.nju.edu.cn/open-webui/open-webui:main
``` 
启动成功之后打开 http://host-name:8080 并在设置中新建连接
![截图](https://ooo.0x0.ooo/2025/05/27/OdVgYK.png)
然后在工作空间中添加模型即可使用   
查看显卡监控发现尽管是个 4B 模型并且用了 FP8 量化，但 VLLM 的进程还是占用了9个多GB显存
```shell
$ nvidia-smi 
Tue May 27 12:32:42 2025       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.171.04             Driver Version: 535.171.04   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce RTX 4070        Off | 00000000:01:00.0  On |                  N/A |
|  0%   37C    P8               7W / 200W |   9858MiB / 12282MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A    213900      C   ...envs/llm_hello_world/bin/python3.12     9852MiB |
+---------------------------------------------------------------------------------------+
```
但查看 VLLM 启动时的日志模型只占用了 4.1 GB
> INFO 05-25 23:37:44 [gpu_model_runner.py:1347] Model loading took 4.1402 GiB and 0.962489 seconds
