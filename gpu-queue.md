# FICS GPU 炼丹指南

> 文档维护：Zhirui Huang（黄至锐）& Haozhe Zhu（朱浩哲）

## FICS 基础设施

FICS 的 makkapakka 队列包括 4 个计算节点，每个节点配备了 5 张 NVIDIA RTX-3090 24G 计算卡。

为了提高 GPU 计算节点读取磁盘的带宽，FICS 提供了 lamport 分布式文件夹：
- `/lamport/shared`：登录节点和所有计算节点可见（炼丹原则上不需要用它）；
- `/lamport/makkapakka`：仅登录节点和 makkapakka 队列计算节点可见。

原则上，请将所有的 Model 和 Dataset 存储在 `/lamport/makkapakka`，建议将您的 `conda` 环境也存放在其中。请您在`/lamport/makkapakka`下自行新建和您的用户名完全相同的文件夹（否则会被管理员不定期删除），然后您炼丹需要读写的文件都放在该文件夹下。

请务必理解并确认：
- 您的 Model、Dataset、Log 等各类和炼丹有关的磁盘读写必须在 `/lamport/makkapakka/USERNAME`下；
- 严禁在个人 home 目录下（即 `/capsule/home/USERNAME`）进行密集的 I/O 读写；
- 违反上述规则可能会导致您的 FICS 被封禁且删除。

我们预置了一些常用的数据集：
- ImageNet（ILSVRC2012）：`/lamport/makkapakka/Dataset/imagenet`
- 如果您认为您使用的数据集比较常用，可以联系我补充在这里，方便其他人直接使用（省去下载时间）。

## 软件环境设置

一般而言，炼丹不会涉及图形界面，因此本教程不会用到 VNC。

### Conda 开发环境

#### Conda 的安装
FICS 的 makkapakka 计算节点是自带 Python 环境的，但不建议使用这个环境（版本非常古老）。建议您使用 `conda` 管理自己的 Python 虚拟环境。常见的 `conda` 发行包包括 `Anaconda`、`Miniconda`、`Miniforge` 等。

这里以 `Miniforge` 为例，介绍下如何在 FICS 上安装 `conda` 环境：
1. 从 [Miniforge 的 GitHub 仓库](https://github.com/conda-forge/miniforge/releases)下载用于 Linux x86_64 平台的安装文件 `Miniforge3-Linux-x86_64.sh` （根据版本不同，这个名字可能会有所差异）到您本地计算机上，然后通过 SCP 上传到 FICS 您的用户 home 目录下（关于 SCP 的使用，可参考 “[FICS 和本地间的文件上传/下载](README.md#fics-和本地间的文件上传下载)”）：
2. 通过 SSH 登录 FICS，然后在 Terminal 中执行以下命令：
  ```bash
  # 为该脚本添加可执行权限（如果没有的话）
  chmod u+x /path/to/your/Miniforge3-Linux-x86_64.sh
  # 直接运行该脚本
  /path/to/your/Miniforge3-Linux-x86_64.sh
  ```
3. 按照提示完成安装，安装过程中需要您回答一些问题，请根据实际需求和具体情况回答。建议您将安装目录设置为 /lamport 分布式文件夹下您的个人目录：`/lamport/makkapakka/USERNAME/miniforge3`；
4. 安装程序会在最后将一段初始化脚本增加到您的 `.bashrc`/`.zshrc` 的最后，这样以来您每次登录 FICS 时它都会设置好您的 Python 环境（命令行中会显示一个 `(base)` 字样表示您当前的 Python 环境）。
5. 重启您的 shell（或者重新登录 FICS），然后执行以下命令，确认您的 `conda` 环境已经安装成功：
  ```bash
  conda info
  ```

为了减少您下载安装文件并上传到服务器上的困扰，[Haozhe Zhu](https://github.com/zhutmost) 个人在服务器上预置一个 `Miniforge` 的安装文件：`/capsule/home/hzzhu/Downloads/Miniforge3-Linux-x86_64.sh`，即您可以跳过上面的第 1、2 步骤，直接执行以下命令进行安装：
```bash
/capsule/home/hzzhu/Downloads/Miniforge3-Linux-x86_64.sh
```

#### 更换 conda 和 pip 的下载源

默认情况下 conda/pip 会从境外的服务器上下载软件包。为了提高下载速度，我们建议您将下载源更换为国内的镜像源。您可以参考其他[网络教程](https://zhuanlan.zhihu.com/p/474087997)，也可以直接参考下面的超快速流程：

修改 `conda` 的下载源：
```bash
# 超快速流程之直接复制 Haozhe Zhu 的 conda 配置文件
# 如果不需要了，直接删除 /capsule/home/USERNAME/.condarc 即可
cp /capsule/home/hzzhu/.condarc /capsule/home/USERNAME/.condarc
```

修改 `pip` 的下载源：
```bash
# 设置为清华 TUNA 的下载源
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 换回默认源
pip config unset global.index-url
```

#### Python 虚拟环境

如果您已经成功安装 `conda` 环境，那么在默认情况下您处于 `(base)` 虚拟环境中。

我们可以创建其他虚拟环境，并在其中安装软件包。例如，我们需要创建一个名为 `myenv` 的虚拟环境，并在其中安装 `pytorch`：
```bash
# 创建名为 myenv 的虚拟环境
conda create -n myenv
# 激活该虚拟环境，您会在命令行中看到您的虚拟环境从 (base) 换成了 (myenv)
conda activate myenv
# 安装 python、pytorch
conda install python
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
# 退出该虚拟环境，您会在命令行中看到您的虚拟环境从 (myenv) 换回了 (base)
```

您在 `(myenv)` 虚拟环境中安装的软件包不会影响到 `(base)` 和其他虚拟环境中的软件包。您可以使用 `conda activate/deactivate` 自如地在多个虚拟环境之间切换。

如果您需要删除某个虚拟环境，可以使用以下命令：
```bash
# 删除名为 myenv 的虚拟环境
conda remove -n myenv --all
```

## 提交炼丹任务到 makkapakka 队列

当您使用 SSH 连接 FICS 时，您处于 mgmt 登录节点。您的 Python 炼丹程序需要通过 SLURM 调度器提交到 makkapakka 队列中执行。您可以查阅 [FICS 服务器集群使用指南](README.md)了解如何提交任务。

由于炼丹任务一般需要精细地调整 CPU 核数、进程数等，我们建议您自己编写 srun、sbatch 命令和脚本，不要使用 ilaunch 或 zlaunch 以免遇到奇怪的性能瓶颈。

如果您确实需要使用 ilaunch 或 zlaunch，您可以参考以下命令提交任务：
```bash
ilaunch q makkapakka gpu 0,1,2,3 python main.py
zlaunch -q m --gpu=5 -- python main.py
```

**警告**：未经管理员邮件同意，严禁在 mgmt 登录节点上直接运行 Python 程序，以及绕过 SLURM 调度器直接在 makkapakka 计算节点上运行 Python 程序。违反此规则可能会导致您的 FICS 被封禁且删除。

## 高级使用

### 多机多卡并行

TODO：写不动了，先就这样吧

### Jupyter on FICS

如果您需要使用 makkapakka 的 GPU 运行 Jupyter Notebook，可参考[网络教程]([：](https://alexanderlabwhoi.github.io/post/2019-03-08_jpn_slurm/))

注意：
- 不要直接使用 8888 端口
- 参考文档中的 node1 对应为 makkapakka0x（x表示队列下的第几台机器）

### Jetbrains PyCharm 远程开发

TODO：写不动了，先就这样吧
