# 外存归并排序时间复杂度推导

> 适用场景：数据总量 $N$ 远大于内存容量 $M$，磁盘 I/O 为主导代价。

---

## 一、符号约定

| 符号 | 含义 |
|---|---|
| $N$ | 待排序数据总量（记录数或字节数，下同） |
| $M$ | 可用内存大小 |
| $B$ | 磁盘块大小，**每次 I/O 传输 $B$ 单位数据** |
| $k$ | 归并**路数**（fan-in） |

---

## 二、算法流程回顾

1. **初始 run 生成**  
   每次读入 $M$ 数据，内排后输出一个长度为 $M$ 的有序 run。  
   共生成  
   $$R = \left\lceil \frac{N}{M} \right\rceil$$  
   个 runs。

2. **多路归并**  
   每轮同时归并 $k$ 个 runs，内存为每个输入 run 预留 **1 块缓冲区**（大小 $B$），外加 **1 块输出缓冲区**（大小 $B$）。  
   因此最大路数  
   $$k = \left\lfloor \frac{M}{B} \right\rfloor \quad (\text{常直接取 } \frac{M}{B} \text{ 忽略余数})$$

---

## 三、I/O 次数计算

| 阶段 | 说明 | I/O 次数 |
|---|---|---|
| **1. 初始 run 生成** | 读 $N$ 写 $N$ | $2\cdot\dfrac{N}{B}$ |
| **2. 归并** | 每轮读、写全集 $N$ | 每轮 $2\cdot\dfrac{N}{B}$ |

归并轮数 $L$ 满足 $k^L \ge R$，故  
$$L = \left\lceil \log_k R \right\rceil  
     = \left\lceil \log_{M/B} \left(\frac{N}{M}\right) \right\rceil.$$

**总 I/O 次数**  
$$\text{Total} = 2\cdot\frac{N}{B} \cdot \left(1 + L\right)  
                = 2\cdot\frac{N}{B} \cdot \left(1 + \log_{M/B}\frac{N}{M}\right).$$

---

## 四、渐近复杂度

利用换底公式  
$$\log_{M/B}\frac{N}{M} = \log_{M/B}\frac{N}{B} - 1,$$  
代入得  
$$\text{Total} = 2\cdot\frac{N}{B} \cdot \log_{M/B}\frac{N}{B}.$$

因此**外存归并排序的磁盘 I/O 复杂度**为  
$$\boxed{O\left(\frac{N}{B} \cdot \log_{M/B}\frac{N}{B}\right)}$$  
该结果常记为 *Sort*$(N,M,B)$，是外存排序模型的标准下界。