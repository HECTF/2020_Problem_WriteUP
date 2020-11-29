

已知明文前九位 猜测是三阶的希尔密码

对应密文的前九位，求出密钥矩阵

K* $\begin{bmatrix}
7  \\
4  \\
2  
\end{bmatrix}$$\equiv $$\begin{bmatrix}
2  \\
3  \\
16  
\end{bmatrix}$(mod 26)

K*$\begin{bmatrix}
19  \\
5  \\
4  
\end{bmatrix}$$\equiv$$\begin{bmatrix}
18  \\
17  \\
15 
\end{bmatrix}$(mod 26)

K*$\begin{bmatrix}
25  \\
19  \\
7  
\end{bmatrix}$$\equiv$$\begin{bmatrix}
15  \\
22  \\
20  
\end{bmatrix}$(mod 26) 

故得出

K*$\begin{bmatrix}
7 & 19 &25 \\
4 & 5 & 19\\
2 & 4 & 7 
\end{bmatrix}$$\equiv$$\begin{bmatrix}
2 & 18 &15 \\
3 & 17 & 22\\
16 & 15 & 20 
\end{bmatrix}$(mod 26)

K$\equiv$$\begin{bmatrix}
2 & 18 &15 \\
3 & 17 & 22\\
16 & 15 & 20 
\end{bmatrix}$$\begin{bmatrix}
7 & 19 &25 \\
4 & 5 & 19\\
2 & 4 & 7 
\end{bmatrix}^{-1}$(mod 26)

求出Key矩阵

明文矩阵$\equiv$K$^{-1}$*密文矩阵(mod 26)

即可得到flag

```python
import string, gmpy2
from numpy import *
tab = string.ascii_lowercase


def getmat(data):
    array = [[], [], []]
    for i in range(len(data)):
        ind = tab.find(data[i])
        array[i % 3].append(ind)
    datamat = mat(array)
    return datamat


def getkey(ciphermat, plainmat):
    d = int(round(linalg.det(plainmat)))
    keymat = mod(
        ciphermat *
        multiply(linalg.inv(plainmat), d * gmpy2.invert(d, 26)).astype(int32),
        26)
    return keymat


def decode(ciphermat, keymat):
    d = int(round(linalg.det(keymat)))
    flagmat = mod(
        multiply(linalg.inv(keymat), d * gmpy2.invert(d, 26)).astype(int32) *
        ciphermat, 26)
    flagarray = flagmat.getA()
    flag = ''
    for i in range(len(cipher) // 3):
        for j in range(3):
            flag += tab[flagarray[j][i]]
    return flag


if __name__ == "__main__":
    p_plain = 'hectfezth'
    cipher = 'cdqsrppwuzzyjhwrhyzripfpxol'
    p_cipher = cipher[:9]
    p_plainmat = getmat(p_plain)
    p_ciphermat = getmat(p_cipher)
    keymat = getkey(p_ciphermat, p_plainmat)
    ciphermat = getmat(cipher)
    print(decode(ciphermat, keymat))
#hectfezthirdorderhillcipher
```

同样可以使用sagemath求解

```sage
part_cipher=matrix(Zmod(26),3,3)
part_cipher[0]=[2,18,15]
part_cipher[1]=[3,17,22]
part_cipher[2]=[16,15,20]

part_plain=matrix(Zmod(26),3,3)
part_plain[0]=[7,19,25]
part_plain[1]=[4,5,19]
part_plain[2]=[2,4,7]

k=part_plain.solve_left(part_cipher)

cipher=matrix(Zmod(26),3,9)
cipher[0]=[2,18,15,25,9,17,25,15,23]
cipher[1]=[3,17,22,25,7,7,17,5,14]
cipher[2]=[16,15,20,24,22,24,8,15,11]

flag=k.solve_right(cipher)
print(flag)
```



flag:HECTF{ezthirdorderhillcipher}

