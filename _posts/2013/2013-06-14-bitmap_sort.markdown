---
author: wlxwolves
comments: true
date: 2013-06-14 08:09:22+00:00
layout: post
slug: bitmap_sort
title: 位图排序
wordpress_id: 114
categories:
- programming
---

# 1. 题目：


_输入：一个最多包含N个正整数的文件，每个数都小于N，N=10000000，这些数没有重复。_

_输出：按升序排列的数据。_

_要求：内存空间约1M，磁盘空间无限。_


# 2. 数据来源：


自己编写脚本产生随机数，由于最近在写PHP，就PHP吧，代码如下，没什么说的：<!-- more -->

{% highlight php %}
    <?php
    $num = 800000;
    $data = array();
    for($i = 0; $i < $num; $i ++)
    {
       $data[]  = rand(0,10000000);
    }
    $data = array_unique($data);
    $path = "data.txt";
    $file = fopen($path,"a");
    if(NULL == $file)
    {
        echo "open file failed";
        exit -1;
    }
    foreach($data as $v)
    {
       fputs($file,$v . "\n");
    }
    fclose($file);
    ?>
{% endhighlight %}

产生的数据保存在data.txt中，改变$num可以改变产生的随机数数量。需要的可以自己运行脚本产生数据进行测试，没有PHP环境的可以到[这里](http://pan.baidu.com/share/link?shareid=1848260083&uk=657450545)下载测试数据。






# 3. 解答




## 3.1 理想状态


首先，如果不缺内存，可以怎么弄？

最简单的，从data.txt中读取数据，将数据存到全局变量data数组中，然后，利用快排进行排序即可。

下面的程序从文件中将数据读到数组中。



{% highlight cpp %}
    #define NUM 10000000
    long int data[NUM] = {0};
    long int getData(char *file)
    {
        char ch[10] = {0};
        FILE *f = fopen(file,"r");
        if( NULL == f )
        {
            printf("open file failed\n");
            return -1;
        }
        int i = 0;
        while( fgets(ch,10,f) && i < NUM)
        {
            if(*ch == EOF)
            {
                printf("fgets failed\n");
                break;
            }
           data[i++] = atol(ch);
        }
        fclose(f);
        return i;
    }
{% endhighlight %}

在C语言中，快排用qsort即可，于是，很容易写下如下程序


{% highlight cpp %}
    #include <time.h>
    //asc
    int comp(const void *p, const void *q)
    {
        return ( *(int *)p - *(int *)q );
    }
    int main()
    {
        char *file = "data.txt";
    
        clock_t start, finish;//为了统计运行时间
        start = clock();
    
        long int len = getData(file);
        qsort(data, len, sizeof(long int), comp);
    
        finish = clock();
        double duration = (double)(finish - start) / CLOCKS_PER_SEC;
        printf("%lfs\n",duration);//显示运行时间
    
        //保存结果，主要是为了测试
        FILE *f = fopen ("ans.txt","w");
        long int i = 0;
        for(i = 0; i < len; i ++)
        {
            fprintf(f,"%ld\n",data[i]);
        }
        fclose(f);
        return 0;
    }
{% endhighlight %}


上面是理想状态，但是


## 3.2 残酷现实


我们的内存并不总是足够的，特别是当排序是某个大系统的很小的一部分的时候，更是受到各方面的限制，不仅仅是内存，很可能对时间要求也会更高，这里采用快排，时间复杂度达到N*logN，那么，有没有更好的方法呢？

注意，这里的数没有重复，那么，很自然的想到了位向量/位图。

定义一个保存结果的int型数组result[N]，每位对应一个整数，如果该整数存在，那么，将result数组中该位设为1，否则，设为0，最后，顺序扫描一遍结果数组result，顺序取出为1的位对应的整数，即得到排序后的结果。现在，剩下的问题就变成了待排序的整数怎么跟结果result数组中的位一一对应起来的问题了。

带排序数组data[NUM]，位向量result[N]。result数组为int型，在此，假设每个int型变量占4个字节，即32位，那么，对于data[NUM]中任意一个数data[i]，其对应位一定存在与result[i/32]这个数据中，对应于是result[i]中第 i%32 位。

于是，很容易写下操作位向量的相关代码：

{% highlight cpp %}
    #define WORD 32
    
    //set the bit
    void set(long int i)
    {
        result[i/WORD] |= ( 1 << (i%WORD) );
    }
    //clear the bit
    void clear(long int i)
    {
        result[i>/WORD] &= ~( 1 << (i%WORD) );
    }
    //test the bit
    int test(long int i)
    {
        return result[i/WORD] & ( 1 << (i%WORD) );
    }
{% endhighlight %}


可是，大多数资料上的代码是这样的：

    
{% highlight cpp %}
    #define SHIFT 5
    
    #define MASK 0x1F
    
    //set the bit
    void set(long int i)
    {
        result[i>>SHIFT] |= ( 1 << (i&MASK) );
    }
    //clear the bit
    void clear(long int i)
    {
        result[i>>SHIFT] &= ~( 1 << (i&MASK) );
    }
    //test the bit
    int test(long int i)
    {
        return result[i>>SHIFT] & ( 1 << (i&MASK) );
    }
{% endhighlight %}


其实，上述两段代码功能是一样的，i>>SHIFT，表示位运算i右移5位，25 = 32，表示i/32，举个例子，i=111，二进制表示为1101111，右移5位后，变成11，对应十进制3，刚好等于111/32。

同理，i & MASK，与i%32的功能一样。MASK的二进制表示位011111(前面的0省略了)，i & MASK相当于取i的最后5位，前面的都置0，也就表示i % 32.

那么，既然功能一样，为什么用第二种方法，即位运算，而不是第一种方法，直接用除法和取模呢？

从查到的资料来看，这涉及到效率问题了，**在现代架构中，位运算的速度比乘除法快很多，跟加减法差不多**。但是，我在这里对两种方法进行了测试，发现二者所需时间基本一致，测试方法如下面的程序所示。即使将

    
{% highlight cpp %}
    finish = clock();
{% endhighlight %}


放到最后也一样。


这是为什么呢？**原因是现代的编译器对乘除法进行了优化**！


最后，利用位运算进行排序，并测试结果，代码如下：

{% highlight cpp %}
    #include <time.h>
    int main()
    {
        char *file = "data.txt";
        char ch[10] = {0};
        clock_t start, finish;
        start = clock();
        FILE *f = fopen(file,"r");
        if( NULL == f )
        {
            printf("open file failed\n");
            return -1;
        }
    
        long int tempdata = 0;
        long int len = 0;
    
        //读取数据，设置相应位的值
        while( fgets(ch,10,f) && len < NUM)
        {
            if(*ch == EOF)
            {
                printf("fgets failed\n");
                break;
            }
            tempdata = atol(ch);
            len ++; 
            set(tempdata);
        }
        fclose(f);
    
        //统计运行时间
        finish = clock();
        double duration = (double)(finish - start) / CLOCKS_PER_SEC;
        printf("%lfs\n",duration);
    
        //保存结果以便测试
        FILE *fout = fopen("ans_new.txt","w");
        if( NULL == fout )
        {
            printf("fopen fail when save the result\n");
            return -1;
        }
        long int i = 0;
        for(i = 0; i < NUM; i ++)
        {
            if( test(i) )
            {
                fprintf(fout,"%ld\n",i);
            }
        }
        fclose(fout);
    
        return 0;
    }
{% endhighlight %}

总结一下发现，采用位向量进行排序将时间复杂度降到了O(N)，同时，占用的空间也大大的减少了。

利用qsort和位向量两种方法时，分别进行了时间统计，如程序中所示。其中，测试数据中有9996663条数据。



	
  * qsort：3.78s

  * 位向量:1.25s




# 4. 扩展




## 4.1 适用条件


_要用位向量进行排序有一定的条件：_



	
  1. _待排序数据需要限制在一个较小的范围内。_
  2. _数据没有重复。 _


_那么，条件能不能放宽呢？_

对于第一条，应该是不行的（如果有，还望不吝赐教啊）。

对第二条，如果输入的数据是有重复的，每个数据至多出现K次，是否仍然能用位向量进行排序呢？

答案是肯定的，例如K=13，23 < K < 24, 所以，那么，可以用result中的4位对应一个整数，也就是说，修改3个位向量函数即可。


## 4.2 内存限制


_文中的算法需要占用约1.25M内存，那么，如果严格要求1M内存，该怎么办？_

这样，可以借助外排思想，分两次处理。

顺序读取输入数据，如果是[0,5000000)，则处理，设置result相应位，否则，读取下一个数据。

保存[0,5000000)内的结果。

再次顺序读取输入数据，如果是[5000000，10000000)，则处理，设置result相应位，否则，读取下一个数据。

保存[5000000，10000000)内的结果追加到[0,5000000)内的结果后面(因为结果也是有序的，所以，直接追加即可)。


# 5. 遗留问题


位操作在对于提高程序性能有着重要的作用，还有哪些场景会用到？

最近研究下，下篇blog可以探讨。




