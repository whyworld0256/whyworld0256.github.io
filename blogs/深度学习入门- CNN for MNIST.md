# CNN for MNIST

本文展示了并详解用CNN来解决手写数字识别问题的全代码， 对于深度学习初学者，以及对pytorch不熟悉的人，我相信这会是非常好的学习资料。下面我会一段一段地解析代码：

请配合[官方文档](https://pytorch.org/docs/stable/index.html)使用！！

###  Preparing and Pre-processing the Dataset

##### Importing Packages

```python
import numpy as np # data processing
import pandas as pd # data processing, CSV file I/O (e.g. pd.read_csv)
import torch # PyTorch
from torch.utils.data import Dataset, DataLoader # data loading
from torchvision import transforms # data processing
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as opt # optimizer
from torch.utils.data.sampler import SubsetRandomSampler # data pre-processing
```

导入需要使用的库。

##### Preparing Data

```python
train_set = pd.read_csv('../input/digit-recognizer/train.csv')
test_set = pd.read_csv('../input/digit-recognizer/test.csv')
```

```python
VALID_SIZE = 0.1 # Percentage of data for validation

num_train = len(train_set)
indices = list(range(num_train))
np.random.shuffle(indices)
split = int(np.floor(VALID_SIZE * num_train))
train_indices, valid_indices = indices[split:], indices[:split]

train_sampler = SubsetRandomSampler(train_indices)
valid_sampler = SubsetRandomSampler(valid_indices)

print(f'training set length: {len(train_indices)}')
print(f'validation set length: {len(valid_indices)}')
```

- `VALID_SIZE`: 定义验证集占整个训练数据的比例
- `indices`: 整个训练数据的下标数组
- `np.random.shuffle(indices)`:   打乱 整个`indices` 下标数组

- `split`: 验证集的大小
- `train_indices, valid_indices = indices[split:], indices[:split]`: 分别拿到训练集和测试集在整个训练数据中的下标
- `train_sampler = SubsetRandomSampler(train_indices),valid_sampler = SubsetRandomSampler(valid_indices) `: 定义了两个`sampler`类，下面我们再讲这是什么。

```python
class DatasetMNIST(Dataset):
    def __init__(self, data, transform=None, labeled=True):
        self.data = data
        self.transform = transform
        self.labeled = labeled

    def __len__(self):
        return len(self.data)

    def __getitem__(self, index):  # Defines how Dataloader will load the items
        item = self.data.iloc[index]
        if self.labeled:
            x = item[1:].values.astype(np.uint8).reshape((28, 28))  # Pixel data
            y = item[0]  # Label
        else:
            x = item[0:].values.astype(np.uint8).reshape((28, 28))  # Pixel data
            y = 0  # 占位用，不影响行为
        if self.transform is not None:
            x = self.transform(x)
        return x, y
```

忘记说数据的格式了：一共有`785` 列，第一列是`label`, 其余`784` 列是`28 x 28` 个像素值。

- `transform` 相当于是一个工具箱，里面有你相对你的数据做的操作，我们下面再说。
- `labeled` 表明这些数据是否被打上了标签。
- ` x = item[1:].values.astype(np.uint8).reshape((28, 28))`: 取出一整行（除了第一列），把这`784` 个像素值压缩为`uint8` (降低存储空间), `reshape` 把这些数据转化为`28 x 28` 的矩阵格式。

```python
# Transform the dataset
transform_train = transforms.Compose([
    transforms.ToPILImage(), # allows the following transformation to be applied
    transforms.RandomAffine(degrees=10, translate=(0, 0.1), scale=(0.9, 1.1)), # random rotations, translations, and scaling
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))  # normalize data to speed up convergence for gradient descent
    # Mean Pixel Value / 255 = 33.31002426147461 / 255 = 0.1307
    # Pixel Values Std / 255 = 78.56748962402344 / 255 = 0.3081
    # IMPORTANT: I did not show how the mean and standard deviation are calculated. I looked up those two values 
    #            in the original MNIST, which is against the rules because the original MNIST included the test set. 
    #            Instead, we should take a sample from the training set and compute the mean and standard deviation. 
    #            If I have time, I'll add to this notebook and fix the problem.
])

transform_valid = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])

transform_test = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])
```

定义了三个`transform` ： `transform_train`, `transform_valid`, `transform_test`

- `transform_train`: 
  - `ToPILImage`: 把数据转化为`PILImage` 格式，为了下一步操作
  - `RandomAffine`:  随机仿射变换，为了数据增强
  - `ToTensor`: 把数据转化为张量
  - `Normalize`: 归一化，加快优化算法收敛

```python
# Creating datasets for training, validation, and test
train_data = DatasetMNIST(train_set, transform=transform_train)
valid_data = DatasetMNIST(train_set, transform=transform_valid)
test_data = DatasetMNIST(test_set, transform=transform_test, labeled=False)
```

分别建立三个`Dataset` 类，并赋予不同类型的`transform`

# Building Up the Network

```python
class ConvNet(nn.Module):
    def __init__(self):
        super(ConvNet, self).__init__()
        self.dr = nn.Dropout(p=0.4)
        self.conv1 = nn.Conv2d(1, 32, (3, 3))  # Output becomes 26x26
        self.bn1 = nn.BatchNorm2d(32)
        self.conv2 = nn.Conv2d(32, 32, (3, 3))  # Output becomes 24x24
        self.bn2 = nn.BatchNorm2d(32)
        self.conv3 = nn.Conv2d(32, 32, (5, 5), stride=(2, 2))  # Output becomes 10x10
        self.bn3 = nn.BatchNorm2d(32)
        self.conv4 = nn.Conv2d(32, 64, (3, 3))  # Output becomes 8x8
        self.bn4 = nn.BatchNorm2d(64)
        self.conv5 = nn.Conv2d(64, 64, (3, 3))  # Output becomes 6x6
        self.bn5 = nn.BatchNorm2d(64)
        self.conv6 = nn.Conv2d(64, 64, (5, 5), stride=(2, 2))  # Output becomes 1x1
        self.bn6 = nn.BatchNorm2d(64)
        self.fc1 = nn.Linear(64 * 1 * 1, 128)  # Num of features * dimension of the output 
        self.bn7 = nn.BatchNorm1d(128)
        self.fc2 = nn.Linear(128, 10)
        
    def forward(self, x):
        x = self.bn1(F.relu(self.conv1(x)))
        x = self.bn2(F.relu(self.conv2(x)))
        x = self.dr(self.bn3((F.relu(self.conv3(x)))))
        x = self.bn4(F.relu(self.conv4(x)))
        x = self.bn5(F.relu(self.conv5(x)))
        x = self.dr(self.bn6(F.relu(self.conv6(x))))
        x = x.view(-1, 64 * 1 * 1)
        x = self.dr(self.bn7(F.relu(self.fc1(x))))
        x = self.fc2(x)
        
        return x
```

- `super(ConvNet, self).__init__()`: 继承`nn.Module` 类的属性和方法
- `self.dr = nn.Dropout(p=0.4)`: dropout层，经典的过拟合方法
- `self.conv1 = nn.Conv2d(1, 32, (3, 3)) `: 第一层卷积层，`input_channel = 1 `, `output_channel = 32, kernel_size = 3x3`
- `self.bn1 = nn.BatchNorm2d(32)`: 第一层归一化层，输入参数为`channel` 数
- `self.conv2 = nn.Conv2d(32, 32, (3, 3))  # Output becomes 24x24`: 第二层卷积层，`input_channel = 32`, `output_channel  = 32, kernel_size = 3x3`
- `self.bn2 = nn.BatchNorm2d(32)` 第二层归一化层
- `self.conv3 = nn.Conv2d(32, 32, (5, 5), stride=(2, 2))`: 第三层卷积层，`input_channel = 32`, `output_channel  = 32, kernel_size = 5x5`
- `self.bn3 = nn.BatchNorm2d(32)`: 第三层归一化层
- `self.conv4 = nn.Conv2d(32, 64, (3, 3))`第四层卷积层，`input_channel = 32`, `output_channel  = 64, kernel_size = 3x3`
- `self.bn4 = nn.BatchNorm2d(64)` 第四层归一化层
- `self.conv5 = nn.Conv2d(64, 64, (3, 3))` 第五层卷积层，`input_channel = 64, ` `output_channel  = 64, kernel_size = 3x3`
- `self.bn5 = nn.BatchNorm2d(64)` 第五层归一化层
- `self.conv6 = nn.Conv2d(64, 64, (5, 5), stride=(2, 2)) `  第六层卷积层，`input_channel = 64, ` `output_channel  = 64, kernel_size = 5x5`
- `self.bn6 = nn.BatchNorm2d(64)` 第六层归一化层
- `self.fc1 = nn.Linear(64 * 1 * 1, 128)` 第一层线性层
- `self.bn7 = nn.BatchNorm1d(128)` 第七层归一化层
- `self.fc2 = nn.Linear(128, 10) `第二层线性层

# Training the Network

##### Hyperparameters

```python
# Hyperparameters
NUM_EPOCHS = 35
BATCH_SIZE = 64
LEARNING_RATE = 0.001`
```

- `NUM_EPOCHS` 训练次数
- `LEARNING_RATE` 学习率

##### Data Loading

```python
train_dataloader = DataLoader(dataset=train_data, batch_size=BATCH_SIZE, sampler=train_sampler)
valid_dataloader = DataLoader(dataset=valid_data, sampler=valid_sampler)
test_dataloader = DataLoader(dataset=test_data)
```

- `sampler` 取数的策略，`subsetrandomsampler(indices)` 表示从`indices` 中随机选取下标

##### Defining the Train Function

```python
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')  # Use CUDA when available
model = ConvNet().to(device)

criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=LEARNING_RATE)

# Reduce learning rate as the training progresses for producing more locally optimal results
lr_scheduler = torch.optim.lr_scheduler.ExponentialLR(optimizer, gamma=0.9)


def train():
    n_total_steps = len(train_dataloader)  # Displaying loss once at the every end of an epoch
    for epoch in range(NUM_EPOCHS):
        model.train()  # Sets the module in training mode
        for i, (images, labels) in enumerate(train_dataloader):
            images = images.to(device)
            labels = labels.to(device)

            # Forward pass
            outputs = model(images)
            loss = criterion(outputs, labels)

            # Backward and optimize
            optimizer.zero_grad()  #Sets the gradients of all optimized torch.Tensor to zero.
            loss.backward()  # Computes dloss/dx for every parameter x which has requires_grad=True
            optimizer.step()  # Updates the value of x using the gradient x.grad.
            if (i + 1) % n_total_steps == 0:
                print(
                    f'Epoch [{epoch + 1}/{NUM_EPOCHS}], Step [{i + 1}/{n_total_steps}, Loss = {loss.item():4f}]'
                )

        lr_scheduler.step()
        print('Current learning rate:', optimizer.param_groups[0]['lr'])
        get_valid_acc()  # Defined in 5.1. Checking Accuracy on Validation Set
```

- `model = ConvNet().to(device)` 把定义的模型拿到对应的设备上面去。
- `criterion = nn.CrossEntropyLoss()` 损失函数定义为交叉熵损失
- `optimizer = torch.optim.Adam(model.parameters(), lr=LEARNING_RATE)` 优化方法定义为`Adam`
- `lr_scheduler` 学习率调整器、

```python
# Test accuracy on validation set
def get_valid_acc():
    # Temporarily set the attribute requires_grad of tensors to False 
    # and deactivates the Autograd engine which computes the gradients with respect to parameters
    # as we do not need backpropagation for evaluation. 
    with torch.no_grad():
        # Sets the module in evaluating mode
        # Ignoring layers including dropout, etc. to produce the correct outputs
        model.eval()
        n_correct = 0
        n_samples = 0
        for images, labels in valid_dataloader:
            images = images.reshape(-1, 1, 28, 28).to(device)
            labels = labels.to(device)
            outputs = model(images)

            predictions = outputs.cuda().data.max(1, keepdim=True)[1]
            n_samples += labels.shape[0]
            n_correct += (predictions == labels).sum().item()

        acc = 100.0 * n_correct / n_samples
        print(f'Validation set accuracy = {acc}')
```

- `with torch.no_grad()`:我们不需要梯度来更新参数，并且不需要考虑`dropout` 层

- `predictions = outputs.cuda().data.max(1, keepdim=True)[1]` 返回 `outputs` 中最大值的位置，也就是当前的预测值

  

```python
train()  # Starts training

# Run the model on test set and generate submission.csv
with torch.no_grad():
    model.eval()
    results = torch.ShortTensor()
    for predict_images, _ in test_dataloader:
        predict_images = predict_images.reshape(-1, 1, 28, 28).to(device)
        predict_outputs = model(predict_images)
        test_predictions = predict_outputs.cpu().data.max(1, keepdim=True)[1]
        results = torch.cat((results, test_predictions), dim=0)

    submission = pd.DataFrame(np.c_[np.arange(1, len(test_set) + 1)[:, None], results.numpy()],
                              columns=['ImageId', 'Label'])
    submission.to_csv('/kaggle/working/submission.csv', index=False)
    check('/kaggle/working/submission.csv')  # Get test set accuracy
```

