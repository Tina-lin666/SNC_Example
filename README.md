# SNC时延上界计算程序

参考论文：J. Xie and Y. Jiang, "A network calculus approach to delay evaluation of IEEE 802.11 DCF," *IEEE Local Computer Network Conference*, Denver, CO, 2010, pp. 560-567, doi: 10.1109/LCN.2010.5735773.

实现了论文中基于矩生成函数的DCF随机接入的时延上界。同时实现了TDMA形式的时延上界计算。参数调用方法参考`main.py`，运行`python main.py`后，可以得到以下两条示例的时延上界：

![DCF Delay Upper Bound](.\imgs\dcf_delay.png)

![TDMA Delay Upper Bound](.\imgs\tdma_delay.png)

## Example

用法示例：

```python
from get_moments import dcf_delay_bound
from tdma_reward import tdma_delay_bound

# maximum transmission try
k = 6
# maximum content window grow times
m = 4
# initial content window size
cw = 32
# number of nodes in the same communication range
n = 10
# channel busy slot number because of success send
ts = 256
# channel busy slot number because of collision
tc = 5

sigma = 0.0001 # slot duration (unit: second)

xs = range(2000, 19000, 1000)
dcf = [dcf_delay_bound(x, n, k, m, cw, ts, tc, sigma) for x in xs]
print("DCF Delay Upper Bound:")
print(zip([x * sigma for x in xs], dcf))

arrival_rate = 10
link_rate = [4, 3, 5]
portion = [1, 1, 0.5]
max_packet_length = 12
bound = tdma_delay_bound(
        arrival_rate,
        link_rate,
        portion,
        max_packet_length)

xs = range(5, 20)
tdma = [bound(x) for x in xs]
sigma = 0.1

print("TDMA Delay Upper Bound:")
print(zip([x * sigma for x in xs], tdma))
```

## Random access network calculus

### 1. Probability

$$
p_c:=1-e^{（1-N)p_a},\\ p_a:=\frac{\sum_{k=0}^{K}p_c^k}{\sum_{k=0}^K}\mu_kp_c^k,\\ p_s=(N-1)P_a(1-p_a)^{N-2}
$$

- $p_c$：表示$N-1$个不相关节点（STA）中的至少一个在同一时隙中传输的冲突概率；
- $p_a$：表示在随机时隙中存在节点尝试发送报文的概率(可理解为传输概率)，是一个常数，与退避阶段无关
- $p_s$：表示只有一个非相关STA成功占用信道的传输概率

### 2. Moments

1)  每包服务时间：
$$
\delta=C+B\sigma+I+t_s
$$


一阶矩（平均服务时间）：$M_\delta^1=M_c^1+M_{B+I}^1+t_s$

- $C = \kappa \ t_c$ 表示$\kappa$次冲突的总和($t_c$：冲突的持续时间)
- $B=\sum_{k=0}^\kappa b_k$ 表示kth 退避阶段的退避间隔的总和（$b_k$表示kth退避阶段的退避间隔）
- $I$:  表示指数退避时发现信道被占用后的等待时间
- $t_s$: 表示由于成功传输信道被占用的间隔时间

2) $X_i$表示退避计数器递减1的持续时间，$i = 1,2,3...$
$$
M_X^1=(1-p_c)\sigma+p_s(t_s-t_c)+p_ct_c
$$

$$
M_X^2=(1-p_c)\sigma^2+p_s(t_s^2-t_c^2)+p_ct_c^2
$$

$$
B\sigma+I=\sum_{i=1}^BX_i
$$
​	等式右端为复合随机变量，一阶二阶矩为：
$$
M_{B+I}^1=M_B^1M_X^1,\ \ M_{B+I}^2=M_B^1M_X^2+(M_B^2-M_B^1)(M_X^1)^2
$$
3)
$$
M_C^1=t_c\sum_{k=1}^Kp_c^k,\ \ M_C^1=(t_c)^2\sum_{k=1}^K(2k-1)p_c^k
$$
4) $\triangle=C+B\sigma+I$

则，
$$
M_\triangle^1=M_C^1+M_{B+I}^1,\ \ M_\triangle^2=M_C^2+M_{B+I}^2+2M_C^1M_{B+I}^1
$$


### 3. Bound

$$
p\{\delta>x\}\leq \inf \left[\frac{M_\Delta^1}{x-t_s},\frac{M_\Delta^2}{(x-t_s)^2}\right]
$$

​	该Bound是针对单个报文的服务时间，没有考虑排队时延，对于单个报文，服务时间和时延等价，即$\delta(n)=D(n)$,不等式右端则为delay-bound，由这一不等式便可画出delay_bound曲线图

## Reservation  access network calculus

### 1 .前提假设

有以下假设：
1. 给定一个任务，泊松到达， $\lambda$已知。
2. 给定该任务要经过的每条链路可以给该任务提供的处理速率（相当于每条链路的总速率已知，给各个任务分配的比例也已知）。
3. 每个节点都是收到立刻转发，不做任何处理。

于是，相当于结点只做队列缓存功能，不需要单独考虑，每条链路实际上是网络演算中的一个server ，该演算服务器模型采用delay-rate server model,即service curve为：
$$
\beta(t) = r(t-T)^+
$$
​	 其中，$r$为考虑业务分配比例后的链路速率

​	由服务器模型的级联特性：
$$
\beta=\beta^1\otimes\beta^2\cdots\beta^n
$$
​	则得：
$$
\beta(t)=\min(r_1,r_2,\cdots r_n)(r-\sum_{i=0}^nT_i)^+
$$

​	另外，bounding function: $g(x)=0$

### 2. Bound

由网络演算时延上界结论：
$$
P\{D(t)>h(\alpha +x,\beta)\}\leq f\otimes g(x)
$$
其中$h(\alpha ,\beta)=\sup_{s\geq 0}\{\inf\{\tau \geq0:\alpha(s)\leq\beta(s+\tau)\}\}$

经一系列推导可得：
$$
P\{D(t)>\frac{x}{r}+T\}\leq \inf_{0\leq y\leq x} \left[ \sum_{k=\lceil y+\lambda t\rceil}^\infty \left\{  \frac{e^{-\lambda t}[\lambda t]^k}{k!}\right\}\right]
$$

令$D=\frac{x}{r}+T$,则有：
$$
P\{D(t)>D\}\leq \inf_{0\leq y\leq r(D-T)} \left[ \sum_{k=\lceil y+\lambda t\rceil}^\infty \left\{  \frac{e^{-\lambda t}[\lambda t]^k}{k!}\right\}\right]
$$
将y取最大值，右式求和达最小值，即$y=r(D-T)^+$