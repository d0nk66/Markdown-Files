# CRSF协议数据帧解析

### CRC8校验的原理

CRC8 是一种 8 位的循环冗余校验算法，它通过对数据进行异或运算来计算一个校验值。
假设0xD5 是该算法使用的生成多项式。

0xD5 用二进制表示是 `11010101`，表示生成多项式为：

​		$x^8 + x^7 + x^6 + x^4 + x^2 + 1$

这就是 CRC8 校验运算的常数，它决定了数据帧如何被校验和运算。

### CRC8 校验的基本运算步骤

CRC8 的校验过程基本如下：

1. **初始化：** 通常将 CRC 初始值设置为 0x00 或 0xFF，具体取决于协议标准。
2. **逐字节处理：** 将数据流中的每个字节与当前 CRC 进行运算。每次操作时，都会将 CRC 和当前字节做异或运算。
3. **左移和生成多项式处理：** 如果在某一轮异或运算后，CRC 的最高位是 1，那么将 CRC 与生成多项式 0xD5 进行异或。否则，继续左移 CRC 一位，重复此操作直到所有字节处理完毕。
4. **得到 CRC 值：** 最终，经过所有字节的处理后，CRC 就是最后得到的校验和。

如果还是不太明白，这里有一段示例代码：

``````python
def crc8(data, polynomial=0xD5, init_crc=0x00):
    crc = init_crc
    for byte in data:
        crc ^= byte  # 与当前字节做异或
        for _ in range(8):  # 处理8位
            if crc & 0x80:  # 如果CRC的最高位为1
                crc = (crc << 1) ^ polynomial  # 左移并与多项式做异或
            else:
                crc <<= 1  # 只左移
            crc &= 0xFF  # 保持CRC为8位
    return crc

# 示例数据
data = [0x12, 0x34, 0x56]
checksum = crc8(data)
print(hex(checksum))
``````

运算后得到的结果为`0xAD`。

### CRSF数据帧结构

接收CRSF数据包的缓冲数组共有26字节，结构如下：

1. 接收设备地址（一般为**0xC8**）
2. 数据帧总长度（**22**字节对应**16**个channel）
3. 数据帧类型（一般为`0x16`，这是RC遥控器数据帧的类型，由协议定义，一般不变）
4. 各个通道的数据从第四个字节开始，**22**字节共**176bit**，对应16个channel，每个channel的数据长度为**11bit**，11bit对应的就是每个channel的值（0~2047）。
5. 最后一位采用CRC8校验，原理见上文。

### 解析数据帧中的22字节数据

成功将26字节存到缓冲数组中时（此处假设缓冲数组为data[26]），由上文**数据帧结构**可知前三字节为帧头，故从数据帧的第四字节data[3]开始处理，需要将2个甚至3个字节的数据拼成一个11bit的数据，如下图所示：

[![image-20250402034338550.png](https://www.helloimg.com/i/2025/04/02/67ec528d9609a.png)](https://www.helloimg.com/i/2025/04/02/67ec528d9609a.png)

上图中，箭头指向左边代表作为高位，指向右边代表作为低位，采用不同的颜色表示不同的channel。

data[3]整个字节作为**channel1**的低八位，原地不动，这样data[4]就需要整体左移8位，然后与data[3]作**按位或**运算，再将结果与**0x07FF**作**按位与**运算，0x07FF是个11位掩码，只截取**按位或**运算后的低11位数据，运算结果作为channel1的值。

也就是说，data[4]的低三位被放到data[3]的左侧作为了channel1的**高3位**，代码表达如下：

```C
channels[0] = ((uint16_t)data[3] | ((uint16_t)data[4] << 8)) & 0x07FF;
```

以此类推，**data[4]**没用到的**高5位**就要**右移3位**作为channel2的**低5位**，这时候channel2高位还差6位，于是要把**data[5]**的**低5位**作为channel2的**高6位**，于是data[5]就需要左移5位，之后作**按位或**运算，再和掩码作**按位与**运算，就得到了channel2的11bit信息，代码表达如下：

``` C
channels[1] = ((uint16_t)data[4] >> 3 | ((uint16_t)data[5] << 5)) & 0x07FF;
```

接着往下写，就会遇到3个字节拼一个channel数据的情况了。

由于data[5]只用到了**低6位**，所以它的**高2位**要拿来作channel3的**低2位**，所以要把data[5]**右移6位**，然后再把data[6]**左移2位**作为中间8位，这样下来还缺1位，把data[7]的最低位拿来作channel3的最高位（就是把data[7]左移10位），再搞**按位或、按位与**之后就可以作为channel3的值了，代码表达如下：

``` C
channels[2] = ((uint16_t)data[5] >> 6 | ((uint16_t)data[6] << 2) | ((uint16_t)data[7] << 10)) & 0x07FF;
```

接着写下去就可以得到channel1~channel16的值。

### 数据映射

得到16个channel的数据之后，需要将它们映射为ELRS_Data结构体中的变量。

ELRS_Data结构体定义如下：

``````C
typedef struct
{
    // ---------------- RC通道数据 (CRSF_FRAMETYPE_RC_CHANNELS_PACKED) ----------------
    uint16_t channels[16]; // 存储 16 个通道的原始数据(每个通道的值介于0~2047之间)

    float Left_X;  // 左摇杆 X
    float Left_Y;  // 左摇杆 Y
    float Right_X; // 右摇杆 X
    float Right_Y; // 右摇杆 Y
    float S1;      // 左滑块
    float S2;      // 右滑块

    // 开关或按键
    uint8_t A;
    uint8_t B;
    uint8_t C;
    uint8_t D;
    uint8_t E;
    uint8_t F;

    // ---------------- Link 数据 (CRSF_FRAMETYPE_LINK_STATISTICS) ----------------
    uint8_t uplink_RSSI_1;
    uint8_t uplink_RSSI_2;
    uint8_t uplink_Link_quality;
    int8_t uplink_SNR;
    uint8_t active_antenna;
    uint8_t rf_Mode;
    uint8_t uplink_TX_Power;
    uint8_t downlink_RSSI;
    uint8_t downlink_Link_quality;
    int8_t downlink_SNR;

    // ---------------- 心跳数据 (CRSF_FRAMETYPE_HEARTBEAT) ----------------
    uint16_t heartbeat_counter;

} ELRS_Data;
``````

首先是摇杆映射，这里采用了一种带中值映射函数，代码如下：

``````C
// 映射函数
float float_Map(float input_value, float input_min, float input_max, float output_min, float output_max)
{
    float output_value;
    if (input_value < input_min)
    {
        output_value = output_min;
    }
    else if (input_value > input_max)
    {
        output_value = output_max;
    }
    else
    {
        output_value = output_min + (input_value - input_min) * (output_max - output_min) / (input_max - input_min);
    }
    return output_value;
}

// 带中值映射函数
float float_Map_with_median(float input_value, float input_min, float input_max, float median, float output_min, float output_max)
{
    float output_median = (output_max - output_min) / 2 + output_min;//计算输出区间的中值
    if (input_min >= input_max || output_min >= output_max || median <= input_min || median >= input_max)
    {
        return output_min;
    }

    if (input_value < median)
    {
        return float_Map(input_value, input_min, median, output_min, output_median);
    }
    else
    {
        return float_Map(input_value, median, input_max, output_median, output_max);
    }
}
``````

线性映射函数可以将输入值从一个区间**线性映射**到另一个区间。

带中值的映射函数则在线性映射函数的基础上引入了一个中值`median`，它表示输入值中一个非常重要的拐点位置，从而将输入区间拆成两段分别映射，示例如下：

假设输入为：

```C

input_value = 3
input_min = 0
input_max = 10
median     = 5
output_min = 0
output_max = 100
```

这时`output_median = (100 - 0)/2 + 0 = 50`，由于`input_value < median`，所以执行：

```C
float_Map(3, 0, 5, 0, 50)
= 0 + (3 - 0)*(50 - 0)/(5 - 0)
= 30
```

并且调整`median`的值可以改变映射的偏向，接下来演示一种非线性映射：

假设输入为：

```C
input_value = 3
input_min = 0
input_max = 10
median     = 2
output_min = 0
output_max = 100
```

此时有`output_median = (100 - 0)/2 + 0 = 50`，并且`input_value > median`，所以执行：

```C
float_Map(3, 2, 10, 50, 100)
= 50 + (3 - 2)*(100 - 50)/(10 - 2)
= 50 + 1 * 50 / 8 = 56.25
```

虽然 3 处于整个输入区间 0~10 的前段，但因为中值设在 2，它将落入右半边，输出结果**略高于50**，更偏向右区间。

**综上**，由于摇杆在中心位置较为敏感，所以摇杆的映射就可以采用非线性映射，代码表达如下：

```C
//这里为了方便阅读，省去了变量前的结构体名称
Left_X = float_Map_with_median(channels[3], 174, 1808, 992, -100, 100);
Left_Y = float_Map_with_median(channels[2], 174, 1811, 992, 0, 100);
Right_X = float_Map_with_median(channels[0], 174, 1811, 992, -100, 100);
Right_Y = float_Map_with_median(channels[1], 174, 1808, 992, -100, 100);
S1 = float_Map_with_median(channels[8], 191, 1792, 992, 0, 100);
S2 = float_Map_with_median(channels[9], 191, 1792, 992, 0, 100);
```

映射完成后就可以在其他处理函数中使用这些变量了。
