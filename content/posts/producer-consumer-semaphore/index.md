---
title: 生产者消费者模型与信号量

date: 2020-09-26T07:35:26+08:00

tag: [ 操作系统, 线程, 信号量]

categories: 操作系统
description: 生产者消费者模型与信号量
---



最近在看操作系统有关线程同步的章节,关于 `信号量` , `条件变量` , `互斥量` 之间的具体区别有点分不清楚,所以在这里记录一下



## 信号量 , 条件变量, 互斥量

首先信号量,条件变量,互斥量都是有关操作系统线程同步的有关知识,我相信在每一本操作系统教材上都有提到过,下面我写一下我自己理解

+ `互斥量` : 互斥量总是和互斥锁相联系,我们所说的互斥锁本质上就是锁住一个互斥量,所以互斥量只有两种情况
  + 被上锁状态
  + 解锁状态
  + 所以就可以用`0` , `-1`  表示 , `0` 表示自由, 没有被上锁, `-1` 表示已经被上锁
+ `信号量`:  信号量(Semaphore)，有时被称为信号灯，是在多线程环境下使用的一种设施，是可以用来保证两个或多个关键代码段不被[并发](https://baike.baidu.com/item/并发/11024806)调用。在进入一个关键代码段之前，线程必须获取一个信号量；一旦该关键代码段完成了，那么该线程必须释放信号量。
  + 按我的理解,信号量就是用来表示某个资源可以被其他线程同时获取的数量, 比如信号量为 5,可以允许对做5个线程对资源进行获取,当然每个线程获取之前,必须将信号量减一, 而每个线程释放之后,有必须将信号量加一,以供其他资源获取.
  + 当信号量初始化最大为1时,此时只允许同时有一个线程对资源进行操作, 那么也就是互斥了,那么这就形成了一把互斥锁,所以 `二值信号量` 我们可以当做一把锁来用 . 
+ `条件变量` : 条件变量 ( cond ) 是在多线程程序中用来实现"等待 --> 唤醒" 逻辑常用的方法。条件变量利用线程间共享的全局变量进行同步的一种机制，主要包括两个动作：一个线程等待"条件变量的条件成立"而挂起；另一个线程使“条件成立”。为了防止竞争，条件变量的使用总是和一个互斥锁结合在一起。线程在改变条件状态前必须首先锁住互斥量，函数 pthread_cond_wait 把自己放到等待条件的线程列表上，然后对互斥锁解锁(这两个操作是原子操作)。在函数返回时，互斥量再次被锁住。
  + 众所周知,如果要防止多个线程对资源进行竞争, 最简单粗暴的方法就是对该资源进行上锁,但是用个问题,一旦 某个线程想要获取某个上锁的线程,它只能 `等待` -> `尝试获取` -> `失败` -> `尝试获取` -> `等待` ,如此的循环自旋是比较消耗 CPU 资源的,所以就需要条件变量.
  + 条件变量的作用就是让获取不过锁的线程休眠,然后在释放 锁/条件变量 的时候唤醒休眠的线程.
  + 所以条件变量 = 上锁/解锁 + 休眠/唤醒 线程的组合操作 , 它把这两个操作放在一起,构成了一个原子操作.
+ **互斥量是信号量的一种特例，互斥量的本质是一把锁。A mutex is basically a lock that we set (lock) before accessing a shared resource and release (unlock) when we're done**



## 基于信号量的生产者消费者模型



```c
#include <string.h>
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <stdlib.h>
#include <time.h>

sem_t empty,full;

pthread_mutex_t mutex;

int buffer_count = 0; //全局变量,管道内 产品数目大小

void *producer (void *arg); //定义生产者

void *consumer (void *arg); // 定义消费者


int main(int argc, char **argv) {
    srand((unsigned)time(NULL));  //初始化随机种子
    pthread_t p_prod; // 生产者线程
    pthread_t p_cons; // 消费者线程
    pthread_mutex_init(&mutex, NULL); //  初始化互斥量
    sem_init(&full,0, 0);  //初始化 full 信号量
    sem_init(&empty, 0 ,5 ); // 初始化 empty empty 信号量
    // 创建生产者和消费者线程
    if (pthread_create(&p_prod, NULL, producer, NULL) != 0) 
    {
        printf("thread producter create failed. \n");
    }
    if (pthread_create(&p_cons, NULL, consumer, NULL) != 0) 
    {
        printf("thread consumer create failed. \n");       
    }

    pthread_join(p_cons, NULL);
    pthread_join(p_prod, NULL);
    return 0;
}

void *producer(void *arg) {
    while (1) {
        sem_wait(&empty); // wait 使 empty 信号量 减1 ,当empty 减为 -1 时,线程休眠
        pthread_mutex_lock(&mutex);
        printf("producer put an element..");
        buffer_count++;
        printf("now buffer_count is %d\n", buffer_count);
        pthread_mutex_unlock(&mutex);
        sem_post(&full); // post 使 full 信号量加1 唤醒消费者可以消费
        sleep(rand() % 3 ); //用于三秒内随机控制时间
    }
}


void *consumer (void *arg) {
    while (1) {
        sem_wait(&full); // wait 使 full 信号量 减一 ,为-1 时 说明 负的full 就是 empty 所以休眠
        pthread_mutex_lock(&mutex);
        printf("consumer get an element..");
        buffer_count--;
        printf("now buffer_count is %d\n", buffer_count);
        pthread_mutex_unlock(&mutex);
        sem_post(&empty); // post 使 empty 信号量 加1 ,生产者可以生产
        sleep(rand() % 3);//用于三秒内随机控制时间
    }
}
```

执行结果:

```shell
consumer get an element..now buffer_count is 0
producer put an element..now buffer_count is 1
producer put an element..now buffer_count is 2
consumer get an element..now buffer_count is 1
consumer get an element..now buffer_count is 0
producer put an element..now buffer_count is 1
consumer get an element..now buffer_count is 0
producer put an element..now buffer_count is 1
consumer get an element..now buffer_count is 0
producer put an element..now buffer_count is 1
producer put an element..now buffer_count is 2
producer put an element..now buffer_count is 3
```

