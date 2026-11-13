---
title: 用 AMD AI MAX+ 395 部署 ComfyUI
date: 2026-11-13
tags: [ai, comfyui, amd, gfx1151]
summary: 记录使用 AMD AI MAX+ 395 部署 ComfyUI 的一些经验。
---

# AMD AI MAX+ 395 部署 ComfyUI

当前环境为 Linux kernel 6.17, ROCm 10.1, Python 3.14, ComfyUI v0.45.2

## Python

推荐使用[Astra UV](https://docs.astral.sh/uv/getting-started/installation/)，然后安装python3.14：`uv python install 3.14`  

## ROCm

可以参考[这篇帖子](./overview.md)

## ComfyUI

推荐以下安装方式： 
1. 创建python环境  
    ```
    mkdir comfyui && cd comfyui
    uv venv --python 3.14
    source .venv/bin/activate
    ```
2. 准备ComfyUI
    ```
    git clone https://github.com/Comfy-Org/ComfyUI
    cd ComfyUI
    git checkout v0.45.1 #换成你想用的版本
    cd ..
    ```
3. 装依赖
    ```
    uv pip install -U -r ComfyUI/requirements.txt -i https://pypi.org/simple
    uv pip install -U -r ComfyUI/manager_requirements.txt -i https://pypi.org/simple
    ```
    镜像源同步不及时会导致安装出错，所以这里指定官方pypi源。如果网络环境问题导致连不上就用镜像源的试试吧
4. 启动参数
    创建一个`start.sh`
    ```
    source .venv/bin/activate && \
    python ComfyUI/main.py --listen --enable-manager \
        --disable-mmap --bf16-vae \
        --disable-smart-memory \
        --disable-pinned-memory \
        --disable-dynamic-vram \
        --use-ck-attention
    ```
    目前的环境版本我测试出来只有这套是最快最稳定的，问AI给出来的要么慢要么直接卡死，气笑了
