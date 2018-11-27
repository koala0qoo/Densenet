# Densenet

### 文件说明

本模型在 TensorFlow 的 Slim 图像分类框架的基础上修改而成。其中 densenet 的模型定义通过 densenet.py 实现，位于 nets 文件夹下。

### 模型实现

#### 整体网络结构：
- 参照论文中 DenseNet-161 的结构，网络由 4 个 Dense Block 和 3 个 Transition Layer 构成。
- 每个 Dense Block 依次包含 6、12、36、24 个 Layer，而每个 Layer 均包含一组 BN-ReLU-Conv-BN-ReLU-Conv 操作。其中第一个卷积层为 1×1（或称为Bottleneck Layer），目的是融合各个通道的特征，减少输入的 Feature Map 数量。第二个卷积层为 3×3，用于提取特征。在每个 Dense Block 内部，Feature Map 的大小不变。
- Transition Layer 包含一个归一化层（Batch Normalization）、一个 1×1 的卷积层（用于减小 Feature Map 的数量）以及一个 2×2 的平均池化层（用于减小 Feature Map 的大小）。
- 在进入第一个 Dense Block 之前，先进行一次 7×7、步长为2的卷积操作，以及一次 3×3、步长为2的最大池化；由最后一个 Dense Block 输出之后，先经过一次全局平均池化转化为特征向量，再经过归一化并激活后送入最后的全连接层（输出层）进行分类。

#### 对稠密连接的理解：
- 在每个 Dense Block 内部，各 Layer 之间采取稠密链接的策略，在各个层之间建立直接连接，即每一个 Layer 将前面所有 Layer 的输出合并起来作为其输入。
- 由于每个 Layer 都可以重新使用前面所有 Layer 计算出的特征图信息，提高了特征的利用效率，使模型更加紧凑，参数使用效率更高。
- 这种连接方式使得每一层与输入信息和 loss 的连接更加紧密（途经更少的中间层），因此特征和梯度的传递会更加有效。一方面，特征的有效传递鼓励每一层都独立学习对分类有直接作用的特征；另一方面，梯度的有效传递可以减轻梯度消失现象，便于训练更深的网络。

#### 对 Growth 的理解:
- Growth 用于控制每个卷积层输出的 Feature Map 数量，即网络宽度。由于每一个 Layer 的输出都要和前面所有 Layer 的输出叠加起来作为当前的 Feature Maps，如果一个 Dense Block 中有 L 个 Layer，每层输出 k 个 Feature Map，则该 Dense Block 最终输出 k×(L-1)+ k0 个 Feature Map。从这个角度看，Growth 实际上控制了每一个 Layer 为该 Dense Block 最终输出的特征图增加了多少信息。
- 与 Growth 一起控制网络宽度的还有 Bottleneck Layer 和 Compression Rate。Bottleneck Layer 在每个 3×3 的卷积前面增加了一个 1×1 的卷积操作，将输入特征图的数量降为 Growth 的 4 倍；Compression Rate 在 Transition Layer 中增加 1×1 的卷积操作，将上一层输出的特征图降为原来的 Compression Rate 倍，这里设为 0.5。
- 以上三者共同的作用使得网络变窄，参数减少，既增加了计算效率，又在一定程度上抑制了过拟合。
