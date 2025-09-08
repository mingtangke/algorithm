# 外存排序的归并排序的时间复杂度推导

## 介绍

外存排序（外部排序）用于处理大数据集，当数据无法全部放入内存时。归并排序是常用的外部排序算法。时间复杂度主要考虑磁盘I/O操作，因为磁盘访问比内存访问慢得多。

## 参数定义

- $N$：外存中的数据总量（以字节或记录数表示）
- $M$：内存的总容量（以字节表示）  
- $B$：数据块的大小（以字节表示），每次磁盘I/O操作读写一个块

## 初始阶段

首先，将数据分成多个块，每个块大小为 $B$。读入内存进行排序，产生排序后的运行（runs）。每个运行的大小为 $M$（内存容量），因此初始运行的数量为：

$$R = \frac{N}{M}$$

初始阶段的I/O操作：读取所有数据一次和写入所有数据一次，所以I/O次数为：

$$2 \times \frac{N}{B}$$

## 归并阶段

进行多路归并。归并的路数 $k$ 受内存限制。内存需要为每个运行分配一个输入缓冲区，每个缓冲区大小为 $B$，因此：

$$k \leq \frac{M}{B}$$

通常取 $k = \frac{M}{B}$。

初始运行数 $R = \frac{N}{M}$。归并轮数 $L$ 满足 $k^L \geq R$，所以：

$$L = \left\lceil \log_k R \right\rceil$$

由于 $k = \frac{M}{B}$ 和 $R = \frac{N}{M}$，我们有：

$$L = \left\lceil \log_{\frac{M}{B}} \left(\frac{N}{M}\right) \right\rceil$$

每轮归并需要读取所有数据并写入所有数据，因此每轮的I/O次数为 $2 \times \frac{N}{B}$。

总I/O次数为：

$$2 \times \frac{N}{B} \times L = 2 \times \frac{N}{B} \times \log_{\frac{M}{B}} \left(\frac{N}{M}\right)$$

## 简化时间复杂度

注意到：

$$\log_{\frac{M}{B}} \left(\frac{N}{M}\right) = \log_{\frac{M}{B}} \left(\frac{N}{B}\right) - \log_{\frac{M}{B}} \left(\frac{M}{B}\right) = \log_{\frac{M}{B}} \left(\frac{N}{B}\right) - 1$$

因此，总I/O次数：

$$2 \times \frac{N}{B} \times \left(\log_{\frac{M}{B}} \left(\frac{N}{B}\right) - 1\right) = 2 \times \frac{N}{B} \times \log_{\frac{M}{B}} \left(\frac{N}{B}\right) - 2 \times \frac{N}{B}$$

对于大的 $\frac{N}{B}$，主导项是 $\frac{N}{B} \times \log_{\frac{M}{B}} \left(\frac{N}{B}\right)$，所以时间复杂度为：

$$O\left(\frac{N}{B} \log_{\frac{M}{B}} \left(\frac{N}{B}\right)\right)$$

## 结论

外存排序的归并排序的时间复杂度（基于磁盘I/O次数）为：

$$\frac{N}{B} \times \log_{\frac{M}{B}} \left(\frac{N}{B}\right)$$

其中：
- $N$ 是数据总量
- $B$ 是数据块大小  
- $M$ 是内存容量

这个复杂度表明，减少数据块数量 $\frac{N}{B}$ 或增加归并路数 $\frac{M}{B}$ 都能提高排序效率。