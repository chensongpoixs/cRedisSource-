# redis源码的学习



###    比特位定位  （指数哥伦布码）  

一种很奇妙定位到比特位上位操作的方法值到我们学习




 在计算机中，一般数字的编码都为二进制，但是由于以相等长度来记录不同数字，因此会出现很多的冗余信息，如下：

|十进制   | 5        |  4       | 255     |2        |   1       |
|:--:|:--:|:--:|:--:|:--:|:--:|
|二进制   | 00000101 | 00000100 |11111111 |00000010 |00000001   |
|有效字节 | 3        |3         |  8      |   2     |    1      |

如数字1，原本只需要1个bit就能表示的数据，如今需要8个bit来表示，那么其余7个bit就可以看做是冗余数据，在网络传输时，如果以原本等长的编码方式来传输数据，则会出现很大的冗余量，加重网络负担但是如果只用有效字节来传输上述码流，则会是：10110011111111101，这样根本不能分离出原本的数据。哥伦布编码则是作为一种压缩编码算法，能很有效地对原本的数据进行压缩，并且能很容易地把编码后的码流分离成码字。


用来表示非负整数的k阶指数哥伦布码可用如下步骤生成：

1. 将数字以二进制形式写出，去掉最低的k个比特，之后加1
2. 计算留下的比特数，将此数减一，即是需要增加的前导零个数
3. 将第一步中去掉的最低k个比特位补回比特串尾部

```
/* Get current values */
    // 这里有一个定义就是  什么要 >> 是3 ->>> 它是一个字节8个比特位, 一个数组向右移动三个比特位就 大约定位到8个比特位上了， 再使用低三位定位到8个比特位中具体的的比特位上是不是好完美啊
    byte = bitoffset >> 3;     //  例子: 53   --> 0011 0101 => 0000 0011
    byteval = ((uint8_t*)o->ptr)[byte];
    bit = 7 - (bitoffset & 0x7); // 底的3位拿到 -> 
    bitval = byteval & (1 << bit); // 0001 ====> 1000, 0100, 0010

    /* Update byte with new bit value and return original value */
    byteval &= ~(1 << bit);
    byteval |= ((on & 0x1) << bit);
    ((uint8_t*)o->ptr)[byte] = byteval;

```



###  计算字符串中"1"的个数

1. 查表法
2. 汉明重量(28byte) hamming weight算法

```
/* Count number of bits set in the binary array pointed by 's' and long
 * 'count' bytes. The implementation of this function is required to
 * work with a input string length up to 512 MB. */
size_t redisPopcount(void *s, long count) {
    size_t bits = 0;
    unsigned char *p = s;
    uint32_t *p4;
    /**
     * 这边8个比特位对应"1"个数罗列出来了  char = > 256   使用空间换时间的做法
     */
    static const unsigned char bitsinbyte[256] = {0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,4,5,5,6,5,6,6,7,5,6,6,7,6,7,7,8};

    /* Count initial bytes not aligned to 32 bit. */
    while((unsigned long)p & 3 && count) {
        bits += bitsinbyte[*p++];
        count--;
    }

    /* Count bits 28 bytes at a time */
    p4 = (uint32_t*)p;
    while(count>=28) {
        uint32_t aux1, aux2, aux3, aux4, aux5, aux6, aux7;

        aux1 = *p4++;
        aux2 = *p4++;
        aux3 = *p4++;
        aux4 = *p4++;
        aux5 = *p4++;
        aux6 = *p4++;
        aux7 = *p4++;
        count -= 28;

        aux1 = aux1 - ((aux1 >> 1) & 0x55555555);
        aux1 = (aux1 & 0x33333333) + ((aux1 >> 2) & 0x33333333);
        aux2 = aux2 - ((aux2 >> 1) & 0x55555555);
        aux2 = (aux2 & 0x33333333) + ((aux2 >> 2) & 0x33333333);
        aux3 = aux3 - ((aux3 >> 1) & 0x55555555);
        aux3 = (aux3 & 0x33333333) + ((aux3 >> 2) & 0x33333333);
        aux4 = aux4 - ((aux4 >> 1) & 0x55555555);
        aux4 = (aux4 & 0x33333333) + ((aux4 >> 2) & 0x33333333);
        aux5 = aux5 - ((aux5 >> 1) & 0x55555555);
        aux5 = (aux5 & 0x33333333) + ((aux5 >> 2) & 0x33333333);
        aux6 = aux6 - ((aux6 >> 1) & 0x55555555);
        aux6 = (aux6 & 0x33333333) + ((aux6 >> 2) & 0x33333333);
        aux7 = aux7 - ((aux7 >> 1) & 0x55555555);
        aux7 = (aux7 & 0x33333333) + ((aux7 >> 2) & 0x33333333);
        bits += ((((aux1 + (aux1 >> 4)) & 0x0F0F0F0F) +
                    ((aux2 + (aux2 >> 4)) & 0x0F0F0F0F) +
                    ((aux3 + (aux3 >> 4)) & 0x0F0F0F0F) +
                    ((aux4 + (aux4 >> 4)) & 0x0F0F0F0F) +
                    ((aux5 + (aux5 >> 4)) & 0x0F0F0F0F) +
                    ((aux6 + (aux6 >> 4)) & 0x0F0F0F0F) +
                    ((aux7 + (aux7 >> 4)) & 0x0F0F0F0F))* 0x01010101) >> 24;
    }
    /* Count the remaining bytes. */
    p = (unsigned char*)p4;
    while(count--) 
    {
        bits += bitsinbyte[*p++];
    }
    return bits;
}


```



