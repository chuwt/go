# Go运行时 (runtime)
## before
- version: 1.17.1

## 启动流程
The bootstrap sequence is:
1. call osinit
2. call schedinit
3. make & queue new G
4. call runtime·mstart
5. The new G calls runtime·main.

程序启动后首先执行的是[runtime.proc](src/runtime/proc.go)下的`main`方法，用户编写的main方法叫做`main_main`。
```
//go:linkname main_main main.main
func main_main()
// 这是一个使用linkname的方法，要是用此方法需要引入"unsafe"包
```
### runtime.main
0. 已经初始化了m0和g0
1. 初始化sysmon
    1. 创建一个m来运行sysmon方法，sysmon直接运行在m上，没有p
       1. sysmon主要包含以下功能
          - 网络轮训
            - 启动空闲的p和m，将就绪的g放入全局
          - 计时器
          - 线程抢占处理(retake)
            - 遍历所有p，如果p运行过长，则抢占
            - 如果psyscall超过一个sysmon周期，则抢占
          - 垃圾回收
            - 检查是否需要强制gc
2. 锁住当前g在主线程上(lockOSThread)
   1. _g_.m.lockedg = _g_
   2. _g_.lockedm = _g_.m
3. 初始化runtimede里package里的init方法
   1. go forcegchelper() 辅助gc
      1. 初始化状态和锁
      2. gopark暂停
         1. gopark会检查park原因（reason）
         2. 设置一些当前ctx
         3. mcall调用切换到g0执行park_m
            1. 获取当前g
            2. 将g的状态改为Gwaiting
            3. dropg 将g和m解绑
               1. 将当前g.m.curg.m 设置为nil
               2. 将当前g.m.curg 设置为nil
            4. schedule调度核心
4. newproc
   1. 上述go关键字触发newproc调用
   2. 调用newproc1
      1. 获取当前p
      2. 从p中获取未使用的g（gfget）只已经结束或者刚初始化的g
         1. 如果g队列为空，则从全局中偷取，直到p.gFree大于32
      3. 如果g为空
         1. 创建一个g(malg)
         2. 修改g的状态为Gdead
         3. 将g添加到全局变量allg中
      4. 计算堆栈和sp的位置
      5. 设置g为可运行
   3. 放入当前p的队列，如果满了，放入全局队列
   4. 如果mainStarted已经完成，则唤醒
      1. 如果没有空闲p，则返回
      2. 如果存在自旋的m，则返回
      3. startm, 自旋为true
5. 开启gc（开启两个携程）
   1. go bgsweep(c) 开启一个新线程
      1. 暂时挂起（gopark), 等待goready
   2. go bgscavenge(c) 开启一个新线程
      1. 暂时挂起（gopark), 等待goready
6. 初始化main（用户代码）的init方法 



## 重要方法
1. allocm 新建一个m
2. malg   新建一个g
3. newosproc 创建内核线程
4. acquirem 对m加锁，禁止被抢占
5. gfget 从p的本地队列中获取一个g，如果没有，则从全局中偷

