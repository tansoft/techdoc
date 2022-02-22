## -E 匹配多个需要的行输出

```bash
cat /proc/cpuinfo | grep -E 'processor|model name|cpu MHz'

processor	: 0
model name	: Intel(R) Xeon(R) Platinum 8259CL CPU @ 2.50GHz
cpu MHz		: 2499.998
processor	: 1
model name	: Intel(R) Xeon(R) Platinum 8259CL CPU @ 2.50GHz
cpu MHz		: 2499.998
```

## -A -B -C 显示匹配行的前后行

```bash
cat txt.txt
line1
line2
line3
line4
line5

#匹配内容和后一行
grep 'line3' -A 1
line3
line4

#匹配内容和前一行
grep 'line3' -B 1
line2
line3

#匹配内容和前后一行
grep 'line3' -C 1
line2
line3
line4


```