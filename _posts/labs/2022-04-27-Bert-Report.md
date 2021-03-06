---
title: BERT 实验记录
layout: post
author: jiamin
categories: [Laboratory, Deep Learning]
tags: [Deep Learning, BERT]
math: true
abstract: 将 BERT 模型对 CoLA 任务进行微调训练，获取在不同 Batch size 下，微调任务每 Epoch 所用时间与 GPU 占用情况。
image:
  path: /lab/bert/google-bert.jpg
  width: 800
  height: 500

---

## 实验环境

* GPU：GeForce RTX 2080 Ti $\times 1$
* Docker image：tensorflow/tensorflow:1.11.0-gpu
  * CUDA version：9.0
  * Linux 发行版本：Ubuntu 16.04.5 LTS
  * Python version：2.7.12

## 实验代码

实验所用代码仓库：<a href="https://github.com/google-research/bert" target="_blank">https://github.com/google-research/bert</a>，其所提供的代码只支持 CoLA、MRPC、MNLI、XNLI 分类任务与 SQuAD、SQuAD 2.0 问答任务。

本次实验只对 CoLA 任务进行了测试。所用实验代码为：<a href="https://github.com/google-research/bert/blob/master/run_classifier.py"  target="_blank">run_classifier.py</a>。

### 获取时间性能数据

该代码中是用 `tf.contrib.tpu.TPUEstimator` 对象的 `train` 方法来进行训练，由于我对 `TensorFlow` 的使用并不熟悉，除了将整个代码改写，我并不知道还有什么方法能够获取到每个 epoch 开始/结束的时间点。

但幸运的是，在其输出的一堆 Log 中，我发现了这样的信息：

```sh
....
INFO:tensorflow:global_step/sec: 55.2661
INFO:tensorflow:examples/sec: 1768.51
INFO:tensorflow:global_step/sec: 94.3678
INFO:tensorflow:examples/sec: 3019.77
INFO:tensorflow:global_step/sec: 90.2155
....
```
{: file="code output" }

程序给出了每秒钟处理的样例数，这意味着，我可以获取训练过程中输出的 `examples/sec` 然后取平均，从而估计出平均样例处理速度，然后用训练样例的总数 `num_examples` 去除以平均样例处理速度，从而估计出微调训练时每 epoch 所需的时间，即：

```python
time_per_epoch = num_examples / average(examples_per_sec_list)
```

### 获取GPU占用情况

由于我并不知道如何结合 TensorFlow 在代码 <a href="https://github.com/google-research/bert/blob/master/run_classifier.py"  target="_blank">run_classifier.py</a> 中监控 GPU 占用情况，所以我使用了 `watch -n 0.2 "nvidia-smi | grep % >> $NVIDIA_OUT_FILE` 命令来每 0.2s 运行一次 `nvidia-smi` 命令，并用 `grep %` 过滤输出只留下 GPU 占用情况相关的行，并输出到 `$NVIDIA_OUT_FILE` 文件。

> 实际执行过程中，`nvidia-smi | grep % >> $NVIDIA_OUT_FILE` 命令并非严格 0.2s 执行一次，可用如下命令验证：
>
> ```shell
> $ watch -n 0.2 "date +%S.%N >> tmp.txt; sleep 0.1"
> $ cat tmp.txt
> 55.138318563
> 55.447502740
> 55.757528402
> 56.068227311
> ```
> 可以看出命令 `date +%S.%N >> tmp.txt; sleep 0.1` 并非每 0.2s 启动一次，而是等该命令执行完以后，停顿 0.2s 后再次启动，因此该命令会大约 0.31s 执行一次

为了矫正时间误差，我改进为用如下命令来监控 GPU 使用情况：

```shell
$ watch -n 0.2 "nvidia-smi | grep % >> $NVIDIA_OUT_FILE; date +%s.%N  >> $NVIDIA_OUT_FILE"
```

可以获得如下所示的输出用来分析实时的 GPU 占用情况：

```
....
| 69%   72C    P2   206W / 250W |  10538MiB / 11019MiB |     86%      Default |
1650966846.042205165
| 69%   72C    P2   193W / 250W |  10538MiB / 11019MiB |     86%      Default |
1650966846.324194099
| 69%   72C    P2   190W / 250W |  10538MiB / 11019MiB |     86%      Default |
1650966846.610179581
....
```
{: file="$NVIDIA_OUT_FILE" }

执行训练任务所用的命令如下所示：

```shell
docker run --rm \
    --name "$DOCKER_NAME" \
    --gpus "\"device=$GPU_ID\"" \
    # ... docer run 的其他参数省略
    tensorflow/tensorflow:1.11.0-gpu \
    bash -c " \
    python2 run_classifier.py \
    	-- ... 参数省略 \
    ; sleep 0.5"
```

由于该 `docker run` 命令添加了 `--rm` 选项，并且在执行训练任务后 `sleep 0.5` ，因此，当训练任务结束后 0.5s，该 container 就会被删掉。

利用这一特性，可以使用 `docker exec` 来监控 GPU，训练任务结束 0.5s 后，监控任务会被杀掉从而结束监控。命令如下所示：

```shell
# $TMP_FILE 是用来防止 watch 命令的显示界面占用 shell 窗口
docker exec -t $DOCKER_NAME bash -c "watch -n 0.2 \"nvidia-smi | grep % >>$NVIDIA_OUT_FILE\"" > $TMP_FILE
```

为了能够在任务开始时及时开始监控 GPU，我的 shell script 中有如下命令。

```shell
trainMain &
while :; do
    sleep 0.2
    # 失败后下次循环再次尝试，成功后break循环
    res=$(./get_nvidia.sh -d "$DOCKER_NAME" -f "$NVIDIA_FILE" 2>&1 | grep "Error")
    if [[ -z "$res" ]]; then
    break
    fi
done
```

其中 `trainMain` 函数主要功能是运行 `docker run` 以开启训练任务。`get_nvidia.sh` 中即为上述 `docker exec` 命令，由于 container 的建立需要时间，若在建立完成前运行 `docker exec`，将会报错：`Error: No such container: ...`（ `...` 处为 `$DOCKER_NAME` 的值），因此需要每 0.2s 尝试一次，直到 `docker exec` 成功执行。

## 测试

对于所有测试，如下的超参数固定：

* `max_seq_length = 128`
* `learning_rate = 2e-5`
* `num_train_epochs = 5`

对于可用的 24 个 BERT 模型，本次实验仅测试了 BERT-Tiny (2/128)、BERT-Mini (4/256)、BERT-Small (4/512)、BERT-Medium (8/512)、BERT-Base (12/768)。

**任务描述**：CoLA 是 GLUE 中的一个单句子分类任务，语料来自语言理论的书籍和期刊，每个句子被标注为是否合乎语法的单词序列。本任务是一个二分类任务，标签共两个，分别是 0 和 1，其中 0 表示不合乎语法，1 表示合乎语法。

**样本个数**：训练集 8,551 个，开发集 1,043 个，测试集 1,063 个。

在相同的学习率与相同 max sequence length 下微调训练后，可以得出结论，对于不同的 BERT 模型，模型规模越大，其模型的估计准确度越高，但随之，每个 Epoch 所用时间越长，功耗也越大，GPU 利用率也越高。

对于相同的 BERT 模型，增加 batch size，并不能给预测准确率带来提升，但其每个 epoch 所花费的时间明显减少，虽然 GPU 的功率也随之增加，但从整体来看，每个 epoch 所花费的能耗是降低的。

## 附录：实验数据

<div class="table-wrapper">
<table><thead><tr><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Model</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Batch Size</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Evaluated Accuracy</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Evaluated Loss</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Train Loss</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Time of Per Epoch</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Average Power <p style="font-size: 50%; color:gray; margin: 0">(Rated Power: 250W)</p></th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Average Memory <p style="font-size: 50%; color:gray; margin: 0">(All Memory: 11019MiB)</p></th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Average Utilization</th><th style="text-align: center;padding: 0.15rem 0.1rem; font-size: 90%; white-space: normal; line-height: 1.2">Real time GPU performance</th></tr></thead><tbody><tr style="background-color: rgb(252, 243, 244); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Tiny</td><td style="padding: 0.175rem 0.15rem;">2</td><td style="padding: 0.175rem 0.15rem;">69.13%</td><td style="padding: 0.175rem 0.15rem;">0.90</td><td style="padding: 0.175rem 0.15rem;">0.90</td><td style="padding: 0.175rem 0.15rem;">53.11s</td><td style="padding: 0.175rem 0.15rem;">89.68W</td><td style="padding: 0.175rem 0.15rem;">10486.2MiB</td><td style="padding: 0.175rem 0.15rem;">33.16%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Tiny_2.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Tiny_2.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(252, 243, 244); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Tiny</td><td style="padding: 0.175rem 0.15rem;">4</td><td style="padding: 0.175rem 0.15rem;">69.13%</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">23.33s</td><td style="padding: 0.175rem 0.15rem;">103.02W</td><td style="padding: 0.175rem 0.15rem;">10431.9MiB</td><td style="padding: 0.175rem 0.15rem;">35.31%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Tiny_4.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Tiny_4.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(252, 243, 244); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Tiny</td><td style="padding: 0.175rem 0.15rem;">8</td><td style="padding: 0.175rem 0.15rem;">69.13%</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">15.17s</td><td style="padding: 0.175rem 0.15rem;">95.64W</td><td style="padding: 0.175rem 0.15rem;">10413.6MiB</td><td style="padding: 0.175rem 0.15rem;">29.60%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Tiny_8.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Tiny_8.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(252, 243, 244); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Tiny</td><td style="padding: 0.175rem 0.15rem;">16</td><td style="padding: 0.175rem 0.15rem;">69.13%</td><td style="padding: 0.175rem 0.15rem;">0.61</td><td style="padding: 0.175rem 0.15rem;">0.61</td><td style="padding: 0.175rem 0.15rem;">5.73s</td><td style="padding: 0.175rem 0.15rem;">115.74W</td><td style="padding: 0.175rem 0.15rem;">10291.1MiB</td><td style="padding: 0.175rem 0.15rem;">35.01%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Tiny_16.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Tiny_16.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(252, 243, 244); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Tiny</td><td style="padding: 0.175rem 0.15rem;">32</td><td style="padding: 0.175rem 0.15rem;">69.13%</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">3.21s</td><td style="padding: 0.175rem 0.15rem;">117.05W</td><td style="padding: 0.175rem 0.15rem;">10118.2MiB</td><td style="padding: 0.175rem 0.15rem;">36.09%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Tiny_32.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Tiny_32.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(252, 243, 244); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Tiny</td><td style="padding: 0.175rem 0.15rem;">64</td><td style="padding: 0.175rem 0.15rem;">69.13%</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">2.53s</td><td style="padding: 0.175rem 0.15rem;">126.77W</td><td style="padding: 0.175rem 0.15rem;">10048.6MiB</td><td style="padding: 0.175rem 0.15rem;">38.79%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Tiny_64.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Tiny_64.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(255, 255, 235); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Mini</td><td style="padding: 0.175rem 0.15rem;">2</td><td style="padding: 0.175rem 0.15rem;">70.57%</td><td style="padding: 0.175rem 0.15rem;">1.14</td><td style="padding: 0.175rem 0.15rem;">1.14</td><td style="padding: 0.175rem 0.15rem;">102.27s</td><td style="padding: 0.175rem 0.15rem;">111.77W</td><td style="padding: 0.175rem 0.15rem;">10509.9MiB</td><td style="padding: 0.175rem 0.15rem;">36.02%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Mini_2.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Mini_2.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(255, 255, 235); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Mini</td><td style="padding: 0.175rem 0.15rem;">4</td><td style="padding: 0.175rem 0.15rem;">70.37%</td><td style="padding: 0.175rem 0.15rem;">0.84</td><td style="padding: 0.175rem 0.15rem;">0.84</td><td style="padding: 0.175rem 0.15rem;">51.18s</td><td style="padding: 0.175rem 0.15rem;">118.30W</td><td style="padding: 0.175rem 0.15rem;">10494.2MiB</td><td style="padding: 0.175rem 0.15rem;">38.39%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Mini_4.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Mini_4.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(255, 255, 235); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Mini</td><td style="padding: 0.175rem 0.15rem;">8</td><td style="padding: 0.175rem 0.15rem;">70.18%</td><td style="padding: 0.175rem 0.15rem;">0.63</td><td style="padding: 0.175rem 0.15rem;">0.63</td><td style="padding: 0.175rem 0.15rem;">26.27s</td><td style="padding: 0.175rem 0.15rem;">138.96W</td><td style="padding: 0.175rem 0.15rem;">10459.0MiB</td><td style="padding: 0.175rem 0.15rem;">42.70%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Mini_8.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Mini_8.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(255, 255, 235); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Mini</td><td style="padding: 0.175rem 0.15rem;">16</td><td style="padding: 0.175rem 0.15rem;">69.22%</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">14.14s</td><td style="padding: 0.175rem 0.15rem;">158.72W</td><td style="padding: 0.175rem 0.15rem;">10389.6MiB</td><td style="padding: 0.175rem 0.15rem;">50.77%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Mini_16.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Mini_16.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(255, 255, 235); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Mini</td><td style="padding: 0.175rem 0.15rem;">32</td><td style="padding: 0.175rem 0.15rem;">70.76%</td><td style="padding: 0.175rem 0.15rem;">0.61</td><td style="padding: 0.175rem 0.15rem;">0.61</td><td style="padding: 0.175rem 0.15rem;">10.54s</td><td style="padding: 0.175rem 0.15rem;">165.95W</td><td style="padding: 0.175rem 0.15rem;">10383.2MiB</td><td style="padding: 0.175rem 0.15rem;">54.42%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Mini_32.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Mini_32.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(255, 255, 235); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Mini</td><td style="padding: 0.175rem 0.15rem;">64</td><td style="padding: 0.175rem 0.15rem;">69.51%</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">0.62</td><td style="padding: 0.175rem 0.15rem;">8.92s</td><td style="padding: 0.175rem 0.15rem;">174.75W</td><td style="padding: 0.175rem 0.15rem;">10361.4MiB</td><td style="padding: 0.175rem 0.15rem;">58.74%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Mini_64.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Mini_64.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(247, 255, 247); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Small</td><td style="padding: 0.175rem 0.15rem;">2</td><td style="padding: 0.175rem 0.15rem;">74.02%</td><td style="padding: 0.175rem 0.15rem;">1.51</td><td style="padding: 0.175rem 0.15rem;">1.52</td><td style="padding: 0.175rem 0.15rem;">112.29s</td><td style="padding: 0.175rem 0.15rem;">147.67W</td><td style="padding: 0.175rem 0.15rem;">10515.0MiB</td><td style="padding: 0.175rem 0.15rem;">59.22%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Small_2.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Small_2.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(247, 255, 247); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Small</td><td style="padding: 0.175rem 0.15rem;">4</td><td style="padding: 0.175rem 0.15rem;">74.40%</td><td style="padding: 0.175rem 0.15rem;">1.34</td><td style="padding: 0.175rem 0.15rem;">1.34</td><td style="padding: 0.175rem 0.15rem;">56.64s</td><td style="padding: 0.175rem 0.15rem;">182.98W</td><td style="padding: 0.175rem 0.15rem;">10480.8MiB</td><td style="padding: 0.175rem 0.15rem;">66.60%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Small_4.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Small_4.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(247, 255, 247); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Small</td><td style="padding: 0.175rem 0.15rem;">8</td><td style="padding: 0.175rem 0.15rem;">73.44%</td><td style="padding: 0.175rem 0.15rem;">0.99</td><td style="padding: 0.175rem 0.15rem;">1.00</td><td style="padding: 0.175rem 0.15rem;">34.76s</td><td style="padding: 0.175rem 0.15rem;">203.46W</td><td style="padding: 0.175rem 0.15rem;">10475.2MiB</td><td style="padding: 0.175rem 0.15rem;">73.51%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Small_8.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Small_8.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(247, 255, 247); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Small</td><td style="padding: 0.175rem 0.15rem;">16</td><td style="padding: 0.175rem 0.15rem;">74.21%</td><td style="padding: 0.175rem 0.15rem;">0.73</td><td style="padding: 0.175rem 0.15rem;">0.74</td><td style="padding: 0.175rem 0.15rem;">25.74s</td><td style="padding: 0.175rem 0.15rem;">204.59W</td><td style="padding: 0.175rem 0.15rem;">10462.6MiB</td><td style="padding: 0.175rem 0.15rem;">74.33%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Small_16.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Small_16.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(247, 255, 247); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Small</td><td style="padding: 0.175rem 0.15rem;">32</td><td style="padding: 0.175rem 0.15rem;">73.54%</td><td style="padding: 0.175rem 0.15rem;">0.72</td><td style="padding: 0.175rem 0.15rem;">0.73</td><td style="padding: 0.175rem 0.15rem;">20.50s</td><td style="padding: 0.175rem 0.15rem;">203.58W</td><td style="padding: 0.175rem 0.15rem;">10426.4MiB</td><td style="padding: 0.175rem 0.15rem;">75.43%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Small_32.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Small_32.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(247, 255, 247); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Small</td><td style="padding: 0.175rem 0.15rem;">64</td><td style="padding: 0.175rem 0.15rem;">73.73%</td><td style="padding: 0.175rem 0.15rem;">0.70</td><td style="padding: 0.175rem 0.15rem;">0.70</td><td style="padding: 0.175rem 0.15rem;">18.03s</td><td style="padding: 0.175rem 0.15rem;">204.82W</td><td style="padding: 0.175rem 0.15rem;">10436.3MiB</td><td style="padding: 0.175rem 0.15rem;">77.36%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Small_64.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Small_64.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(245, 251, 255); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Medium</td><td style="padding: 0.175rem 0.15rem;">2</td><td style="padding: 0.175rem 0.15rem;">78.72%</td><td style="padding: 0.175rem 0.15rem;">1.52</td><td style="padding: 0.175rem 0.15rem;">1.52</td><td style="padding: 0.175rem 0.15rem;">176.69s</td><td style="padding: 0.175rem 0.15rem;">153.54W</td><td style="padding: 0.175rem 0.15rem;">10523.1MiB</td><td style="padding: 0.175rem 0.15rem;">61.83%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Medium_2.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Medium_2.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(245, 251, 255); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Medium</td><td style="padding: 0.175rem 0.15rem;">4</td><td style="padding: 0.175rem 0.15rem;">78.62%</td><td style="padding: 0.175rem 0.15rem;">1.30</td><td style="padding: 0.175rem 0.15rem;">1.31</td><td style="padding: 0.175rem 0.15rem;">94.66s</td><td style="padding: 0.175rem 0.15rem;">194.78W</td><td style="padding: 0.175rem 0.15rem;">10512.7MiB</td><td style="padding: 0.175rem 0.15rem;">69.43%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Medium_4.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Medium_4.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(245, 251, 255); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Medium</td><td style="padding: 0.175rem 0.15rem;">8</td><td style="padding: 0.175rem 0.15rem;">78.52%</td><td style="padding: 0.175rem 0.15rem;">1.14</td><td style="padding: 0.175rem 0.15rem;">1.15</td><td style="padding: 0.175rem 0.15rem;">60.66s</td><td style="padding: 0.175rem 0.15rem;">204.15W</td><td style="padding: 0.175rem 0.15rem;">10492.2MiB</td><td style="padding: 0.175rem 0.15rem;">76.68%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Medium_8.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Medium_8.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(245, 251, 255); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Medium</td><td style="padding: 0.175rem 0.15rem;">16</td><td style="padding: 0.175rem 0.15rem;">77.85%</td><td style="padding: 0.175rem 0.15rem;">0.88</td><td style="padding: 0.175rem 0.15rem;">0.89</td><td style="padding: 0.175rem 0.15rem;">47.73s</td><td style="padding: 0.175rem 0.15rem;">205.18W</td><td style="padding: 0.175rem 0.15rem;">10482.9MiB</td><td style="padding: 0.175rem 0.15rem;">76.96%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Medium_16.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Medium_16.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(245, 251, 255); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Medium</td><td style="padding: 0.175rem 0.15rem;">32</td><td style="padding: 0.175rem 0.15rem;">78.62%</td><td style="padding: 0.175rem 0.15rem;">0.71</td><td style="padding: 0.175rem 0.15rem;">0.71</td><td style="padding: 0.175rem 0.15rem;">38.97s</td><td style="padding: 0.175rem 0.15rem;">206.95W</td><td style="padding: 0.175rem 0.15rem;">10486.3MiB</td><td style="padding: 0.175rem 0.15rem;">78.64%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Medium_32.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Medium_32.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(245, 251, 255); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Medium</td><td style="padding: 0.175rem 0.15rem;">64</td><td style="padding: 0.175rem 0.15rem;">79.10%</td><td style="padding: 0.175rem 0.15rem;">0.71</td><td style="padding: 0.175rem 0.15rem;">0.72</td><td style="padding: 0.175rem 0.15rem;">34.70s</td><td style="padding: 0.175rem 0.15rem;">212.36W</td><td style="padding: 0.175rem 0.15rem;">10479.2MiB</td><td style="padding: 0.175rem 0.15rem;">81.29%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Medium_64.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Medium_64.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(253, 248, 241); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Base</td><td style="padding: 0.175rem 0.15rem;">2</td><td style="padding: 0.175rem 0.15rem;">83.13%</td><td style="padding: 0.175rem 0.15rem;">1.52</td><td style="padding: 0.175rem 0.15rem;">1.53</td><td style="padding: 0.175rem 0.15rem;">324.06s</td><td style="padding: 0.175rem 0.15rem;">190.54W</td><td style="padding: 0.175rem 0.15rem;">10528.7MiB</td><td style="padding: 0.175rem 0.15rem;">83.31%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Base_2.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Base_2.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(253, 248, 241); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Base</td><td style="padding: 0.175rem 0.15rem;">4</td><td style="padding: 0.175rem 0.15rem;">82.84%</td><td style="padding: 0.175rem 0.15rem;">1.26</td><td style="padding: 0.175rem 0.15rem;">1.27</td><td style="padding: 0.175rem 0.15rem;">196.86s</td><td style="padding: 0.175rem 0.15rem;">212.73W</td><td style="padding: 0.175rem 0.15rem;">10525.8MiB</td><td style="padding: 0.175rem 0.15rem;">86.83%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Base_4.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Base_4.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(253, 248, 241); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Base</td><td style="padding: 0.175rem 0.15rem;">8</td><td style="padding: 0.175rem 0.15rem;">81.88%</td><td style="padding: 0.175rem 0.15rem;">1.15</td><td style="padding: 0.175rem 0.15rem;">1.16</td><td style="padding: 0.175rem 0.15rem;">140.55s</td><td style="padding: 0.175rem 0.15rem;">213.59W</td><td style="padding: 0.175rem 0.15rem;">10514.1MiB</td><td style="padding: 0.175rem 0.15rem;">86.66%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Base_8.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Base_8.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(253, 248, 241); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Base</td><td style="padding: 0.175rem 0.15rem;">16</td><td style="padding: 0.175rem 0.15rem;">83.22%</td><td style="padding: 0.175rem 0.15rem;">0.89</td><td style="padding: 0.175rem 0.15rem;">0.90</td><td style="padding: 0.175rem 0.15rem;">113.88s</td><td style="padding: 0.175rem 0.15rem;">219.03W</td><td style="padding: 0.175rem 0.15rem;">10514.8MiB</td><td style="padding: 0.175rem 0.15rem;">87.67%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Base_16.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Base_16.png" style="margin: 0; height: 30px"/></td></tr><tr style="background-color: rgb(253, 248, 241); text-align: center; font-size: 80%;"><td style="padding: 0.175rem 0.15rem;">Base</td><td style="padding: 0.175rem 0.15rem;">32</td><td style="padding: 0.175rem 0.15rem;">81.02%</td><td style="padding: 0.175rem 0.15rem;">0.81</td><td style="padding: 0.175rem 0.15rem;">0.82</td><td style="padding: 0.175rem 0.15rem;">98.39s</td><td style="padding: 0.175rem 0.15rem;">223.67W</td><td style="padding: 0.175rem 0.15rem;">10511.9MiB</td><td style="padding: 0.175rem 0.15rem;">88.19%</td><td style="padding: 0.175rem 0.15rem;"><img alt="bert_report_CoLA_Base_32.png" class="preview-img bg normal" data-loaded="false" height="30px" src="/lab/bert/bert_report_CoLA_Base_32.png" style="margin: 0; height: 30px"/></td></tr></tbody></table>
</div>
