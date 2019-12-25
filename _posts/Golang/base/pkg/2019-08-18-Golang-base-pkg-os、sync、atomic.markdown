---
layout:     post
title:      "Go标准库-进程、os、sync、atomic"
subtitle:   ""
date:       2019-08-18 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### 创建进程

在Unix中，创建⼀个进程，通过系统调⽤fork实现（及其⼀些变种，如vfork、clone）。在Go语⾔中，Linux下创建进程使⽤的系统调⽤是clone。很多时候，系统调⽤fork、execve、wait和exit会在⼀起出现。此处先简要介绍这4个系统调⽤及其典型⽤法

- fork: 允许⼀进程（⽗进程）创建⼀新进程（⼦进程）。具体做法是，新的⼦进程⼏近于对⽗进程的翻版：⼦进程获得⽗进程的栈、数据段、堆和执⾏⽂本段的拷⻉。可将此视为把⽗进程⼀分为⼆
- exit(status)：终⽌⼀进程，将进程占⽤的所有资源（内存、⽂件描述符等）归还内核，交其进⾏再次分配。参数status为⼀整型变量，表示进程的退出状态。⽗进程可使⽤系统调⽤wait()来获取该状态
- wait(&status)⽬的有⼆：其⼀，如果进程尚未调⽤exit()终⽌，那么wait会挂起⽗进程直⾄⼦进程终⽌；其⼆，⼦进程的终⽌状态通过wait的status参数返回
- execve(pathname,argv,envp)加载⼀个新程序（路径名为pathname，参数列表为argv，环境变量列表为envp）到当前进程的内存。这将丢弃现存的程序⽂本段，并为新程序重新创建栈、数据段以及堆。通常将这⼀动作称为执⾏⼀个新程序


在Go语⾔中，没有直接提供fork系统调⽤的封装，⽽是将fork和execve合⼆为⼀，提供了syscall.ForkExec。如果想只调⽤fork，得⾃⼰通过syscall.Syscall(syscall.SYS_FORK,0,0,0)实现

##### os.Process 及其相关方法

os.Process 存储了通过 StartProcess 创建的进程的相关信息

```
type Process struct {
	Pid    int
	handle uintptr      // handle is accessed atomically on Windows
	isdone uint32       // process has been successfully waited on, non zero if true
	sigMu  sync.RWMutex // avoid race between wait and signal
}
```

###### StartProcess
⼀般通过 StartProcess 创建 Process 的实例

```
func StartProcess(name string, argv []string, attr *ProcAttr) (*Process, error)
```

它使⽤提供的程序名、命令⾏参数、属性开始⼀个新进程。StartProcess是⼀个低级别的接⼝。os/exec包提供了⾼级别的接⼝，⼀般应该尽量使⽤os/exec包。如果出错，错误的类型会是*PathError

其中的参数attr，类型是ProcAttr的指针，⽤于为StartProcess创建新进程提供⼀些属性。定义如下

```
type ProcAttr struct {
    // 如果 Dir ⾮空，⼦进程会在创建 Process 实例前先进⼊该⽬录。（即设为⼦进程的当前⼯作⽬
    录）
    Dir string
    // 如果 Env ⾮空，它会作为新进程的环境变量。必须采⽤ Environ 返回值的格式。
    // 如果 Env 为 nil，将使⽤ Environ 函数的返回值。
    Env []string
    // Files 指定被新进程继承的打开⽂件对象。
    // 前三个绑定为标准输⼊、标准输出、标准错误输出。
    // 依赖底层操作系统的实现可能会⽀持额外的⽂件对象。
    // nil 相当于在进程开始时关闭的⽂件对象。
    Files []*File
    // 操作系统特定的创建属性。
    // 注意设置本字段意味着你的程序可能会执⾏异常甚⾄在某些操作系统中⽆法通过编译。这时候可以
    通过为特定系统设置。
    // 看 syscall.SysProcAttr 的定义，可以知道⽤于控制进程的相关属性。
    Sys *syscall.SysProcAttr
}
```

###### FindProcess

FindProcess可以通过pid查找⼀个运⾏中的进程。该函数返回的Process对象可以⽤于获取关于底层操作系统进程的信息。在Unix系统中，此函数总是成功，即使pid对应的进程不存在

```
func FindProcess(pid int) (*Process, error)
```

Process提供了四个⽅法：Kill、Signal、Wait和Release。其中Kill和Signal跟信号相关，⽽Kill实际上就是调⽤Signal，发送了SIGKILL信号，强制进程退出。Release⽅法⽤于释放Process对象相关的资源，以便将来可以被再使⽤。该⽅法只有在确定没有调⽤Wait时才需要调⽤。Unix中，该⽅法的内部实现只是将Process的pid置为-1。

 Wait 方法

 ```
 func (p *Process) Wait() (*ProcessState, error)
 ```

 在多进程应⽤程序的设计中，⽗进程需要知道某个⼦进程何时改变了状态——⼦进程终⽌或因收到信号⽽停⽌。Wait⽅法就是⼀种⽤于监控⼦进程的技术。Wait⽅法阻塞直到进程退出，然后返回⼀个ProcessState描述进程的状态和可能的错误。Wait⽅法会释放绑定到Process的所有资源。在⼤多数操作系统中，Process必须是当前进程的⼦进程，否则会返回错误

 ProcessState 的内部结构

 ```
type ProcessState struct {
    pid int // The process's id.
    status syscall.WaitStatus // System-dependent status info.
    rusage *syscall.Rusage
}
 ```

 ProcessState保存了Wait函数报告的某个进程的信息。status记录了状态原因，通过syscal.WaitStatus类型定义的⽅法可以判断

 - Exited()：是否正常退出，如调⽤os.Exit；
 - Signaled()：是否收到未处理信号⽽终⽌；
 - CoreDump()：是否收到未处理信号⽽终⽌，同时⽣成coredump⽂件，如SIGABRT
 - Stopped()：是否因信号⽽停⽌（SIGSTOP）；
 - Continued()：是否因收到信号SIGCONT⽽恢复


示例程序

执行外部命令 ls,并通过ProcessState获取执行的基本信息

```
package main

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"time"
)

func main() {
	if len(os.Args) < 2 {
		fmt.Printf("Usage: %s [command]\n", os.Args[0])
		os.Exit(1)
	}
	cmdName := os.Args[1]
	if filepath.Base(os.Args[1]) == os.Args[1] {
		if lp, err := exec.LookPath(os.Args[1]); err != nil {
			fmt.Println("look path error:", err)
			os.Exit(1)
		} else {
			cmdName = lp
		}
	}
	procAttr := &os.ProcAttr{
		Files: []*os.File{os.Stdin, os.Stdout, os.Stderr},
	}
	cwd, err := os.Getwd()
	if err != nil {
		fmt.Println("look path error:", err)
		os.Exit(1)
	}
	start := time.Now()
	process, err := os.StartProcess(cmdName, []string{cwd}, procAttr)
	if err != nil {
		fmt.Println("start process error:", err)
		os.Exit(2)
	}
	processState, err := process.Wait()
	if err != nil {
		fmt.Println("wait error:", err)
		os.Exit(3)
	}
	fmt.Println()
	fmt.Println("real", time.Now().Sub(start))
	fmt.Println("user", processState.UserTime())
	fmt.Println("system", processState.SystemTime())
}

输出

cmd             main.go         sort.go         time_test.go
cmd.go          ring.go         time.go         timer.go

real 6.30397ms
user 1.483ms
system 4.023ms
```

#####  运行外部命令

通过os包可以做到运⾏外部命令，如前⾯的例⼦。不过，Go标准库为我们封装了更好⽤的包：os/exec，运⾏外部命令，应该优先使⽤它，它包装了os.StartProcess函数以便更容易的重定向标准输⼊和输出，使⽤管道连接I/O，以及作其它的⼀些调整。

###### 查找可执⾏程序

exec.LookPath函数在PATH指定⽬录中搜索可执⾏程序，如file中有/，则只在当前⽬录搜索。该函数返回完整路径或相对于当前路径的⼀个相对路径。

```
func LookPath(file string) (string, error)

// 例
str, err := exec.LookPath("/Users/AllWorkSpace/GoWorkSpace/src/base/")
```

###### CMD相关命令

Cmd结构代表⼀个正在准备或者在执⾏中的外部命令，调⽤了Run、Output或CombinedOutput后，Cmd实例不能被重⽤

```
type Cmd struct {
	// Path 是将要执⾏的命令路径。
    // 该字段不能为空（也是唯⼀⼀个不能为空的字段），如为相对路径会相对于 Dir 字段。
    // 通过 Command 初始化时，会在需要时调⽤ LookPath 获得完整的路径
	Path string

	// Args 存放着命令的参数，第⼀个值是要执⾏的命令（Args[0])；如果为空切⽚或者nil，使⽤ {Path} 运⾏。
    // ⼀般情况下，Path 和 Args 都应被 Command 函数设定
	Args []string


	// Env 指定进程的环境变量，如为 nil，则使⽤当前进程的环境变量，即 os.Environ()，⼀般就是当前系统的环境变量
	Env []string

	// Dir 指定命令的⼯作⽬录。如为空字符串，会在调⽤者的进程当前⼯作⽬录下执⾏
	Dir string
	// Stdin 指定进程的标准输⼊，如为 nil，进程会从空设备读取（os.DevNull）
    // 如果 Stdin 是 *os.File 的实例，进程的标准输⼊会直接指向这个⽂件
    // 否则，会在⼀个单独的 goroutine 中从 Stdin 中读数据，然后将数据通过管道传递到该命令中
    //（也就是从 Stdin 读到数据后，写⼊管道，该命令可以从管道读到这个数据）。在 goroutine 停⽌数据
    // 拷⻉之前（停⽌的原因如遇到EOF或其他错误，或管道的 write 端错误），Wait ⽅法会⼀直堵塞
	Stdin io.Reader

	// Stdout 和 Stderr 指定进程的标准输出和标准错误输出。
    // 如果任⼀个为 nil，Run ⽅法会将对应的⽂件描述符关联到空设备（os.DevNull）
    // 如果两个字段相同，同⼀时间最多有⼀个线程可以写⼊
	Stdout io.Writer
	Stderr io.Writer

	// ExtraFiles specifies additional open files to be inherited by the
	// new process. It does not include standard input, standard output, or
	// standard error. If non-nil, entry i becomes file descriptor 3+i.
	//
	// ExtraFiles is not supported on Windows.
	ExtraFiles []*os.File

	// SysProcAttr holds optional, operating system-specific attributes.
	// Run passes it to os.StartProcess as the os.ProcAttr's Sys field.
	SysProcAttr *syscall.SysProcAttr

	// Process is the underlying process, once started.
	Process *os.Process

	// ProcessState contains information about an exited process,
	// available after a call to Wait or Run.
	ProcessState *os.ProcessState

	ctx             context.Context // nil means none
	lookPathErr     error           // LookPath error, if any.
	finished        bool            // when Wait was called
	childFiles      []*os.File
	closeAfterStart []io.Closer
	closeAfterWait  []io.Closer
	goroutine       []func() error
	errch           chan error // one send per goroutine
	waitDone        chan struct{}
}
```

执行方法

```
// CMD
func Command(namme string, agg ...strig)*Cmd
```
该函数返回一个*Cmd,用于使用给出的参数执行name指定的程序。
返回的*Cmd只设定了Path和Args两个字段

如果name不好喊路径分隔符，将使用LookPath获取完整路径;否则直接使用name

得到*Cmd实例过后:
- 调用 Start(),接着调用Wait(),然后会阻塞直到命令执行完成
- 调用 Run(),它内部会调用Start(),接着调用Wait()


```
// Start
func (c *Cmd)Start() error

开始执⾏c包含的命令，但并不会等待该命令完成即返回。Wait⽅法会返回命令的退出状态码并在命令执⾏完后释放相关的资源。内部调⽤os.StartProcess，执⾏forkExec。


// Wait
func (c *Cmd)Wait() error

Wait会阻塞直到该命令执⾏完成，该命令必须是先通过Start执⾏。如果命令成功执⾏，stdin、stdout、stderr数据传递没有问题，并且返回状态码为0，⽅法的返回值为nil；如果命令没有执⾏或者执⾏失败，会返回*ExitError类型的错误；否则返回的error可能是表示I/O问题。如果c.Stdin不是*os.File类型，Wait会等待，直到数据从c.Stdin拷⻉到进程的标准输⼊。Wait⽅法会在命令返回后释放相关的资源。


//Output
// Output运行命令并返回其标准输出。
// 错误通常都是* ExitError类型。
//如果c.Stderr为nil，则Output填充ExitError.Stderr。
func (c *Cmd) Output() ([]byte, error)


//CombinedOutput
Output()只返回Stdout的结果，⽽CombinedOutput组合Stdout和Stderr的输出，即Stdout和Stderr都赋值为同⼀个bytes.Buffer。
```


StdoutPipe、StderrPipe 和 StdinPipe

```
除了上⾯介绍的Output和CombinedOutput直接获取命令输出结果外，还可以通过StdoutPipe返回io.ReadCloser来获取输出；相应的StderrPipe得到错误信息；⽽StdinPipe则可以往命令写⼊数据

func (c *Cmd) StdoutPipe() (io.ReadCloser, error)

StdoutPipe⽅法返回⼀个在命令Start执⾏后与命令标准输出关联的管道。Wait⽅法会在命令结束后会关闭这个管道，所以⼀般不需要⼿动关闭该管道。但是在从管道读取完全部数据之前调⽤Wait出错了，则必须⼿动关闭

func (c *Cmd) StderrPipe() (io.ReadCloser, error)

StderrPipe⽅法返回⼀个在命令Start执⾏后与命令标准错误输出关联的管道。Wait⽅法会在命令结束后会关闭这个管道，⼀般不需要⼿动关闭该管道。但是在从管道读取完全部数据之前调⽤Wait出错了，则必须⼿动关闭

func (c *Cmd) StdinPipe() (io.WriteCloser, error)

StdinPipe⽅法返回⼀个在命令Start执⾏后与命令标准输⼊关联的管道。Wait⽅法会在命令结束后会关闭这个管道。必要时调⽤者可以调⽤Close⽅法来强⾏关闭管道。例如，标准输⼊已经关闭了，命令执⾏才完成，这时调⽤者需要显示关闭管道

因为 Wait 之后，会将管道关闭，所以，要使⽤这些⽅法，只能使⽤ Start + Wait 组
合，不能使⽤ Run
```

###### 执行外部命令

通过Cmd实例后，有两种⽅式运⾏命令。有时候，我们不只是简单的运⾏命令，还希望能控制命令的输⼊和输出。通过上⾯的API介绍，控制输⼊输出有⼏种⽅法

- 得到Cmd实例后，直接给它的字段Stdin、Stdout和Stderr赋值
- 通过 Output 或 CombinedOutput 获得输出
- 通过带 Pipe 后缀的⽅法获得管道，⽤于输⼊或输出

```
package main

import (
	"bytes"
	"os/exec"
)

func main() {

}

func FillStd(name string, arg ...string) ([]byte, error){
	cmd := exec.Command(name, arg...)
	var out = new(bytes.Buffer)
	cmd.Stdout = out
	cmd.Stderr = out
	err := cmd.Run()
	if err != nil {
		return nil, err
	}
	return out.Bytes(), nil
}

func UseOutput(name string, arg ...string) ([]byte, error) {
	return exec.Command(name, arg...).Output()
}

func UsePipe(name string, arg ...string) ([]byte, error){
	cmd := exec.Command(name, arg...)
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		return nil, err
	}
	if err = cmd.Start(); err != nil {

		return nil, err
	}
	var out = make([]byte, 0, 1024)
	for {
		tmp := make([]byte, 128)
		n, err := stdout.Read(tmp)
		out = append(out, tmp[:n]...)
		if err != nil {
			break
		}
	}
	if err = cmd.Wait(); err != nil {
		return nil, err
	}
	return out, nil
}
```

### 进程相关

##### 获取进程ID
每个进程都会有⼀个进程ID，可以通过os.Getpid获得。同时，每个进程都有创建⾃⼰的⽗进程，通过os.Getppid获得

##### 进程凭证
unix中进程都有⼀套数字表示的⽤户ID(UID)和组ID(GID)，有时也将这些ID称之为进程凭证。Windows下总是-1。

##### 实际用户ID和实际组ID
实际⽤户ID（realuserID）和实际组ID（realgroupID）确定了进程所属的⽤户和组。登录shell从/etc/passwd⽂件读取⽤户ID和组ID。当创建新进程时（如shell执⾏程序），将从其⽗进程中继承这些ID。可通过os.Getuid()和os.Getgid()获取当前进程的实际⽤户ID和实际组ID

##### 有效⽤户 ID 和有效组 ID

⼤多数Unix实现中，当进程尝试执⾏各种操作（即系统调⽤）时，将结合有效⽤户ID、有效组ID，连同辅助组ID⼀起来确定授予进程的权限。内核还会使⽤有效⽤户ID来决定⼀个进程是否能向另⼀个进程发送信号。有效⽤户ID为0（root的⽤户ID）的进程拥有超级⽤户的所有权限。这样的进程⼜称为特权级进程（privilegedprocess）。某些系统调⽤只能由特权级进程执⾏。可通过os.Geteuid()和os.Getegid()获取当前进程的有效⽤户ID（effectiveuserID）和有效组ID（effectviegroupID）。

通常，有效⽤户ID及组ID与其相应的实际ID相等，但有两种⽅法能够致使⼆者不同。⼀是使⽤相关系统调⽤；⼆是执⾏set-user-ID和set-group-ID程序


包os/user允许通过名称或ID查询⽤户账号

```
type User struct {
    Uid string // user id
    Gid string // primary group id
    Username string
    Name string
    HomeDir string
}
```

Current函数可以获取当前⽤户账号。⽽Lookup和LookupId则分别根据⽤户名和⽤户ID查询⽤户。如果对应的⽤户不存在，则返回user.UnknownUserError或user.UnknownUserIdError

```
fmt.Println(user.Current())
fmt.Println(user.Lookup("yangping"))
fmt.Println(user.LookupId("0"))


&{501 20 yangping 杨平 /Users/yangping} <nil>
&{501 20 yangping 杨平 /Users/yangping} <nil>
&{0 0 root System Administrator /var/root} <nil>
```

#### 线程

与进程类似，线程是允许应⽤程序并发执⾏多个任务的⼀种机制。⼀个进程可以包含多个线程。同⼀个程序中的所有线程均会独⽴执⾏相同程序，且共享同⼀份全局内存区域。

同⼀进程中的多个线程可以并发执⾏。在多处理器环境下，多个线程可以同时并⾏。如果⼀个线程因等待I/O操作⽽遭阻塞，那么其他线程依然可以继续运⾏。

在Linux中，通过系统调⽤clone()来实现线程的。从前⾯的介绍，我们知道，该系统调⽤也可以⽤来创建进程。实际上，从内核的⻆度来说，它并没有线程这个概念。Linux把所有的线程都当作进程来实现。内核并没有准备特别的调度算法或是定义特别的数据结构来表征线程。相反，线程仅仅被视为⼀个使⽤某些共享资源的进程。所以，在内核中，它看起来就是⼀个普通的进程（只是该进程和其他⼀些进程共享某些资源，如地址空间）

在Go中，通过clone()系统调⽤来创建线程，其中的clone_flags为:

```
cloneFlags = _CLONE_VM | /* share memory */
        _CLONE_FS | /* share cwd, etc */
        _CLONE_FILES | /* share fd table */
        _CLONE_SIGHAND | /* share sig handler table */
        _CLONE_THREAD /* revisit - okay for now */
```

也就是说，⽗⼦俩共享了地址空间(_CLONE_VM)、⽂件系统资源(_CLONE_FS)、⽂件描述符(_CLONE_FILES)和信号处理程序(_CLONE_SIGHAND)。⽽_CLONE_THREAD则会将⽗⼦进程放⼊相同的线程组。这样⼀来，新建的进程和⽗进程都叫做线程


#### 进程间通信

##### ⾮类型安全操作

unsafe库徘徊在“类型安全”边缘，由于它们绕过了Golang的内存安全原则，⼀般被认为使⽤该库是不安全的。但是，在许多情况下，unsafe库的作⽤⼜是不可替代的，灵活地使⽤它们可以实现对内存的直接读写操作。在reflect库、syscall库以及其他许多需要操作内存的开源项⽬中都有对它的引⽤。

unsafe库源码极少，只有两个类型的定义和三个⽅法的声明

###### Arbitrary 类型

官⽅导出这个类型只是出于完善⽂档的考虑，在其他的库和任何项⽬中都没有使⽤价值，除⾮程序员故意使⽤它

###### Pointer 类型

这个类型⽐较重要，它是实现定位欲读写的内存的基础。官⽅⽂档对该类型有四个重要描述

- 任何类型的指针都可以被转化为Pointer
- Pointer可以被转化为任何类型的指针
- uintptr可以被转化为Pointer
- Pointer可以被转化为uintptr

```
func main() {
    i := 100
    fmt.Println(i) // 100
    p := (*int)unsafe.Pointer(&i)
    fmt.Println(*p) // 100
    *p = 0
    fmt.Println(i) // 0
    fmt.Println(*p) // 0
}
```
###### Sizeof 函数

```
func Sizeof(v ArbitraryType) uintptr
```

Sizeof函数返回变量v占⽤的内存空间的字节数，该字节数不是按照变量v实际占⽤的内存计算，⽽是按照v的“toplevel”内存计算。⽐如，在64位系统中，如果变量v是int类型，会返回16，因为v的“toplevel”内存就是它的值使⽤的内存；如果变量v是string类型，会返回16，因为v的“toplevel”内存不是存放着实际的字符串，⽽是该字符串的地址；如果变量v是slice类型，会返回24，这是因为slice的描述符就占了24个字节

###### Offsetof 函数

```
func Offsetof(v ArbitraryType) uintptr
```

该函数返回由v所指示的某结构体中的字段在该结构体中的位置偏移字节数，注意，v的表达⽅式必须是“struct.filed”形式。

```
package main

import (
	"fmt"
	"unsafe"
)

type Datas struct {
	c0 byte
	c1 int
	c2 string
	c3 int
}

func main() {
	var d Datas
	fmt.Println(unsafe.Offsetof(d.c0)) // 0
	fmt.Println(unsafe.Offsetof(d.c1)) // 8
	fmt.Println(unsafe.Offsetof(d.c2)) // 16
	fmt.Println(unsafe.Offsetof(d.c3)) // 32
}

```

###### Alignof

```
//返回结构体中某个field的对齐值（字节对齐的原因）
func Alignof(x ArbitraryType) uintptr
```


```
package main

import (
	"fmt"
	"unsafe"
)

type Datas struct {
	c0 byte
	c1 int
	c2 string
	c3 int
}

func main() {
	var x struct {
		a bool
		b int16
		c []int
	}

	/**
	  unsafe.Offsetof 函数的参数必须是一个字段 x.f, 然后返回 f 字段相对于 x 起始地址的偏移量, 包括可能的空洞.
	*/

	/**
	  uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)
	  指针的运算
	*/
	// 和 pb := &x.b 等价
	pb := (*int16)(unsafe.Pointer(uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)))
	*pb = 42
	fmt.Println(x.b) // "42"
}

```

#### sync - 处理同步需求

golang是⼀⻔语⾔级别⽀持并发的程序语⾔。golang中使⽤go语句来开启⼀个新的协程。goroutine是⾮常轻量的，除了给它分配栈空间，它所占⽤的内存空间是微乎其微的。

但当多个goroutine同时进⾏处理的时候，就会遇到⽐如同时抢占⼀个资源，某个goroutine等待另⼀个goroutine处理完某⼀个步骤之后才能继续的需求。在golang的官⽅⽂档上，作者明确指出，golang并不希望依靠共享内存的⽅式进⾏进程的协同操作。⽽是希望通过管道channel的⽅式进⾏。当然，golang也提供了共享内存，锁，等机制进⾏协同操作的包。sync包就是为了这个⽬的⽽出现的。

##### 锁

sync包中定义了Locker结构来代表锁

```
type Locker interface {
    Lock()
    Unlock()
}
```

并且创造了两个结构来实现Locker接⼝：Mutex 和 RWMutex

Mutex就是互斥锁，互斥锁代表着当数据被加锁了之后，除了加锁的程序，其他程序不能对数据进⾏读操作和写操作。这个当然能解决并发程序对资源的操作。但是，效率上是个问题。当加锁后，其他程序要读取操作数据，就只能进⾏等待了。

这个时候就需要使⽤读写锁。读写锁分为读锁和写锁，读数据的时候上读锁，写数据的时候上写锁。有写锁的时候，数据不可读不可写。有读锁的时候，数据可读，不可写。互斥锁就不举例⼦，读写锁可以看下⾯的例⼦：

```
package main

import (
	"sync"
	"time"
)

type Human struct {
	tes  bool
	sex  bool
	age  uint8
	min  int
	name string
}

var m *sync.RWMutex
var val = 0

func main() {
	m = new(sync.RWMutex)
	go read(1)
	go write(2)
	go read(3)
	time.Sleep(5 * time.Second)
}

func read(i int) {
	m.RLock()
	time.Sleep(1 * time.Second)
	println("val", val)
	time.Sleep(1 * time.Second)
	m.RUnlock()
}

func write(i int) {
	m.Lock()
	val = 10
	time.Sleep(1 * time.Second)
	m.Unlock()
}


// 一般情况下

val : 0
val : 10
```

###### 对象池

当多个goroutine都需要创建同⼀个对象的时候，如果goroutine过多，可能导致对象的创建数⽬剧增。⽽对象⼜是占⽤内存的，进⽽导致的就是内存回收的GC压⼒徒增。造成“并发⼤－占⽤内存⼤－GC缓慢－处理并发能⼒降低－并发更⼤”这样的恶性循环。在这个时候，我们⾮常迫切需要有⼀个对象池，每个goroutine不再⾃⼰单独创建对象，⽽是从对象池中获取出⼀个对象（如果池中已经有的话）。这就是sync.Pool出现的⽬的了。

sync.Pool的使⽤⾮常简单，提供两个⽅法:Get和Put和⼀个初始化回调函数New

看下⾯这个例⼦（取⾃gomemcache）:

```
// keyBufPool returns []byte buffers for use by PickServer's call to
// crc32.ChecksumIEEE to avoid allocations. (but doesn't avoid the
// copies, which at least are bounded in size and small)
var keyBufPool = sync.Pool{
	New: func() interface{} {
		b := make([]byte, 256)
		return &b
	},
}
func (ss *ServerList) PickServer(key string) (net.Addr, error) {
	ss.mu.RLock()
	defer ss.mu.RUnlock()
	if len(ss.addrs) == 0 {
		return nil, ErrNoServers
	}
	if len(ss.addrs) == 1 {
		return ss.addrs[0], nil
	}
	bufp := keyBufPool.Get().(*[]byte)
	n := copy(*bufp, key)
	cs := crc32.ChecksumIEEE((*bufp)[:n])
	keyBufPool.Put(bufp)
	return ss.addrs[cs%uint32(len(ss.addrs))], nil
}
```

###### Once - 只执行一次操作

操作只希望被执⾏⼀次

```
func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
		}()
	}
	time.Sleep(3e9)
}


只会打印一次 Only once
```

###### WaitGroup 和 Cond

⼀个goroutine需要等待⼀批goroutine执⾏完毕以后才继续执⾏，那么这种多线程等待的问题就可以使⽤WaitGroup了

```
func main() {
    wp := new(sync.WaitGroup)
    wp.Add(10);
    for i := 0; i < 10; i++ {
    go func() {
    fmt.Println("done ", i)
    wp.Done()
    }()
    }
    wp.Wait()
    fmt.Println("wait end")
}
```

还有个sync.Cond是⽤来控制某个条件下，goroutine进⼊等待时期，等待信号到来,,然后重新启动

```
func main() {
    locker := new(sync.Mutex)
    cond := sync.NewCond(locker)
    done := false
    cond.L.Lock()
    go func() {
    time.Sleep(2e9)
    done = true
    cond.Signal()
    }()
    if (!done) {
    cond.Wait()
    }
    fmt.Println("now done is ", done);
}

这⾥当主goroutine进⼊cond.Wait的时候，就会进⼊等待，当从goroutine发出信号之后，主goroutine才会继续往下⾯⾛
```

sync.Cond还有⼀个BroadCast⽅法，⽤来通知唤醒所有等待的gouroutine

```
package main

import (
	"fmt"
	"sync"
	"time"
)

var locker = new(sync.Mutex)
var cond = sync.NewCond(locker)

func test(x int) {
	cond.L.Lock()
	cond.Wait() // 等待通知 暂时阻塞
	fmt.Println(x)
	time.Sleep(time.Second * 1)
	cond.L.Unlock() // 释放锁，不释放的话将只会有⼀次输出
}
func main() {
	for i := 0; i < 40; i++ {
		go test(i)
	}
	fmt.Println("start all")
	cond.Broadcast() // 下发⼴播给所有等待的goroutine
	time.Sleep(time.Second * 60)
}

```

主gouroutine开启后，可以创建多个从gouroutine，从gouroutine获取锁后，进⼊cond.Wait状态，当主gouroutine执⾏完任务后，通过BroadCast⼴播信号。处于cond.Wait状态的所有gouroutine收到信号后将全部被唤醒并往下执⾏。需要注意的是，从gouroutine执⾏完任务后，需要通过cond.L.Unlock释放锁，否则其它被唤醒的gouroutine将没法继续执⾏。

```
func (c *Cond) Wait() {
    c.checker.check()
    if raceenabled {
        raceDisable()
    }
    atomic.AddUint32(&c.waiters, 1)
    if raceenabled {
        raceEnable()
    }
    // 释放
    c.L.Unlock()
    runtime_Syncsemacquire(&c.sema)
    // 重新上锁
    c.L.Lock()
}
```

Cond.Wait会⾃动释放锁等待信号的到来，当信号到来后，第⼀个获取到信号的Wait将继续往下执⾏并从新上锁，如果不释放锁，其它收到信号的gouroutine将阻塞⽆法继续执⾏。由于各个Wait收到信号的时间是不确定的，因此每次的输出顺序也都是随机的

#### sync/atomic - 原子操作

对于并发操作⽽⾔，原⼦操作是个⾮常现实的问题。典型的就是i++的问题。当两个CPU同时对内存中的i进⾏读取，然后把加⼀之后的值放⼊内存中，可能两次i++的结果，这个i只增加了⼀次。如何保证多CPU对同⼀块内存的操作是原⼦的。golang中sync/atomic就是做这个使⽤的。

具体的原⼦操作在不同的操作系统中实现是不同的。⽐如在Intel的CPU架构机器上，主要是使⽤总线锁的⽅式实现的。⼤致的意思就是当⼀个CPU需要操作⼀个内存块的时候，向总线发送⼀个LOCK信号，所有CPU收到这个信号后就不对这个内存块进⾏操作了。等待操作的CPU执⾏完操作后，发送UNLOCK信号，才结束。在AMD的CPU架构机器上就是使⽤MESI⼀致性协议的⽅式来保证原⼦操作。所以我们在看atomic源码的时候，我们看到它针对不同的操作系统有不同汇编语⾔⽂件。如果我们善⽤原⼦操作，它会⽐锁更为⾼效。

##### CAS

原⼦操作中最经典的CAS(compare-and-swap)在atomic包中是Compare开头的函数

```
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)

func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer)
(swapped bool)

func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)

func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)
```

###### 增加或减少

对⼀个数值进⾏增加或者减少的⾏为也需要保证是原⼦的，它对应于atomic包的函数如下:

```
func AddInt32(addr *int32, delta int32) (new int32)

func AddInt64(addr *int64, delta int64) (new int64)

func AddUint32(addr *uint32, delta uint32) (new uint32)

func AddUint64(addr *uint64, delta uint64) (new uint64)

func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)
```

###### 读取或写入

当我们要读取⼀个变量的时候，很有可能这个变量正在被写⼊，这个时候，我们就很有可能读取到写到⼀半的数据。所以读取操作是需要⼀个原⼦⾏为的。在atomic包中就是Load开头的函数群

```
func LoadInt32(addr *int32) (val int32)

func LoadInt64(addr *int64) (val int64)

func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)

func LoadUint32(addr *uint32) (val uint32)

func LoadUint64(addr *uint64) (val uint64)

func LoadUintptr(addr *uintptr) (val uintptr)
```

是同样的，如果有多个CPU往内存中⼀个数据块写⼊数据的时候，可能导致这个写⼊的数据不完整。在atomic包对应的是Store开头的函数群。

```
func StoreInt32(addr *int32, val int32)

func StoreInt64(addr *int64, val int64)

func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)

func StoreUint32(addr *uint32, val uint32)

func StoreUint64(addr *uint64, val uint64)

func StoreUintptr(addr *uintptr, val uintpt
```