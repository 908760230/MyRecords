# 死循环内的sleep函数

看到别人框架内，线程函数死循环里sleep了一毫秒。顿时对这个很不解，问了一下大佬，同时也看了一下其他人写的博客。

对这个函数的作用大概弄懂了。

我从两方面来写我的理解：

## **1）** 
如果函数内时刻有任务需要执行，那么sleep就是多余的。

比如：
```c++
void print(){
    while(true){
        cout<<i++<<endl;
    }
}
```
在这个不断打印i值的函数里，加入sleep就是多余的，妨碍执行速度。

但是很多时候并不是这种情况。
```c++
while (true)
		{
			std::this_thread::sleep_for(std::chrono::milliseconds(1));
			
			//pick the first task and do it
			NFThreadTask task;
			if (mTaskList.TryPop(task))
			{
				task.xThreadFunc->operator()(task);

				//repush the result to the main thread
				//and, do we must to tell the result?
				if (task.xEndFunc)
				{
					m_pThreadPoolModule->TaskResult(task);
				}
			}
		}
```
在原作者的框架里，在线程内不断的pop出task，然后执行里面的回调函数。
这里有一个问题就是taskList里面并不是一直都有task执行，也就是说出现什么都没做却一直占用cpu的情况。

## **2）**
线程是进程的逻辑流，是进程在cpu上执行的最小单位。

这里就需要考虑操作系统的调度算法了：

windows是抢占式的谁的优先级高，谁就先执行。

linux是基于时间片，每一个线程的执行分配一个时间片，如果在规定的时间内未执行完，则会保存到队列的尾部，等待继续执行。

死循环内未加sleep可能对linux 的影响较小，但是对windows来说，如果未sleep，cpu一直被当前线程占据。其他线程会出现饿死的情况。而且cpu的使用率会过高，但是却未被合理使用。