## 步骤，所有代码基于python3运行

1. **运行 `read-split.py`**
   - 该步骤会处理数据文件，并进行数据拆分。

2. **运行 `read-input.py`**
   - 该步骤将处理输入数据，并进行必要的数据准备。

3. **运行 `read-treat.py`**
   - 该步骤将执行数据的过滤和保存。

4. **运行 `train-force-e3.py`**
   - 该步骤用于训练模型。

# 安装依赖

此项目运行所需的 Python 库及其安装命令如下：

### 必要的库

- **numpy**: 用于高效的数值计算，特别是处理数组。
  - 安装命令: `pip install numpy`

- **pandas**: 用于数据处理和分析，尤其是操作数据框（DataFrame）。
  - 安装命令: `pip install pandas`

- **pytorch ≥2.4.1 with CUDA ≥12.2**: 用于深度学习训练和模型构建，CUDA 用于加速计算。
  - 安装命令:
    - 对于有 CUDA 支持的版本，可以使用如下命令安装：
      ```bash
      pip install torch==2.4.1+cu12.2 torchvision==0.15.1+cu12.2 torchaudio==2.4.1+cu12.2
      ```
    - 如果没有 CUDA 支持，直接安装 PyTorch：
      ```bash
      pip install torch==2.4.1
      ```

- **torch_scatter**: 用于在图神经网络中进行散射操作的库，支持稀疏张量操作。
  - 安装命令: `pip install torch-scatter`

- **e3nn**: 用于处理图形卷积神经网络中的对称性操作，专门用于处理对称张量、群表示和等变神经网络。
  - 安装命令: `pip install e3nn`

- **torch.utils.tensorboard**: 用于在训练过程中记录日志并可视化模型训练过程。
  - 安装命令: `pip install tensorboard`

- **scikit-learn**: 提供常用的机器学习工具，如数据预处理和模型评估等。
  - 安装命令: `pip install scikit-learn`

- **torch.amp (Automatic Mixed Precision)**: 提供混合精度训练，帮助提高训练速度。
  - 已包含在 PyTorch 安装包中，无需单独安装。

### 注意事项

- 在训练时，请勿一开始开启混合精度训练功能，因为数值可能会不稳定，导致梯度消失。建议先进行一些初步训练后再启用混合精度训练。



# 以下是代码过程分享
### 将以小问题解答，结合代码解释的方式进行分享。本人非专业coder，数理化基础也一般，如有错误，请望海涵。
# 1. NNP的整体架构是怎么样的？

NNP（Neural Network Potential）主要由两个核心部分组成：嵌入网络（EmbedNet）和主拟合网络（MainNet）。在 `train-force.py` 文件中，这两个部分分别由类 `EmbedNet` 和 `MainNet` 定义。

- **嵌入网络（EmbedNet）**：这是 NNP 中最关键的部分。它根据分子中不同原子及其周围的环境生成对应的描述符矩阵。每个原子的坐标和其周围环境原子通过嵌入网络被映射为一个描述符向量，多个环境原子共同组成一个描述符矩阵，捕捉了该原子在分子中的环境信息。

- **主拟合网络（MainNet）**：主拟合网络接受来自嵌入网络的描述符矩阵作为输入，并输出对应原子的分能量（即每个原子的贡献能量）。

### 描述：

1. 对于每个原子 \( i \) ，嵌入网络生成该原子的描述符矩阵 \( \mathbf{M}_i \)：
   $$
   \mathbf{M}_i = \text{EmbedNet}(\text{Atom}_i, \{\text{Neighbors}\})
   $$
   其中，\(\text{Atom}_i\) 表示原子 \(i\) 本身，\(\{\text{Neighbors}\}\) 表示与原子 \(i\) 相邻的其他原子。

2. 然后，将每个原子的描述符矩阵输入到主拟合网络中，得到原子的分能量 \( E_i \)：
   $$
   E_i = \text{MainNet}(\mathbf{M}_i)
   $$
   其中，\( \mathbf{D}_i \) 是从嵌入网络得到的描述符矩阵，\( E_i \) 是原子 \(i\) 的分能量。

3. 最终，分子的总能量 \( E_{\text{total}} \) 是所有原子分能量的总和：
   $$
   E_{\text{total}} = \sum_{i=1}^{N} E_i
   $$
   其中，\( N \) 是分子中的原子数量，\( E_i \) 是原子 \(i\) 的分能量。

### 损失函数定义:

在神经网络势模型中，能量和力的计算是关键部分。能量 \( E_i \) 是通过主拟合网络得到的，而力 \( F_i \) 则是通过能量对原子位置的梯度来计算的，即：

$$
F_i = -\nabla_{r_i} E
$$

因此，定义损失函数 \( L(p_\varepsilon, p_f, \delta) \) 为：

$$
L(p_\varepsilon, p_f, \delta) = \frac{1}{|N|} \sum_{i \in N} p_\varepsilon L_\delta(E_i - E_i^w) + p_f L_\delta(F_i - F_i^w)
$$

其中，\( L_\delta(x) \) 为 Huber 损失函数，定义为：

$$
L_\delta(x) =
\begin{cases}
\frac{1}{2} x^2, & \text{if } |x| \leq \delta \\
\delta (|x| - \frac{1}{2} \delta), & \text{if } |x| > \delta
\end{cases}
$$

- \( L_\delta(x) \) 是对误差项的 Huber 损失。
- \( p_\varepsilon \) 和 \( p_f \) 分别是能量误差和力误差的权重。
- \( \delta \) 是一个阈值，用于控制 L2 损失与 L1 损失的切换点。
#### 是不是感觉其实很简单？
没错，只用原子的位置作为输入，能量作为目标输出，神经网络就能有效地学习到原子之间的相互作用。通过嵌入网络，我们能够提取每个原子及其邻近环境的特征，从而生成原子的描述符矩阵。这些描述符矩阵被传递给主网络进行进一步处理，预测出每个原子的分能量。
##### 但是如果仅仅用常见的笛卡尔坐标作为输入，模型的泛化性能很难提高。
笛卡尔坐标只是描述原子在空间中的绝对位置，而并没有充分考虑到原子之间的相对关系或它们的相对环境。因此，单纯依赖笛卡尔坐标作为输入，无法有效捕捉原子之间的相对位置和局部环境，这限制了模型的泛化能力。

因此，我们使用相对广义坐标作为输入。广义坐标不仅考虑了原子的绝对位置，还通过转换为相对坐标或使用局部环境描述符来捕捉原子间的相对距离、角度等几何信息（目前只实现了相对距离，角度信息等等将会在以后进行更新）。通过广义坐标，模型不仅能够准确地预测能量和力，还能在不同结构和环境下具备更好的泛化能力。







# 2. 如何转换为广义坐标？

### 2.1 输入文件要求

`read-split.py` 和 `read-input.py` 是两个用于数据处理的脚本，用户无需手动将笛卡尔坐标转换为广义坐标。然而，对于用户提供的原始文件格式，我们有一定的要求。请参考我们提供的 `data.csv` 文件格式。

#### 示例内容：
40
……energy=-29654.71595871336 pbc="F F F"
C 4.65116481 -0.87536020 -0.95341924 -0.00226694 -0.00099080 -0.00055906 6
C 4.52871416 0.49243101 -0.68728938 0.00005086 0.00149402 -0.00094421 6
C 3.26602428 1.06821027 -0.44491504 -0.00220344 -0.00100710 0.00046995 6
C 2.10756895 0.23795426 -0.46852147 0.00088549 0.00012105 -0.00051350 6
……

#### 文件格式要求：

1. **编码格式**：文件应使用 UTF-8 编码。
2. **多帧结构**：多个分子结构应保存在同一个文件中，不同的结构之间通过表头区分。每个结构的表头应包含类似 `energy=xxxxxx` 的字符串，用于提取该结构的能量。
3. **坐标内容**：文件应包含 **8 列数据**，具体格式如下：
   - **原子名称**（如 C、H、O、N 等）
   - **笛卡尔坐标**：每个原子的 x、y、z 坐标。
   - **原子受力**：每个原子在 x、y、z 方向的受力值。
   - **原子序号**：每个原子的序号（如 6、1、8、7 等）。

请确保输入文件符合上述格式要求，便于数据处理和广义坐标转换。
