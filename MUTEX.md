线程间的互斥：

eg:

 
//共享资源
static int num = 0;
//互斥锁
HANDLE  g_Mutex = CreateMutex(NULL, FALSE, NULL);
 
//子线程函数  
unsigned int __stdcall ChildThreadFunc(LPVOID pM)
{
	while (true)
	{
		Sleep(500);
 
		WaitForSingleObject(g_Mutex, INFINITE);//等待互斥量  INFINITE表示永远等待
		num++;
		printf("num:%d\n", num);
		ReleaseMutex(g_Mutex);
	}
	return 0;
}
 
int main()
{
	HANDLE handle[5] = { 0 };
 
	for (int i = 0; i < 5; i++)
	{
		handle[i] = (HANDLE)_beginthreadex(NULL, 0, ChildThreadFunc, NULL, 0, NULL);
	}
	
	//阻塞等待
	for (int i = 0; i < 5; i++)
	{
		WaitForSingleObject(handle[i], -1);
	}
 
	printf("主线程 num:%d\n", num);
	getchar();
	return 0;
}
进程中使用互斥量与在线程中使用其原理和操作流程相同，唯一区别在于线程中可以不为互斥量指定名称，而在进程中需要指定名称，由此其他进程可以根据名称获取该互斥量的句柄。

eg: 两个进程共享名字为pmutex的互斥量

其中一个进程代码：

#include <iostream>
#include <windows.h>
using namespace std;
 
 
int main()
{
	// 若不存在名为"pmutex"的互斥量则创建它；否则获取其句柄
    // Mutex 可以跨进程使用，所以其名称对整个系统而言是全局的，所以命名不要过于普通，类似：Mutex、Object 等。
    HANDLE hMutex = CreateMutex(NULL, false, "mypmutex");
    if(NULL == hMutex)
    {
        cout<<"create mutex error "<<GetLastError()<<endl;
        return 0;
    }
    else 
    {
        cout<<" create mutex success:"<<hMutex<<endl;
    }
 
     for(int i = 0;i<10; i++)
    {
		// 申请对互斥量的占有
        DWORD  d  = WaitForSingleObject(hMutex, INFINITE);
        if(WAIT_OBJECT_0 == d)
        {
			// 模拟对公共内存/文件的操作
            cout<<"begin sleep"<<endl;
            Sleep(2000);
            cout<<"process 1"<<endl;
			
			// 操作完毕，释放对互斥量的占有
            if(ReleaseMutex(hMutex)!=0)
            {
                cout<<"reslease ok"<<endl;
            }
            else
            {
                cout<<"reslease failed"<<endl;
            }
        }
        if(WAIT_ABANDONED == d)
        {
            cout<<"WAIT_ABANDONED"<<endl;
        }
        if(WAIT_FAILED ==d)
        {
            cout<<"mutex error"<<endl;
        }
        Sleep(2000);
    }
	
	// 释放互斥量
    CloseHandle(hMutex);
	hMutex = NULL;
    return 0;
}
 
另一个进程代码：

#include <iostream>
#include <windows.h>
using namespace std;
 
 
int main()
{
	// 若不存在名为"pmutex"的互斥量则创建它；否则获取其句柄
    HANDLE hMutex = CreateMutex(NULL, false, "mypmutex");
    if(NULL == hMutex)
    {
        cout<<"create mutex error "<<GetLastError()<<endl;
        return 0;
    }
    else 
    {
        cout<<" create mutex success:"<<hMutex<<endl;
    }
 
     for(int i = 0;i<10; i++)
    {
		// 申请对互斥量的占有
        DWORD  d  = WaitForSingleObject(hMutex, INFINITE);
        if(WAIT_OBJECT_0 == d)
        {
			// 模拟对公共内存/文件的操作
            cout<<"begin sleep"<<endl;
            Sleep(2000);
            cout<<"process 2"<<endl;
			
			// 操作完毕，释放对互斥量的占有
            if(ReleaseMutex(hMutex)!=0)
            {
                cout<<"reslease ok"<<endl;
            }
            else
            {
                cout<<"reslease failed"<<endl;
            }
        }
        if(WAIT_ABANDONED == d)
        {
            cout<<"WAIT_ABANDONED"<<endl;
        }
        if(WAIT_FAILED ==d)
        {
            cout<<"mutex error"<<endl;
        }
        Sleep(2000);
    }
	
	// 释放互斥量
    CloseHandle(hMutex);
	hMutex = NULL;
    return 0;
}
总结步骤就是：

1、创建一个互斥器：CreateMutex；
2、打开一个已经存在的互斥器：OpenMutex；
3、获得互斥器的拥有权：WaitForSingleObject、WaitForMultipleObjects ……（可能造成阻塞）；
4、释放互斥器的拥有权：ReleaseMutex；
5、关闭互斥器：CloseHandle；

Mutex是内核对象，陷入内核时间性能相对较差（与Critical Section相比）

WaitForMultipleObjects的例子：

#include <iostream>
#include <windows.h>
using namespace std;
 
HANDLE  g_hMutex = NULL;
const int g_Number = 3;
DWORD WINAPI ThreadProc1(__in  LPVOID lpParameter);
DWORD WINAPI ThreadProc2(__in  LPVOID lpParameter);
DWORD WINAPI ThreadProc3(__in  LPVOID lpParameter);
 
int main()
{
    g_hMutex = CreateMutex(NULL,FALSE,NULL);
    //TRUE代表主线程拥有互斥对象 但是主线程没有释放该对象  互斥对象谁拥有 谁释放 
    // FLASE代表当前没有线程拥有这个互斥对象
    HANDLE hThread[ g_Number ] = {0};
    int first = 1, second = 2, third = 3;
    hThread[ 0 ] = CreateThread(NULL,0,ThreadProc1,(LPVOID)first,0,NULL);
    hThread[ 1 ] = CreateThread(NULL,0,ThreadProc2,(LPVOID)second,0,NULL);
    hThread[ 2 ] = CreateThread(NULL,0,ThreadProc3,(LPVOID)third,0,NULL);
 
    WaitForMultipleObjects(g_Number,hThread,TRUE,INFINITE);
    CloseHandle( hThread[0] );
    CloseHandle( hThread[1] );
    CloseHandle( hThread[2] );
 
    CloseHandle( g_hMutex );
    return 0;
}
 
DWORD WINAPI ThreadProc1(__in  LPVOID lpParameter)
{
    WaitForSingleObject(g_hMutex, INFINITE);//等待互斥量
    cout<<(int)lpParameter<<endl;
    ReleaseMutex(g_hMutex);//释放互斥量
    return 0;
}
 
DWORD WINAPI ThreadProc2(__in  LPVOID lpParameter)
{
    WaitForSingleObject(g_hMutex, INFINITE);//等待互斥量
    cout<<(int )lpParameter<<endl;
    ReleaseMutex(g_hMutex);//释放互斥量
    return 0;
}
 
DWORD WINAPI ThreadProc3(__in  LPVOID lpParameter)
{
    WaitForSingleObject( g_hMutex, INFINITE);//等待互斥量
    cout<<(int)lpParameter<<endl;
    ReleaseMutex(g_hMutex);//释放互斥量
    return 0;
}
DWORD WaitForMultipleObjects的（DWORD NCOUNT，CONST HANDLE * lpHandles，BOOLfWaitAll，DWORDdwMilliseconds）;

四个参数分别是：

1. NCOUNT，DWORD类型，用于指定句柄数组的数量
2. lphObjects，指针类型，用于指定句柄数组的内存地址
3. fWaitAll，布尔类型，真表示函数等待所有指定句柄的对象有信号为止，如果bWaitAll为FALSE，则返回值减去WAIT_OBJECT_0表示满足等待条件的对象的lpHandles数组索引。
4. dwTimeout，DWORD类型，用于指定等待时间的超时时间，单位毫秒，可以是INFINITE
当WaitForMultipleObjects等待多个内核对象的时候，如果它的bWaitAll参数设置为false。其返回值包括WAIT_OBJECT_0 。如果同时有多个内核对象被触发，这个函数返回的只是其中序号最小的那个。 

说白了就是，要有限还是无限时间地，全部还是只要有一个完成就可以地，等待几个并且是哪几个内核对象完成。

通常可以根据WaitForMultipleObjects的返回值判断是哪个序号的内核对象设置了信号：

HANDLE h[3]; 
h[0] = hProcess1; 
h[1] = hProcess2; 
h[2] = hProcess3; 
DWORD dw = WaitForMultipleObjects(3, h, FALSE, 5000); 
switch(dw) 
{ 
case WAIT_FAILED: 
break; 
case WAIT_TIMEOUT: 
break; 
case WAIT_OBJECT_0 + 0; 
break; 
case WAIT_OBJECT_0 + 1; 
break; 
case WAIT_OBJECT_0 + 2; 
break; 
} 


但是上面的例子有一个问题，那就是比如hProcess1设置了信号，则switch必定返回WAIT_OBJECT_0+1

如果我们再一次获取，及时hProcess1没有产生信号，那么也还是会返回WAIT_OBJECT_0+1，而无法判断后面的内核对象的状态

这个时候需要手动处理一下。
