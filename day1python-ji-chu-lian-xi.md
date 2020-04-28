# Day1-Python基础练习

**作业一：输出 9\*9 乘法口诀表\(注意格式\)**

```python
 1*1=1   
 2*1=2   2*2=4   
 3*1=3   3*2=6   3*3=9   
 4*1=4   4*2=8   4*3=12  4*4=16  
 5*1=5   5*2=10  5*3=15  5*4=20  5*5=25  
 6*1=6   6*2=12  6*3=18  6*4=24  6*5=30  6*6=36  
 7*1=7   7*2=14  7*3=21  7*4=28  7*5=35  7*6=42  7*7=49  
 8*1=8   8*2=16  8*3=24  8*4=32  8*5=40  8*6=48  8*7=56  8*8=64  
 9*1=9   9*2=18  9*3=27  9*4=36  9*5=45  9*6=54  9*7=63  9*8=72  9*9=81 
```

```python
def table():
    #在这里写下您的乘法口诀表代码吧！
    print ('\n'.join('\t'.join(f'{i}*{j}={i*j}' for i in range(1,j+1)) for j in range(1,10)))
    
if __name__ == '__main__':
    table()
```

一般的方案用两层for循环也可以实现，但是也可以一行实现：

使用列表生成式进行嵌套生成99乘法表，关键在于两层for之间用了一个\(\)括号相隔开，使用.join\(\)使得生成器generator返回值，接着用`f'{}'`大括号作为`i,j`值的占位符。假如没有使用.join\(\)，就没有那么简洁了：

```python
def table():
    #在这里写下您的乘法口诀表代码吧！
    print (*("" if row==0 else "\n" if col>=row+1 else f"{row}*{col}={row*col:<3}" for row in range(0, 10) for col in range(1, row+2)))

if __name__ == '__main__':
    table()
```

相比之下这个版本的可读性明显更差，`"" if row==0`在这里是为了让第一行不输出多余的空格，而`range(1, row+2)`多出的1个使用`if else`用`\n`进行替换达到换行的目的。

**作业二：查找特定名称文件**

遍历”Day1-homework”目录下文件；

找到文件名包含“2020”的文件；

将文件名保存到数组result中；

按照序号、文件名分行打印输出。

```python
[1,'Day1-homework/26/26/new2020.txt'] 
[2,'Day1-homework/18/182020.doc'] 
[3,'Day1-homework/4/22/04:22:2020.txt']
```

```python
#导入OS模块
import os
#待搜索的目录路径
path = "Day1-homework"
#待搜索的名称
filename = "2020"
#定义保存结果的数组
result = []

def findfiles():
    #在这里写下您的查找文件代码吧！
    result=[files[0]+'/'+file for files in os.walk(path) for file in files[2] if filename in file]
    print(*(f'[{index+1},\'{name}\']' if index==0 else f'\n[{index+1},\'{name}\']' for index,name in enumerate(result)))

    

if __name__ == '__main__':
    findfiles()
```

这里也是故技重施，使用列表生成式利用os.walk\(\)函数生成结果的list，然后构造答案的输出格式进行输出。注意此处for和if的先后关系，上式也可以写成正常的循环形式：

```python
for files in os.walk(path):
    for file in files[2]:
        if filename in file:
            result.append(files[0]+'/'+file)
```

