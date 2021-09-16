# Go运行时 (runtime)
## before
- version: 1.17.1

## 启动流程
1. runtime/rt0_darwin_amd64.s:_rt0_amd64_darwin
2. runtime/asm_adm64.s:_rt0_amd64
3. runtime/asm_adm64.s:runtime·rt0_go
```
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
	// copy arguments forward on an even stack
	MOVQ	$runtime·g0(SB), DI // 变量赋值到runtime.g0
	MOVQ	runtime·m0+m_tls(SB), AX // 变量赋值到runtime.m0
	MOVQ	CX, m_g0(AX) // 绑定m0.g0 = g0
	MOVQ	AX, g_m(CX) // 绑定g0.m0 = m0

	CALL	runtime·check(SB) // 代码检查

	CALL	runtime·args(SB) // 调用 runtime.args
	CALL	runtime·osinit(SB) // 调用 runtime.osinit
	CALL	runtime·schedinit(SB) // // 调用 runtime.schedinit

	MOVQ	$runtime·mainPC(SB), AX		// entry 赋值runtime.main
	CALL	runtime·newproc(SB) // 创建一个g包含runtime.main方法

	// start this M
	CALL	runtime·mstart(SB) // 启动m
```

### runtime.osinit
1. 不同系统的osinit不同
### runtime.schedinit
2. 定义各种等级的锁，高等级的锁必须先持有低等级的锁
3. 获取当前g（g0)
4. 设置调度最大的m数量（maxmcount）为1_0000
5. STW
6. 检查go module
7. 初始化stack内存分配池（用于内存分配）
8. 初始化内存管理
9. 初始化m
10. 初始化gc
11. 初始化p（procresize）
    1. 会重新修改当前的所有p
    2. 并释放当前g.m的p（如果存在），然后将g.m.p = allp[0]
12. worldStarted
### runtime·mstart
1. mstart0
   1. mstart1
      1. 重新初始化m0
      2. 初始化m0的信号
      3. 调用main_main（用户方法）
      4. schedule()调度
         1. 如果当前运行的g的m存在lockedg
            1. 将当前m与对应的p解绑
               1. 如果存在其他g（本地或者全局）或者需要netpoll
                  1. 开启（复用）一个m与p绑定，然后唤醒m
               2. 阻塞m，等待唤醒
               3. 判断当前g的状态，如果为可用状态
               4. 将m绑定到nextp上
                  1. 如果之前的p已经被别人使用了，则抛异常
            2. 继续执行lockedg
         2. 设置top标记
         3. 获取当前p，设置为不能抢占
         4. 如果准备gc了
            1. gcstopm
               1. 如果g.m是自旋的，设置为非自旋
               2. 解绑当前p和m
               3. p的状态设置为gcstop
               4. 设置sched.stopwait
               5. 如果sched.stopwait == 0，则唤醒sched.stopnote
               6. 将m设置到midle队列中
               7. mpark，等待信号通知重新运行
            2. 返回top标记处运行
         5. 如果当前需要gc，则将gc的工作绑定到当前p
         6. 如果没有gc，并且当p.schedtick%61 == 0 && 全局队列存在g
            1. 从全局队列中获取一个g
         7. 如果还是没有g，则从本地队列中获取g
         8. 如果本地队列也为空，则从其他p中偷取，或者全局中，或者netpoll中
         9. 如果当前gp的lockedm存在，则将当前p释放，赋值给lockedm，然后本身的m进行阻塞
         10. 运行g


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
   4. 如果mainStarted已经完成，则唤醒p
      1. 如果没有空闲p，则返回
      2. 如果存在自旋的m，则返回
      3. startm, 使用或创建一个m，并且获取一个空闲p来运行g, 自旋为true，如果没有空闲的p，则返回
5. 开启gc（开启两个携程）
   1. go bgsweep(c) 开启一个新线程
      1. 暂时挂起（gopark), 等待goready
   2. go bgscavenge(c) 开启一个新线程
      1. 暂时挂起（gopark), 等待goready
6. 初始化main（用户代码）的init方法
7. 解锁当前g在主线程运行
8. 运行main_main
9. 检测panic后的defer
10. 退出


## 简要说明
```
1. g0初始化
2. m0初始化，m0.g0 = g0
3. runtime.schedinit() 中初始化allp
4. m0.p = allp[0]
5. 创建一个任务为runtime.main()的g
5. 调用mstart()运行 g
   5.1 系统栈调用newm()新建一个m运行sysmon方法，此时不需要p
   5.2 newm() 中调用 newm1() 调用 newosproc() 调用 mstart 运行sysmon方法
   5.3 sysmon是一个for循环，主要处理以下内容
      5.3.1 网络轮询: startm()启动空闲的p和m，将就绪的g放入全局队列中（由于sysmon没有p，所以会直接放入全局，当存在p时，会先计算空闲的p，然后放入全局空闲p的数量，再放入p中，多出来的再放入全局）
      5.3.2 计时器
      5.3.3 线程抢占(retake):
         5.3.3.1 遍历所有p，如果p运行过长，则抢占(preemptone), 设置g的抢占标记为true
         5.3.3.2 如果p当前为syscall超过一个sysmon周期，则抢占，调用handoffp()
            5.3.3.2.1 handoffp
            5.3.3.2.2 如果p队列不为空，则运行startm()，获取一个空闲的m然后唤醒m
            5.3.3.2.3 设置p为空闲状态，然后判读是否需要唤醒netpoll
      5.3.4 垃圾回收
   5.4 lockOSThread()，将m和g绑定
   5.5 初始化runtime包里的init方法
      5.5.1 go forcegchelper()
         5.5.1.1 newproc()
            5.5.1.1.1 创建或复用一个g
            5.5.1.1.2 获取当前p，然后放入本地队列，如果满了，放入全局队列 
	  		5.5.1.1.3 调用startm，启动一个自旋线程
      5.5.2 gopark(), 将当前g与m解绑,g状态改为wait，然后调用schedule()
   5.6 开启gc, gcenable()
      5.6.1 go bgsweep()
      5.6.2 go bgscavengo()
   5.7 初始化用户main包里的init
   5.8 unlockOSThread() 将当前g和m解绑
   5.9 调用main_main()
   5.10 退出
6. 调用mstart进行schedule()调度
   6.1 如果m是自旋的，当m找到一个可运行的g之后，会修改状态为非自旋
```

## 一些说明
1. mstart是已经绑定了p的m开始执行调度
2. startm是启动或新建一个m来和p绑定，新建的m会执行mstart开始调度，而复用的m只需要唤醒即可
3. 调度的本质就是m不断的执行schedule来获取可用的g来执行
4. 特殊情况g系统调用时，会将p和m解绑，当系统调用结束后，m会将g放入全局队列，然后stopm，将m放入空闲队列, 如果当前g和当前m是绑定关系，则会mpark，等待其他m调度这个g的时候，发现g与m绑定，然后出让他的p给当前m，然后唤醒当前m，当前m开始执行调度
5. 特殊情况是当g调用时间过长被抢占，p与m解绑，然后p会与其他的m绑定，继续执行，这叫做handoffp，sysmon会将p抢占，将p与m分离，此时会设置g的调用栈到抢占的代码段。当g在系统调用结束后，会执行抢占的代码段，此时会调用goschedImpl，返回调度，此时会将m与g解绑，然后将g丢入全局队列中，m会开始调度
7. 当m执行完g之后，会继续寻找可以执行的g，此时会根据是否存在自旋*2>pidle来决定是否自旋，如果自旋就会不断的寻找 ，找到后设置为非自旋，然后执行。没有自旋的如果没有找到，则先解绑m和p，然后stopm阻塞m，等待唤醒

## 调度流程说明
1. go func() 会调用newproc创建一个g
2. wakep，如果存在自旋的m，忽略，如果不存在，则startm创建一个m绑定p，然后mstart启动schedule
3. schedule会不断寻找g执行
4. 当g存在系统调用时，p与m会分离。当g执行返回时，m会与g分离，g被放入全局队列，m执行stopm，进入空闲列表。如果g绑定到了这个m上，则m会进入mpark阻塞，
此时g在全局被其他m的schedule获取到后，发现g有绑定的m，所以将p出让给绑定的m，然后唤醒他，自己进入stopm状态

## 参考理解
- [https://zboya.github.io/post/go_scheduler](https://zboya.github.io/post/go_scheduler)
- [https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/05.2.html)
- [https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/schedule/](https://golang.design/under-the-hood/zh-cn/part2runtime/ch06sched/schedule/)