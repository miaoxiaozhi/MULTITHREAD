互斥锁(mutex)
Linux中提供一把互斥锁mutex（也称之为互斥量）。

每个线程在对资源操作前都尝试先加锁，成功加锁才能操作，操作结束解锁。

但通过“锁”就将资源的访问变成互斥操作，而后与时间有关的错误也不会再产生了。



 

但，应注意：同一时刻，只能有一个线程持有该锁。

当A线程对某个全局变量加锁访问，B在访问前尝试加锁，拿不到锁，B阻塞。C线程不去加锁，而直接访问该全局变量，依然能够访问，但会出现数据混乱。

所以，互斥锁实质上是操作系统提供的一把“建议锁”（又称“协同锁”），建议程序中有多线程访问共享资源的时候使用该机制。但，并没有强制限定。
因此，即使有了mutex，如果有线程不按规则来访问数据，依然会造成数据混乱。

1、主要应用函数：
pthread_mutex_init()函数          功能：初始化一个互斥锁

pthread_mutex_destroy()函数   功能：销毁一个互斥锁

pthread_mutex_lock()函数        功能：加锁

pthread_mutex_trylock()函数    功能：尝试加锁

pthread_mutex_unlock()函数    功能：解锁

以上5个函数的返回值都是：成功返回0， 失败返回错误号。

pthread_mutex_t 类型，其本质是一个结构体。为简化理解，应用时可忽略其实现细节，简单当成整数看待。如：

pthread_mutex_t   mutex; 变量mutex只有两种取值1、0。

2、函数分析 
<1>、初始化一个互斥锁(互斥量) ---> 初值可看作1 

int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr); 

参1：传出参数，调用时应传 &mutex

参2：互斥量属性。是一个传入参数，通常传NULL，选用默认属性(线程间共享)。

注意：互斥锁初始化有两种方式：

      【1】、静态初始化：如果互斥锁 mutex 是静态分配的（定义在全局，或加了static关键字修饰），可以直接使用宏进行初始化。e.g.  pthead_mutex_t   muetx = PTHREAD_MUTEX_INITIALIZER;

        【2】、动态初始化：局部变量应采用动态初始化。e.g.  pthread_mutex_init(&mutex, NULL)

 <2>、加锁。可理解为将mutex--（或-1）

 int pthread_mutex_lock(pthread_mutex_t *mutex);

<3>、尝试加锁

int pthread_mutex_trylock(pthread_mutex_t *mutex);

 <4>、解锁。可理解为将mutex ++（或+1）

 int pthread_mutex_unlock(pthread_mutex_t *mutex);

  <5>、销毁一个互斥锁

int pthread_mutex_destroy(pthread_mutex_t *mutex);

3、加锁与解锁
lock与unlock：

lock尝试加锁，如果加锁不成功，线程阻塞，阻塞到持有该互斥量的其他线程解锁为止。

unlock主动解锁函数，同时将阻塞在该锁上的所有线程全部唤醒，至于哪个线程先被唤醒，取决于优先级、调度。默认：先阻塞、先唤醒。

例如：T1 T2 T3 T4 使用一把mutex锁。T1加锁成功，其他线程均阻塞，直至T1解锁。T1解锁后，T2 T3 T4均被唤醒，并自动再次尝试加锁。

可假想mutex锁 init成功初值为1。 lock 功能是将mutex--。 unlock将mutex++

lock与trylock：

lock加锁失败会阻塞，等待锁释放。

trylock加锁失败直接返回错误号（如：EBUSY），不阻塞。

4、加锁步骤测试：
看如下程序：该程序是非常典型的，由于共享、竞争而没有加任何同步机制，导致产生于时间有关的错误，造成数据混乱：

程序1：主线程和子线程打印出来的hello world 被分开输出了

#include <stdio.h>
#include <string.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
 
 
 
void *tfn(void *arg)
{
 
    while (1) {
      
 
        printf("hello ");
        sleep(1);	/*模拟长时间操作共享资源，导致cpu易主，产生与时间有关的错误*/
        printf("world\n");
        sleep(1);//睡眠，释放cpu
    }
 
    return NULL;
}
 
int main(void)
{
   
    pthread_t tid;
    
  
 
    pthread_create(&tid, NULL, tfn, NULL);
    while (1) 
	{
  
        printf("HELLO ");
        sleep(1);
        printf("WORLD\n");      
        sleep(1);
    }
   
 
 
    return 0;
}
 
/*线程之间共享资源stdout*/
程序2：利用互斥锁解决程序1的bug

#include <stdio.h>
#include <string.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
 
pthread_mutex_t mutex;      //定义锁
 
void *tfn(void *arg)
{
 
    while (1) {
        pthread_mutex_lock(&mutex);  //加锁
 
        printf("hello ");
        sleep(1);	/*模拟长时间操作共享资源，导致cpu易主，产生与时间有关的错误*/
        printf("world\n");
        pthread_mutex_unlock(&mutex); //解锁
 
        sleep(1);//睡眠，释放cpu
    }
 
    return NULL;
}
 
int main(void)
{
   
    pthread_t tid;
    
  
    pthread_mutex_init(&mutex, NULL);  //初始化锁 mutex==1
    pthread_create(&tid, NULL, tfn, NULL);
    while (1) 
	{
 
        pthread_mutex_lock(&mutex); //加锁
 
        printf("HELLO ");
        sleep(1);
        printf("WORLD\n");
        pthread_mutex_unlock(&mutex); //解锁
 
        sleep(1);
 
    }
   
    pthread_mutex_destroy(&mutex);  //销毁锁
 
    return 0;
}
 
/*线程之间共享资源stdout*/
结论：

在访问共享资源前加锁，访问结束后立即解锁。锁的“粒度”应越小越好。
