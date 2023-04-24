

> nnUNet粗略解读，详细看这篇[博客](https://zhuanlan.zhihu.com/p/134421623)

<!--more-->

# 网络架构

## 激活函数

- 将ReLU改为Leaky ReLU

## NET

- 2D Unet（在各向异性时，传统的3D方法不行）
- 3D Unet 用了这个，优缺点，图像大了不好训练
- 级联3D Unet 解决3D Unet的缺点

## 归一化策略

- 使用Instance normalization取代Batchnormalization([解释](https://link.zhihu.com/?target=https%3A//blog.csdn.net/z13653662052/article/details/84503024))

<img src="nnUNet%E8%A7%A3%E8%AF%BB/image-20230424185725341.png" alt="image-20230424185725341" />

# 网络拓扑的动态自适应

- 通过输入图像大小来自己调整**图像块大小和每个轴池化（卷积层）操作的数量**，

# 预处理

- Cropping：将所有数据的非零区域进行裁剪
- Resampling：我们知道医学影像数据不同的设备和设置会导致具有不同体素间距的数据，而CNN并不能理解这种体素间距的概念（别说CNN了，我一开始也不懂）。为了让网络学会空间语义，所有的病例重采样到相应数据集的体素间距中值，对数据和mask分别使用三阶样条差值和最近邻差值方法。
  注：如果重采样数据的形状中值可以作为3D U-Net中的输入图像（batch size=2）的4倍以上，则**使用级联U-Net网络**，且数据集需要重新采样到较低的分辨率。可以以2为因子增加体素间距（降低精度）直到4倍那个条件不满足。如果该数据集是各向异性的（比如1.2mm X 1.2mm X 3.5mm），首先更高分辨率的轴进行下采样直到与低分辨率轴匹配，然后才对所有轴同时进行下采样。
- Normalization：对于CT图像，首先搜集分割mask内的像素值，然后所有的数据截断到这些像素值的[0.5, 99.5]%，然后进行z-score标准化；对于MRI图像，直接进行z-score标准化。
  注：如果因为裁剪减少了病例平均大小的1/4或更多，则标准化只在非零元素的mask内部进行，并且mask外的所有值设为0。

# 训练

- [5折交叉验证](https://blog.csdn.net/u014264373/article/details/116241388?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522168233406516800211511492%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=168233406516800211511492&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_click~default-1-116241388-null-null.142^v86^insert_down28v1,239^v2^insert_chatgpt&utm_term=5%E6%8A%98%E4%BA%A4%E5%8F%89%E9%AA%8C%E8%AF%81&spm=1018.2226.3001.4187)

<img src="nnUNet%E8%A7%A3%E8%AF%BB/image-20230424190047518.png" alt="image-20230424190047518" style="zoom:80%;" />



策略：

- 优化器：Adam，初始学习率3e-4，每个epoch有250个batch。
- 学习率调整策略：计算训练集和验证集的指数滑动平均loss，如果训练集的指数滑动平均loss在近30个epoch内减少不够5e-3，则学习率衰减5倍。
- 训练停止条件：当学习率大于10-6且验证集的指数滑动平均loss在近60个epoch内减少不到5e-3，则终止训练。
- 数据扩充（On the fly）：随机旋转，缩放，elastic deformation，gamma矫正和镜像。[代码](https://link.zhihu.com/?target=https%3A//github.com/MIC-DKFZ/batchgenerators)
  注：如果3D U-Net的输入图像块尺寸的最大边长是最短边长的两倍以上，这种情况对每个2维面做数据增广。
- 级联U-Net的第二级接受前一级的输出作为输入的一部分，为了防止强co-adaptation，应用随机形态学操作（腐蚀、膨胀、开运算、闭运算）去随机移除掉这些分割结果的连通域。
- 图像块采样：为了增强网络训练的稳定性，强制每个batch中超过1/3的样本包含至少一个随机选择的前景。

