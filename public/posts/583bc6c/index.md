# 多线程基础


# 多线程基础

## 并发与并行

### 并发执行

![bingfa](https://minio.dionysunrsshub.top:443/myimages/2024-img/bingfa.png)
同一时间只能处理一个任务，每个任务轮着做（时间片轮转），只要我们单次处理分配的时间足够的短，在宏观看来，就是三个任务在同时进行。
而我们Java中的线程，正是这种机制，当我们需要同时处理上百个上千个任务时，很明显CPU的数量是不可能赶得上我们的线程数的，所以说这时就要求我们的程序有良好的并发性能，来应对同一时间大量的任务处理

&lt;!--more--&gt;


---

> Author: Shiping Guo  
> URL: http://localhost:1313/posts/583bc6c/  

