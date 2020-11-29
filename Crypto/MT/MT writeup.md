本题主要考察对伪随机数发生算法MT算法的掌握，当泄露至少624个32比特连续随机数时，能否构建出梅森旋转内部的状态，并回溯预测随机数。

题目对生成的伪随机数进行了简单的对相邻随机数的异或加密，首先进行解密

```python
f = open('output.txt', 'r').read()
xor_out = f.split("\n")[:-1]
for i in range(len(xor_out) - 1, 0, -1):
    xor_out[i - 1] = int(xor_out[i]) ^ int(xor_out[i - 1])
```

由于产生随机数时，梅森旋转内部状态会进行提取，所以先进行逆向提取

```python
def right(result, shift):
    i, value = 0, 0
    while i * shift < 32:
        part_mask = ((kMaxBits << (32 - shift)) & kMaxBits) >> (i * shift)
        part = result & part_mask
        result ^= part >> shift
        value |= part
        i += 1
    return value


def left(result, shift, mask):
    i, value = 0, 0
    while i * shift < 32:
        part_mask = ((kMaxBits >> (32 - shift) & kMaxBits)) << (i * shift)
        part = result & part_mask
        result ^= (part << shift) & mask
        value |= part
        i += 1
    return value

def _convert(m):
    m = right(m, 18)
    m = left(m, 15, 0xefc60000)
    m = left(m, 7, 0x9d2c5680)
    m = right(m, 11)
    return m

state = [_convert(int(i)) for i in num]
```

得到内部状态后，回溯前一组624个32比特的随机数

此处应该进行倒推，从现在已知的最后一个随机数开始推导

已知new_state[623],new_state[0],new_state[396],推导old_state[623]

通过new_state[623]和new_state[396]推导old_state[623]的最高位

通过new_state[622]和new_state[395]推导old_state[623]的后31位

以此类推 得到前一组随机数

```python
def backtrace(state):
    for i in range(623, -1, -1):
        res = 0
        tmp = state[i]
        tmp ^= state[(i + 397) % 624]
        if (tmp & 0x80000000) == 0x80000000:
            tmp ^= 0x9908b0df
        res = (tmp << 1) & 0x80000000
        tmp = state[(i - 1 + 624) % 624]
        tmp ^= state[(i - 1 + 397) % 624]
        if (tmp & 0x80000000) == 0x80000000:
            tmp ^= 0x9908b0df
            res |= 1
        res |= (tmp << 1) & 0x7fffffff
        state[i] = res
    return state
```

由于secret.txt中含有flag后产生的1500个随机数，要进行三次backtrace即可找到flag所在的随机数组。找到对应位置，再对相应的内部状态进行提取

```python
def convert(m):
    m ^= (m >> 11)
    m ^= (m << 7) & 0x9d2c5680
    m ^= (m << 15) & 0xefc60000
    m ^= (m >> 18)
    return m

for _ in range(3):
    	state = backtrace(state)
value = [convert(i) for i in state]
flag = value[-(1500 - 624 * 2) - 4:-(1500 - 624 * 2)]
flag = "HECTF{" + "".join([hex(i)[2:] for i in flag]) + "}"
```

完整代码

```python
import random
kMaxBits = 0xffffffff


def right(result, shift):
    i, value = 0, 0
    while i * shift < 32:
        part_mask = ((kMaxBits << (32 - shift)) & kMaxBits) >> (i * shift)
        part = result & part_mask
        result ^= part >> shift
        value |= part
        i += 1
    return value


def left(result, shift, mask):
    i, value = 0, 0
    while i * shift < 32:
        part_mask = ((kMaxBits >> (32 - shift) & kMaxBits)) << (i * shift)
        part = result & part_mask
        result ^= (part << shift) & mask
        value |= part
        i += 1
    return value


def convert(m):
    m ^= (m >> 11)
    m ^= (m << 7) & 0x9d2c5680
    m ^= (m << 15) & 0xefc60000
    m ^= (m >> 18)
    return m


def _convert(m):
    m = right(m, 18)
    m = left(m, 15, 0xefc60000)
    m = left(m, 7, 0x9d2c5680)
    m = right(m, 11)
    return m


def backtrace(state):
    for i in range(623, -1, -1):
        res = 0
        tmp = state[i]
        tmp ^= state[(i + 397) % 624]
        if (tmp & 0x80000000) == 0x80000000:
            tmp ^= 0x9908b0df
        res = (tmp << 1) & 0x80000000
        tmp = state[(i - 1 + 624) % 624]
        tmp ^= state[(i - 1 + 397) % 624]
        if (tmp & 0x80000000) == 0x80000000:
            tmp ^= 0x9908b0df
            res |= 1
        res |= (tmp << 1) & 0x7fffffff
        state[i] = res
    return state


def main():
    f = open('output.txt', 'r')
    data=f.read()
    xor_out = data.split("\n")[:-1]
    f.close()
    for i in range(len(xor_out) - 1, 0, -1):
        xor_out[i - 1] = int(xor_out[i]) ^ int(xor_out[i - 1])
    state = [_convert(int(i)) for i in xor_out]
    for _ in range(3):
        state = backtrace(state)
    value = [convert(i) for i in state]
    flag = value[-(1500 - 624 * 2) - 4:-(1500 - 624 * 2)]
    flag = "HECTF{" + "".join([hex(i)[2:] for i in flag]) + "}"
    print(flag)


if __name__ == '__main__':
    main()
```

得到flag: HECTF{479fdd0122bf0912ca6ba2c9ef25cff9}

