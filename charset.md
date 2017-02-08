# 字符集浅析

>   字符集在开发过程中经常遇到，不过一般情况下我们都对其知其然，不知其所以然；本文稍微对我们常见的两种字符集（GBK，UTF-8）做一下简单的解析。

###判断字符集

本文给出一种比较笨拙的了解是哪一种字符集的方法，这种方法只做对字符集的了解使用，估计很难在真实中使用。:)

* 1 例如任意选择一个汉字"请"，解析出其字节数组。如果是两个字节的话，优先考虑可能是GBK编码。
* 2 将字节 放入计算器内，看其二进制的表现形式。

* 3 GBK 一般都是 中文的情况，此时可以在 [GBK编码](http://www.qqxiuzi.cn/zh/hanzi-gbk-bianma.php) 下查找是有对应的汉子，查找方式如下，
例如：
    第一个字节 1100 0001 
    第二个字节 1111 0101
    第一个字节 组合起来是 C1， 第二个字节组合起来是F5，则查找 C1（F,5）矩阵，如果能查到 对应的汉字，则必然是 GBK编码

* 4 猜测其是UTF-8编码的话，首先将原来的字符查看对应的ASCII码，对比下表格：

    | Unicode/UCS-4  | bit数 | UTF-8  | byte数 |  备注  |
    | ------------- |:-------------:| -----:| -----  | -----  |
    | 0000 ~ 007F     | 0~7 | 0XXX XXXX | 1  |   |
    | 0080 ~ 07FF     | 8~11|   110X XXXX 10XX XXXX | 2  |    |
    | 0800 ~ FFFF | 12~16     |    1110XXXX 10XX XXXX 10XX XXXX | 3  | 基本定义范围：0~FFFF |
    | 1 0000 ~1F FFFF    | 17~21|  1111 0XXX 10XX XXXX 10XX XXXX 10XX XXXX | 4  |  Unicode6.1定义范围：0~10 FFFF  |
    | 20 0000 ~ 3FF FFFF   | 22~26 |  1111 10XX 10XX XXXX 10XX XXXX 10XX XXXX 10XX XXXX| 5  |  |
    | 400 0000 ~ 7FFF FFFF   | 27~31 |  1111 110X 10XX XXXX 10XX XXXX 10XX XXXX 10XX XXXX 10XX XXXX | 6  |  |
    
    表格来自[百度百科](http://baike.baidu.com/link?url=VqAi7VkK41uZqyclkiQmtjhGTKgXsCTLbCI5OE-NzYGanP9CFEoFNPjziKd3kN3H8-L2t950stxvcwd3xLwLZ-YjYyblWAI0TCb7XrQE6St80h39hmujBk3lcNuK7FUp)

    表格中所属的几种情况中的任意一种的话，则可以判断 是UTF-8 编码.




