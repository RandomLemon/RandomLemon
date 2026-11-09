---
title: 用 AMD AI MAX+ 395 部署本地模型
date: 2026-11-07
tags: [ai, llm, amd, gfx1151]
summary: 记录使用 用 AMD AI MAX+ 395 部署本地模型的一些经验。
---

# 使用 AMD AI MAX+ 395 部署本地模型

笔者接触了几台搭载Strix Halo的机器，这里来谈谈机器本身的硬件和软件。

## 机器

- 壹号本ONEXStation  
六联公模，做工不好，开机键会卡住，BIOS功能很少，没法调显存大小

- TOPC  
硬件拉满，10GbE+2.5GbE，甚至居然是8533MT/s的内存（因为和495共板）。四个m.2，并且有单独风扇散热，综合来说是硬件最好的。  
做工一坨，铝合金外壳对不齐，磁吸固定但是磁铁掉下来了，BIOS拉完了，ACPI有什么神秘bug，开机的时候插网线/HDMI会概率重启，触发的是ACPI power key pressed。型号都没有，官网没驱动下，气笑了

- GMK EVO-X2  
六联公模，两台，来自AMD University Program。有55w 85w 120w的TDP，风扇有点吵。老版本BIOS不支持核显显存调整，可以在windows里面调好再安装linux。

- FEVM FAEX1
均热板，10GbE+2.5GbE，支持100w 120w 160w的TDP，风扇比较安静。有三个M.2，其中一个接到了Oculink，不用Oculink可以换成一个2242（好像是）的NVME盘。BIOS十分全面，可以调显存大小、风扇曲线，甚至是PBO，可以降压超频。推荐把CPU设置为-100MHz，Curve Optimiser全核-20。这个SOC在满载的时候不热，但在单核跑满的时候温度很高，这样设置可以在所有工况下非常安静。全CNC，1L体积，做工很好，SSD贴在底部铝合金上，散热很好。

综合来说比较推荐FAEX1，这机器在很长时间里都是最便宜的395，结果居然还是性能最好、bios最好、体积最小、最稳定的，没绷住。。。

## 系统

两台EVO-X2安装了Debian13，linux7.2。FAEX1安装了PVE9.2，linux6.17。其中PVE+linux7.0经常有奇怪的bug，比如说突然的freeze、reboot、显存分配失败之类。

### Kernel Parameter

推荐在BIOS为核显分配专用显存为最小值，比如512MB。然后设置以下内核参数：

```
amd_iommu=pt amdgpu.gttsize=122880 ttm.pages_limit=31457280
```

这样可以利用120GB显存。其中linux内核需要保留一些内存使用，所以开机之后看到约124GB可用内存是正常的。留一点内存给基础的系统服务比较稳定。

- PVE
使用的是`systemd-boot`，所以需要在`/etc/kernel/cmdline`后面添加参数，然后运行`proxmox-boot-tool refresh`。

- Debian
使用的是`grub`，所以修改`/etc/default/grub`，然后运行`update-grub`。

### ROCm

10.1以前的ROCm有比较严重的bug，主要表现为`AsyncEventsLoop`线程会占据一到两个CPU核心，导致395变得奇热无比（98度撞墙）。更新到10.1终于好了。

[ROCm文档](https://rocm.docs.amd.com/en/latest/install/rocm.html)，这里列出Debian的安装方法：
1. 安装依赖包  
`sudo apt install libatomic1 libquadmath0`
2. 添加用户组
`sudo usermod -a -G render,video $LOGNAME`
3. 添加软件源
```
# Download and install GPG key
sudo mkdir --parents --mode=0755 /etc/apt/keyrings
wget https://stable.repo.amd.com/rocm/gpg/packages.gpg -O - | \
    gpg --dearmor | sudo tee /etc/apt/keyrings/amdrocm.gpg > /dev/null

sudo tee /etc/apt/sources.list.d/amdrocm-stable.sources << EOF
X-Repo-Id: amdrocm-stable
Types: deb
URIs: https://stable.repo.amd.com/rocm/core/packages/debian13/
Suites: stable
Components: main
Architectures: amd64
Signed-By: /etc/apt/keyrings/amdrocm.gpg
Enabled: yes
EOF

sudo apt update
```
4. 安装ROCm核心包
```
sudo apt install amdrocm10.1-gfx1151
```
5. 编译llama.cpp项目需要下面的包
```
sudo apt install amdrocm-core-dev10.1-gfx1151
```

其实那里面的表格让你选系统和设备组合，你会发现没有gfx1151+debian13的组合，但是选`ALL+Compute+Debian13+apt`之后，安装的时候用tab补全一下就会发现其实有gfx1151的特定包的。这样可以省很多空间。

### PVE 允许容器访问显卡

TODO