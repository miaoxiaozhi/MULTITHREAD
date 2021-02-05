C++ 多线程
分类 编程技术
创建线程
在Windows平台，Windows API提供了对多线程的支持。前面进程和线程的概念中我们提到，一个程序至少有一个线程，这个线程称为主线程(main thread)，如果我们不显示地创建线程，那我们产的程序就是只有主线程的间线程程序。

下面，我们看看Windows中线程相关的操作和方法：

CreateThread 与 CloseHandle
CreateThread 用于创建一个线程，其函数原型如下：

HANDLE WINAPI CreateThread(
    LPSECURITY_ATTRIBUTES   lpThreadAttributes, //线程安全相关的属性，常置为NULL
    SIZE_T                  dwStackSize,        //新线程的初始化栈在大小，可设置为0
    LPTHREAD_START_ROUTINE  lpStartAddress,     //被线程执行的回调函数，也称为线程函数
    LPVOID                  lpParameter,        //传入线程函数的参数，不需传递参数时为NULL
    DWORD                   dwCreationFlags,    //控制线程创建的标志
    LPDWORD                 lpThreadId          //传出参数，用于获得线程ID，如果为NULL则不返回线程ID
);
说明：

lpThreadAttributes：指向SECURITY_ATTRIBUTES结构的指针，决定返回的句柄是否可被子进程继承，如果为NULL则表示返回的句柄不能被子进程继承。

dwStackSize ：线程栈的初始化大小，字节单位。系统分配这个值对
lpStartAddress：指向一个函数指针，该函数将被线程调用执行。因此该函数也被称为线程函数(ThreadProc)，是线程执行的起始地址，线程函数是一个回调函数，由操作系统在线程中调用。线程函数的原型如下：

DWORD WINAPI ThreadProc(LPVOID lpParameter);    //lpParameter是传入的参数，是一个空指针
lpParameter：传入线程函数(ThreadProc)的参数，不需传递参数时为NULL

dwCreationFlags：控制线程创建的标志，有三个类型，0：线程创建后立即执行线程；CREATE_SUSPENDED：线程创建后进入就绪状态，直到线程被唤醒时才调用；STACK_SIZE_PARAM_IS_A_RESERVATION：dwStackSize 参数指定线程初始化栈的大小，如果STACK_SIZE_PARAM_IS_A_RESERVATION标志未指定，dwStackSize将会设为系统预留的值。

返回值：如果线程创建成功，则返回这个新线程的句柄，否则返回NULL。如果线程创建失败，可通过GetLastError函数获得错误信息。

BOOL WINAPI CloseHandle(HANDLE hObject);        //关闭一个被打开的对象句柄 可用这个函数关闭创建的线程句柄，如果函数执行成功则返回true(非0),如果失败则返回false(0)，如果执行失败可调用GetLastError.函数获得错误信息。
【Demo1】：创建一个最简单的线程
实例
#include "stdafx.h"
#include <windows.h>
#include <iostream>
 
using namespace std;
 
//线程函数
DWORD WINAPI ThreadProc(LPVOID lpParameter)
{
    for (int i = 0; i < 5; ++ i)
    {
        cout << "子线程:i = " << i << endl;
        Sleep(100);
    }
    return 0L;
}
 
int main()
{
    //创建一个线程
    HANDLE thread = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
    //关闭线程
    CloseHandle(thread);
 
    //主线程的执行路径
    for (int i = 0; i < 5; ++ i)
    {
        cout << "主线程:i = " << i << endl;
        Sleep(100);
    }
 
    return 0;
}
结果如下：

主线程:i = 0 
子线程:i = 0 
主线程:i = 1 
子线程:i = 1 
子线程:i = 2 
主线程:i = 2 
子线程:i = 3 
主线程:i = 3 
子线程:i = 4 
主线程:i = 4
【Demo2】：在线程函数中传入参数
实例
#include "stdafx.h"
#include <windows.h>
#include <iostream>
 
using namespace std;
 
#define NAME_LINE   40
 
//定义线程函数传入参数的结构体
typedef struct __THREAD_DATA
{
    int nMaxNum;
    char strThreadName[NAME_LINE];
 
    __THREAD_DATA() : nMaxNum(0)
    {
        memset(strThreadName, 0, NAME_LINE * sizeof(char));
    }
}THREAD_DATA;
 
 
 
//线程函数
DWORD WINAPI ThreadProc(LPVOID lpParameter)
{
    THREAD_DATA* pThreadData = (THREAD_DATA*)lpParameter;
 
    for (int i = 0; i < pThreadData->nMaxNum; ++ i)
    {
        cout << pThreadData->strThreadName << " --- " << i << endl;
        Sleep(100);
    }
    return 0L;
}
 
int main()
{
    //初始化线程数据
    THREAD_DATA threadData1, threadData2;
    threadData1.nMaxNum = 5;
    strcpy(threadData1.strThreadName, "线程1");
    threadData2.nMaxNum = 10;
    strcpy(threadData2.strThreadName, "线程2");
 
//创建第一个子线程
    HANDLE hThread1 = CreateThread(NULL, 0, ThreadProc, &threadData1, 0, NULL);
    //创建第二个子线程
    HANDLE hThread2 = CreateThread(NULL, 0, ThreadProc, &threadData2, 0, NULL);
    //关闭线程
    CloseHandle(hThread1);
    CloseHandle(hThread2);
 
    //主线程的执行路径
    for (int i = 0; i < 5; ++ i)
    {
        cout << "主线程 === " << i << endl;
        Sleep(100);
    }
 
    system("pause");
    return 0;
}
结果：

主线程 === 线程1 — 0 
0 
线程2 — 0 
线程1 — 1 
主线程 === 1 
线程2 — 1 
主线程 === 2 
线程1 — 2 
线程2 — 2 
主线程 === 3 
线程2 — 3 
线程1 — 3 
主线程 === 4 
线程2 — 4 
线程1 — 4 
线程2 — 5 
请按任意键继续… 线程2 — 6 
线程2 — 7 
线程2 — 8 
线程2 — 9
CreateMutex、WaitForSingleObject、ReleaseMutex
从【Demo2】中可以看出，虽然创建的子线程都正常执行起来了，但输出的结果并不是我们预期的效果。我们预期的效果是每输出一条语句后自动换行，但结果却并非都是这样。这是因为在线程执行时没有做同步处理，比如第一行的输出，主线程输出"主线程 ==="后时间片已用完，这时轮到子线程1输出，在子线程输出"线程1 —"后时间片也用完了，这时又轮到主线程执行输出"0"，之后又轮到子线程1输出"0"。于是就出现了"主线程 === 线程1 — 0 0"的结果。

主线程：cout << "主线程 === " << i << endl; 
子线程：cout << pThreadData->strThreadName << " — " << i << endl;

为避免出现这种情况，我们对线程做一些简单的同步处理，这里我们用互斥量(Mutex)。

互斥量(Mutex)和二元信号量类似，资源仅允许一个线程访问。与二元信号量不同的是，信号量在整个系统中可以被任意线程获取和释放，也就是说，同一个信号量可以由一个线程获取而由另一线程释放。而互斥量则要求哪个线程获取了该互斥量锁就由哪个线程释放，其它线程越俎代庖释放互斥量是无效的。

在使用互斥量进行线程同步时会用到以下几个函数：

HANDLE WINAPI CreateMutex(
    LPSECURITY_ATTRIBUTES lpMutexAttributes,        //线程安全相关的属性，常置为NULL
    BOOL                  bInitialOwner,            //创建Mutex时的当前线程是否拥有Mutex的所有权
    LPCTSTR               lpName                    //Mutex的名称
);
说明： lpMutexAttributes也是表示安全的结构，与CreateThread中的lpThreadAttributes功能相同，表示决定返回的句柄是否可被子进程继承，如果为NULL则表示返回的句柄不能被子进程继承。bInitialOwner表示创建Mutex时的当前线程是否拥有Mutex的所有权，若为TRUE则指定为当前的创建线程为Mutex对象的所有者，其它线程访问需要先ReleaseMutex。lpName为Mutex的名称。

DWORD WINAPI WaitForSingleObject(
    HANDLE hHandle,                             //要获取的锁的句柄
    DWORD  dwMilliseconds                           //超时间隔
);
说明： WaitForSingleObject的作用是等待一个指定的对象(如Mutex对象)，直到该对象处于非占用的状态(如Mutex对象被释放)或超出设定的时间间隔。除此之外，还有一个与它类似的函数WaitForMultipleObjects，它的作用是等待一个或所有指定的对象，直到所有的对象处于非占用的状态，或超出设定的时间间隔。

hHandle：要等待的指定对象的句柄。dwMilliseconds：超时的间隔，以毫秒为单位；如果dwMilliseconds为非0，则等待直到dwMilliseconds时间间隔用完或对象变为非占用的状态，如果dwMilliseconds 为INFINITE则表示无限等待，直到等待的对象处于非占用的状态。

BOOL WINAPI ReleaseMutex(HANDLE hMutex);
说明：释放所拥有的互斥量锁对象，hMutex为释放的互斥量的句柄。

【Demo3】：线程同步
实例
#include "stdafx.h"
#include <windows.h>
#include <iostream>
 
#define NAME_LINE   40
 
//定义线程函数传入参数的结构体
typedef struct __THREAD_DATA
{
    int nMaxNum;
    char strThreadName[NAME_LINE];
 
    __THREAD_DATA() : nMaxNum(0)
    {
        memset(strThreadName, 0, NAME_LINE * sizeof(char));
    }
}THREAD_DATA;
 
HANDLE g_hMutex = NULL;     //互斥量
 
//线程函数
DWORD WINAPI ThreadProc(LPVOID lpParameter)
{
    THREAD_DATA* pThreadData = (THREAD_DATA*)lpParameter;
 
    for (int i = 0; i < pThreadData->nMaxNum; ++ i)
    {
        //请求获得一个互斥量锁
        WaitForSingleObject(g_hMutex, INFINITE);
        cout << pThreadData->strThreadName << " --- " << i << endl;
        Sleep(100);
        //释放互斥量锁
        ReleaseMutex(g_hMutex);
    }
    return 0L;
}
 
int main()
{
    //创建一个互斥量
    g_hMutex = CreateMutex(NULL, FALSE, NULL);
 
    //初始化线程数据
    THREAD_DATA threadData1, threadData2;
    threadData1.nMaxNum = 5;
    strcpy(threadData1.strThreadName, "线程1");
    threadData2.nMaxNum = 10;
    strcpy(threadData2.strThreadName, "线程2");
 
 
    //创建第一个子线程
    HANDLE hThread1 = CreateThread(NULL, 0, ThreadProc, &threadData1, 0, NULL);
    //创建第二个子线程
    HANDLE hThread2 = CreateThread(NULL, 0, ThreadProc, &threadData2, 0, NULL);
    //关闭线程
    CloseHandle(hThread1);
    CloseHandle(hThread2);
 
    //主线程的执行路径
    for (int i = 0; i < 5; ++ i)
    {
        //请求获得一个互斥量锁
        WaitForSingleObject(g_hMutex, INFINITE);
        cout << "主线程 === " << i << endl;
        Sleep(100);
        //释放互斥量锁
        ReleaseMutex(g_hMutex);
    }
 
    system("pause");
    return 0;
}
结果：

主线程 === 0 
线程1 — 0 
线程2 — 0 
主线程 === 1 
线程1 — 1 
线程2 — 1 
主线程 === 2 
线程1 — 2 
线程2 — 2 
主线程 === 3 
线程1 — 3 
线程2 — 3 
主线程 === 4 
线程1 — 4 
请按任意键继续… 线程2 — 4 
线程2 — 5 
线程2 — 6 
线程2 — 7 
线程2 — 8 
线程2 — 9
为进一步理解线程同步的重要性和互斥量的使用方法，我们再来看一个例子。

买火车票是大家春节回家最为关注的事情，我们就简单模拟一下火车票的售票系统(为使程序简单，我们就抽出最简单的模型进行模拟)：有500张从北京到赣州的火车票，在8个窗口同时出售，保证系统的稳定性和数据的原子性。

【Demo4】：模拟火车售票系统
SaleTickets.h
#include "stdafx.h"
#include <windows.h>
#include <iostream>
#include <strstream> 
#include <string>
 
using namespace std;
 
#define NAME_LINE   40
 
//定义线程函数传入参数的结构体
typedef struct __TICKET
{
    int nCount;
    char strTicketName[NAME_LINE];
 
    __TICKET() : nCount(0)
    {
        memset(strTicketName, 0, NAME_LINE * sizeof(char));
    }
}TICKET;
 
typedef struct __THD_DATA
{
    TICKET* pTicket;
    char strThreadName[NAME_LINE];
 
    __THD_DATA() : pTicket(NULL)
    {
        memset(strThreadName, 0, NAME_LINE * sizeof(char));
    }
}THD_DATA;
 
 
 //基本类型数据转换成字符串
template<class T>
string convertToString(const T val)
{
    string s;
    std::strstream ss;
    ss << val;
    ss >> s;
    return s;
}
 
//售票程序
DWORD WINAPI SaleTicket(LPVOID lpParameter);
SaleTickets.cpp
#include "stdafx.h"
#include <windows.h>
#include <iostream>
#include "SaleTickets.h"
 
using namespace std;
 
extern HANDLE g_hMutex;
 
//售票程序
DWORD WINAPI SaleTicket(LPVOID lpParameter)
{
 
    THD_DATA* pThreadData = (THD_DATA*)lpParameter;
    TICKET* pSaleData = pThreadData->pTicket;
    while(pSaleData->nCount > 0)
    {
        //请求获得一个互斥量锁
        WaitForSingleObject(g_hMutex, INFINITE);
        if (pSaleData->nCount > 0)
        {
            cout << pThreadData->strThreadName << "出售第" << pSaleData->nCount -- << "的票,";
            if (pSaleData->nCount >= 0) {
                cout << "出票成功!剩余" << pSaleData->nCount << "张票." << endl;
            } else {
                cout << "出票失败！该票已售完。" << endl;
            }
        }
        Sleep(10);
        //释放互斥量锁
        ReleaseMutex(g_hMutex);
    }
 
    return 0L;
}
测试程序：

//售票系统
void Test2()
{
    //创建一个互斥量
    g_hMutex = CreateMutex(NULL, FALSE, NULL);

    //初始化火车票
    TICKET ticket;
    ticket.nCount = 100;
    strcpy(ticket.strTicketName, "北京-->赣州");

    const int THREAD_NUMM = 8;
    THD_DATA threadSale[THREAD_NUMM];
    HANDLE hThread[THREAD_NUMM];
    for(int i = 0; i < THREAD_NUMM; ++ i)
    {
        threadSale[i].pTicket = &ticket;
        string strThreadName = convertToString(i);

        strThreadName = "窗口" + strThreadName;

        strcpy(threadSale[i].strThreadName, strThreadName.c_str());

        //创建线程
        hThread[i] = CreateThread(NULL, NULL, SaleTicket, &threadSale[i], 0, NULL);

        //请求获得一个互斥量锁
        WaitForSingleObject(g_hMutex, INFINITE);
        cout << threadSale[i].strThreadName << "开始出售 " << threadSale[i].pTicket->strTicketName << " 的票..." << endl;
        //释放互斥量锁
        ReleaseMutex(g_hMutex);

        //关闭线程
        CloseHandle(hThread[i]);
    }

    system("pause");
}
结果：

窗口0开始出售 北京–>赣州 的票… 
窗口0出售第100的票,出票成功!剩余99张票. 
窗口1开始出售 北京–>赣州 的票… 
窗口1出售第99的票,出票成功!剩余98张票. 
窗口0出售第98的票,出票成功!剩余97张票. 
窗口2开始出售 北京–>赣州 的票… 
窗口2出售第97的票,出票成功!剩余96张票. 
窗口1出售第96的票,出票成功!剩余95张票. 
窗口0出售第95的票,出票成功!剩余94张票. 
窗口3开始出售 北京–>赣州 的票… 
窗口3出售第94的票,出票成功!剩余93张票. 
窗口2出售第93的票,出票成功!剩余92张票. 
窗口1出售第92的票,出票成功!剩余91张票. 
窗口0出售第91的票,出票成功!剩余90张票. 
窗口4开始出售 北京–>赣州 的票… 
窗口4出售第90的票,出票成功!剩余89张票. 
窗口3出售第89的票,出票成功!剩余88张票. 
窗口2出售第88的票,出票成功!剩余87张票. 
窗口1出售第87的票,出票成功!剩余86张票. 
窗口0出售第86的票,出票成功!剩余85张票. 
窗口5开始出售 北京–>赣州 的票… 
窗口5出售第85的票,出票成功!剩余84张票. 
窗口4出售第84的票,出票成功!剩余83张票. 
窗口3出售第83的票,出票成功!剩余82张票. 
窗口2出售第82的票,出票成功!剩余81张票.
