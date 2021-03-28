---
title: GANS介绍
date: 2020-12-28
tags: 
	- 机器学习
categories:
	- 强化学习
---

### 1.结构

<div align=center><img src="1610005130361-d3374c7e-7456-4bba-b3a7-b7eb49fb6974-20210128162026163.png" width="60%" height="50%"></div>


- **生成器**

这里的生成器可以是任意可以输出图片的模型，比如最简单的全连接神经网络又或者是反卷积网络。
生成器输入的变量可以是用随机生成的变量作为输入即可，随机输入最好满足一个常见的分布，比如高斯分布 **
<!--more-->

- **判别器2**

任何的判别器类型，输入为图片，输出为图片的真伪标签**

- **博弈**

生成网络G的目标就是尽量生成真实的图片去欺骗判别网络D。而D的目标就是尽量把G生成的图片和真实的图片分别开来。


### 2.优化目标
判别器：让判别器分类真假样本的概率达到五五开，让概率-样本分布图达到一条1/2的直线
生成器：让生成器尽可能地减少生成样本与真实样本之间的差距，体现在生成样本的分布接近于真实样本的分布，让生成样本的分布曲线接近于真实样本的分布曲线

<div align=center><img src="1609914460948-9656b2ad-bae1-4804-8cca-cd6e26bbdf49.png" width="60%" height="50%"></div>

### 3.训练过程：

- 初始化判别器D的参数 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442539-285ae8fd-ff52-4b81-9700-ef05cb6e80c3.svg#align=left&display=inline&height=20&margin=%5Bobject%20Object%5D&originHeight=20&originWidth=17&size=0&status=done&style=none&width=17) 和生成器G的参数 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442536-5c3593f4-34c4-4d7b-8fd2-b2bce1f6e908.svg#align=left&display=inline&height=23&margin=%5Bobject%20Object%5D&originHeight=23&originWidth=17&size=0&status=done&style=none&width=17) 。
- 从真实样本中采样 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442546-2805568b-9816-4767-8f7c-730c37c3c246.svg#align=left&display=inline&height=13&margin=%5Bobject%20Object%5D&originHeight=13&originWidth=16&size=0&status=done&style=none&width=16) 个样本 { ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442617-e6fff283-b450-45be-8a93-fc8907d11b81.svg#align=left&display=inline&height=24&margin=%5Bobject%20Object%5D&originHeight=24&originWidth=104&size=0&status=done&style=none&width=104) } ，从先验分布噪声中采样 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442556-c55a146a-8e8b-4932-b9c8-ac9e61fad9f5.svg#align=left&display=inline&height=13&margin=%5Bobject%20Object%5D&originHeight=13&originWidth=16&size=0&status=done&style=none&width=16) 个噪声样本 { ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442556-e8fd7383-c80f-4919-8078-b0d358b6e283.svg#align=left&display=inline&height=24&margin=%5Bobject%20Object%5D&originHeight=24&originWidth=106&size=0&status=done&style=none&width=106) } 并通过生成器获取 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442621-30bfec79-28d2-4afa-af1e-c275d87968ad.svg#align=left&display=inline&height=13&margin=%5Bobject%20Object%5D&originHeight=13&originWidth=16&size=0&status=done&style=none&width=16) 个生成样本 { ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914442561-7ef7f2bb-7710-4814-904c-1973513e864e.svg#align=left&display=inline&height=24&margin=%5Bobject%20Object%5D&originHeight=24&originWidth=112&size=0&status=done&style=none&width=112) } 。固定生成器G，训练判别器D尽可能好地准确判别真实样本和生成样本，尽可能大地区分正确样本和生成的样本。
- **循环k次更新判别器之后，使用较小的学习率来更新一次生成器的参数**，训练生成器使其尽可能能够减小生成样本与真实样本之间的差距，也相当于尽量使得判别器判别错误。
- 多次更新迭代之后，最终理想情况是使得判别器判别不出样本来自于生成器的输出还是真实的输出。亦即最终样本判别概率均为0.5。





### 4.损失函数
判别器使用交叉熵计算损失，其中pi表示真实分布，qi表示生成样本的分布

<div align=center><img src="1609914799643-975cc081-1fb5-4085-9a17-fae9d0e9b590.png" width="60%" height="50%"></div>

因为这里是二元分类问题，判别样本真假

<div align=center><img src="1609914908755-33c89db7-e274-44d8-bf78-d58802988970.png" width="60%" height="50%"></div>

其中，假定 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914940477-5d124bef-cb15-4dfe-9278-6c25429b8d04.svg#align=left&display=inline&height=16&margin=%5Bobject%20Object%5D&originHeight=16&originWidth=18&size=0&status=done&style=none&width=18) 为正确样本分布，那么对应的（ ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914940457-c25d5d10-abf7-492a-8570-c5cdcb31ea2a.svg#align=left&display=inline&height=20&margin=%5Bobject%20Object%5D&originHeight=20&originWidth=50&size=0&status=done&style=none&width=50) ）就是生成样本的分布。 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914940512-461170ea-7944-4535-86a2-f55444e71dd1.svg#align=left&display=inline&height=17&margin=%5Bobject%20Object%5D&originHeight=17&originWidth=15&size=0&status=done&style=none&width=15) 表示判别器，则 ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914940459-549fd31b-9254-4dcd-b0c2-4a9ac75b6e50.svg#align=left&display=inline&height=23&margin=%5Bobject%20Object%5D&originHeight=23&originWidth=49&size=0&status=done&style=none&width=49) 表示判别样本为正确的概率， ![](https://cdn.nlark.com/yuque/0/2021/svg/2648023/1609914940600-fb5fafc4-0798-4d44-a981-81f2ca073bdd.svg#align=left&display=inline&height=27&margin=%5Bobject%20Object%5D&originHeight=27&originWidth=110&size=0&status=done&style=none&width=110) 则对应着判别为错误样本的概率，GAN对交叉熵的改进在于，为了让生成器以假乱真，判别器输出的概率稳定在0.5，0.5，故对上边的yi取1/2

<div align=center><img src="1609983349967-7087eb25-37b6-4bd8-90eb-09dc4953bba8.png" width="60%" height="50%"></div>

### 5.损失函数意义
D(y)表示判别器判别一个样本为真实的概率，其中x是真实样本，G(z)表示生成器经过随机噪声z产生的假样本被判别器判别为真的概率，所以前半部分就是判别器判别真实样本为真的概率期望，后半部分是判别器判别假样本为假的概率期望，为了让判别器判别更加准确，首先是max，尽可能让判别器判别准确，再为了让生成器可以以假乱真，D的能力越强，D(x)应该越大，D(G(x))应该越小。这时V(D,G)会变大。因此式子对于D来说是求最大(内层取max），而生成器G希望目的就是让判别器判别出现更多的失误，D(G(z))尽可能得大，这时V(D, G)会变小，也就是最外边的min



### 6.算法

<div align=center><img src="1609987534946-5568cebc-0d23-4bc7-a175-73a5872b732e.png" width="60%" height="50%"></div>




### 7.与强化学习的区别
强化学习是一方为智能体，一方为环境（环境相对不变），让智能体不断的进化，智能体不断与环境交互，先根据模型输出action后，再根据环境给予的奖励来调整模型。
而GAN是在强化学习上的演变。GAN则是相当于两个environment，判别器目的是让自己判别尽可能的准确（即让自己极可能识别出更多的假样本），体现在损失函数中是体现在max(D,G)是固定生成器G，使得判别真假样本概率极可能地准确，其在做出action（判断输入样本真假概率），才能根据真假样本标签给予的奖励来调整自己的模型（对于生成器：判断正确是正奖励，判断失误是负奖励）；生成器则是做出action后（生成样本），再根据判别器判断失误奖励（对于判别器：判断失误是正奖励，判断正确是负奖励）来提高自己；判别器D试图使得D(G(z))概率接近于0，生成器试图使得D(G(z))概率为1






