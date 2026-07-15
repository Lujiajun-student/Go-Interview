# 1. PyTorch基础数据结构

Pytorch主要使用的是类似NumPy的数组结构。

```python
import torch

# 创建张量
x = torch.tensor([[1.0, 2.0], [3.0, 4.0]])

# 随机张量
a = torch.randn(2, 3)

# 全1张量和全0张量
b = torch.ones(2, 3)
c = torch.zeros(2, 3)

# 查看形状
print(x.shape)
print(x.dtype)
print(x.device)

# 放到GPU
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
x = x.to(device)
print(x.device)

# 维度操作
x = torch.randn(4, 3, 32, 32) # 设为4张3通道的32x32图片
print(x.shape)

# 展平
y = x.view(4, -1) # 只保留第一维，剩余的维度自动计算
print(y.shape) # [4, 3072]

# 增加维度
z = torch.randn(10)
z = z.unsqueeze(0)
print(z.shape) # [1, 10]

# 去掉长度为1的维度
z = z.squeeze(0)
print(z.shape) # [10]

# 维度交换
x = torch.randn(2, 3, 4)
x = x.permute(0, 2, 1)
print(x.shape)

# 自动求导
x = torch.tensor(2.0, requires_grad=True)
y = x ** 2 + 3 * x + 1
# 求导
y.backward()
print("y = x ** 2 + 3 * x + 1 求导结果：", x.grad)
```

而一般操作张量采用的是nn.Module来操作。通过这个可以构建需要的神经网络。

```python
import torch
import torch.nn as nn

class SimpleMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, num_classes):
        super().__init__()

        # 定义网络层
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim), # 全连接层
            nn.ReLU(), # 激活函数
            nn.Linear(hidden_dim, num_classes) # 输出层
        )

    def forward(self, x):
        # 前向传播
        return self.net(x)

model = SimpleMLP(input_dim=784, hidden_dim=128, num_classes=10)
print(model)
```

`__init__`用来定义网络层，forward()能够定义数据如何流向这些层。只需要通过model(x)就能直接调用前向传播。

# 2. 训练自定义模型

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

# 选择设备
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# 数据预处理
transform = transforms.Compose([
    transforms.ToTensor(), # 图片转Tensor
    transforms.Normalize((0.1307,), (0.3081,)) # 均值和方差
])

# 下载数据集
train_dataset = datasets.MNIST(
    root='./data',
    train=True,
    download=True,
    transform=transform
)

test_dataset = datasets.MNIST(
    root='./data',
    train=False,
    download=True,
    transform=transform
)

# DataLoader加载数据
train_loader = DataLoader(
    train_dataset,
    batch_size=64,
    shuffle=True
)

test_loader = DataLoader(
    test_dataset,
    batch_size=1000,
    shuffle=False
)

# 定义神经网络
class SimpleMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(
            nn.Flatten(), # 展平，[B, 1, 28, 28] -> [B, 784]
            nn.Linear(28*28, 256), # 全连接层
            nn.ReLU(), # 激活函数
            nn.Dropout(0.2), # 随机失活20%
            nn.Linear(256, 128), # 全连接层
            nn.ReLU(), # 激活函数
            nn.Linear(128, 10) # 输出层
        )
    def forward(self, x):
        return self.net(x)

model = SimpleMLP().to(device)
# 损失函数和优化器
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=1e-3)

# 训练
def train_one_epoch(model, loader, criterion, optimizer, device):
    model.train()
    total_loss = 0.0
    correct = 0
    total = 0
    for images, labels in loader:
        images = images.to(device)
        labels = labels.to(device)
        # 清空梯度
        optimizer.zero_grad()
        # 前向传播
        outputs = model(images)
        # 计算损失
        loss = criterion(outputs, labels)
        # 反向传播
        loss.backward()
        # 参数更新
        optimizer.step()
        # 计算损失和准确率
        total_loss += loss.item() * images.size(0)
        pred = outputs.argmax(dim=1)
        correct += (pred == labels).sum().item()
        total += labels.size(0)
    avg_loss = total_loss / total
    acc = correct / total
    return avg_loss, acc

# 验证
@torch.no_grad()
def evaluate(model, loader, criterion, device):
    model.eval()
    total_loss = 0.0
    correct = 0
    total = 0
    for images, labels in loader:
        images = images.to(device)
        labels = labels.to(device)
        outputs = model(images)
        loss = criterion(outputs, labels)
        total_loss += loss.item() * images.size(0)
        pred = outputs.argmax(dim=1)
        correct += (pred == labels).sum().item()
        total += labels.size(0)
    avg_loss = total_loss / total
    acc = correct / total
    return avg_loss, acc

# 训练循环
epochs = 5
for epoch in range(epochs):
    train_loss, train_acc = train_one_epoch(model, train_loader, criterion, optimizer, device)
    test_loss, test_acc = evaluate(model, test_loader, criterion, device)
    print(
        f"Epoch{epoch+1}/{epochs} | "
        f"Train Loss: {train_loss:.4f}, Train Acc: {train_acc:.4f} | "
        f"Test Loss: {test_loss:.4f}, Test Acc: {test_acc:.4f}"
    )

# 保存模型参数
torch.save(model.state_dict(), "mnist_mlp.pth")
```

在这里，常见的损失函数有多种。

```python
nn.CrossEntropyLoss() # 多分类
nn.BCELoss() # 二分类，输入sigmoid后的概率
nn.BCEWithLogitsLoss() # 二分类，效果好一点
nn.MSELoss() # 回归
nn.L1Loss() # 回归，对异常值更稳健
```

# 3. CNN

CNN用来处理局部空间结构，用于图像识别、文本分类等任务。

主要的思想是用卷积核在输入数据中滑动，提取局部特征。

例如图片是二维矩阵，卷积核从左到右、从上到下扫描图片，浅层卷积核能够学到表面的变化，深层卷积核能够学习到眼睛、轮子等结构，更深层的卷积核能够得到整体结构。

CNN的特点是局部连接、权重共享、层级特征提取。

局部连接是指神经元不需要连接整张图片，而是只需要关注局部区域。比如3×3的卷积核只看3×3的区域。

权重共享指的是一个卷积核会处理一整张图片，减少参数使用量。

层级特征是指CNN从低级特征组合出高级特征。

简单的CNN表示如下。

```
输入图像->卷积->ReLU->池化->卷积->ReLU->池化->Flatten->全连接->Softmax
```

CNN最大的优点就是处理局部结构的数据，比如图像，由多个局部物体组成。

在后期，ResNet添加了残差，解决了深层CNN训练困难的问题。

但缺点是全局建模较弱，通常看的是局部区域，需要经过很多层才能感知全局信息。

而长程依赖比LSTM、RNN弱很多。

# 4. RNN

RNN是循环神经网络，专门为序列数据设计的神经网络。

普通的前馈神经网络了要求输入长度固定，而且每个输入之间没有顺序关系。而RNN的特点是当前的输出不仅依赖当前的输入，还依赖之前输入的隐藏状态。

这种添加先前隐藏状态的话，通过BPTT时间反向传播训练，需要从后往前计算，最前面的会叠加后面得到的所有隐藏状态的梯度，导致出现梯度消失或者梯度爆炸。

这种长序列的RNN，主要用于自然语言、机器翻译等。

# 5. LSTM

LSTM是长短期记忆网络。RNN只会维护一个隐藏状态，这个状态负责短期记忆，又负责长期记忆，负担重。而LSTM引入了细胞状态，也就是记忆单元。通过门控机制来控制信息的保留和遗忘。

LSTM主要有三个门。

遗忘门，决定上一时刻的记忆哪些要保留，哪些要去掉。输入们，决定当前输入的哪些信息需要写入记忆。输出门，决定当前记忆的哪些信息用于输出隐藏信息。

LSTM由于有记忆单元，所以适合长序列任务，如情感分类、机器翻译等。

# 5. 训练模型

预训练是指用海量的无标签文本来训练模型，基于模型基础的生成文本能力。

监督微调SFT是使用有标签的文本来给模型训练，让模型学习这些文本的输出格式。如指令模式、问答模式、多轮对话等。

奖励模型Reward Model是在强化学习中的组件。奖励模型是一个打分模型，根据用户问题和模型回答给出一个分数，判断这个回答是否符合人类偏好。这样，在强化学习中，能够促使模型生成分数更高的回答。

强化学习是让模型在产生回答后进行打分，根据打分结果调整生成token的概率。大模型强化学习RLHF中，用的方法是PPO，Proximal Policy Optimization是近段策略优化，每次更新模型时，不要让模型变化太大，因为强化学习比较不稳定。如果某一次RM给了高分，模型可能会过度学习这个模式导致输出出现问题。因此，强化学习需要添加约束，KL散度。优化目标是`奖励模型分数-偏离原模型的惩罚`。强化学习是为了解决多个正确答案中挑选更优答案的情况，也就是更自然更符合。但强化学习也有弊端，如果RM偏好长回答，模型可能会变得啰嗦，如果RM偏好自信，模型可能会胡编等。

DPO是直接偏好优化