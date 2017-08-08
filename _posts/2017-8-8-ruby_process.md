---
layout: post
title: "Ruby Process"
date: 2017-8-8
categories: Ruby
image: /assets/images/stopwatch-2061845_1280.jpg
---

基本上来说，在linux平台下，各个语言的进程操作都差不多，区别主要是写法和封装的程度，底层调用都是一样的，所以其实如果了解Linux的进程基本就足够了。

## 启动和终结

启动一个进程有两个接口，exec和fork，前者会用新的进程来替代当前进程，后者会以当前进程为父进程创建一个新的子进程，并且会继承当前进程的内存空间和文件描述符。

新启动的进程默认的前三个文件描述符012就是stdin，stdout和stderr。

exec会启动另外一个进程来取代当前进程  一旦启动，唯一不变的就是pid ppid user group 代码段换新，废弃原有数据段和堆栈段，重新分配。pending的信号会被舍弃，正在捕获的信号会使用默认的处理，自定义的丢失，内存lock会舍弃。thread属性会返回默认值，进程统计信息重置。atexit等东西会被清除。

```ruby
exec ['ls', '-l']
puts 'hello' # 不会被执行，exec之后进程就已经被替换成新的了
```

exec接受的参数可以是字符串或者数组，数组会直接运行另一个进程，字符串会新建一个新的session，一般没必要使用数组即可。

fork基本上是复制当前进程到一个新的进程，会继承当前的内存和文件描述符。

这里已经可以看到fork有些问题，首先是如果每fork一个进程就复制一份内存这个开销很大而且没有必要，所以现在的fork基本都支持CoW，只有当修改的时候才复制对应内存，否则不会复制对应内存。

另一个问题是，已经提到过，只有exec和fork两种启动进程的方式，那么我们当前进程，比如说正在运行的一个rb文件，想启动另外一个进程的话就需要先fork一个进程，然后用fork出来的进程执行exec，那么其实并没有必要跟这个子进程共享内存，虽然是CoW，但是还是要有一些开销的，所以还有另一个posix-spawn的方法，这个只会继承文件描述符，而不会继承内存，ruby里需要额外加载posix-spawn gem来实现。

```ruby
if fork
  puts 'In parent #{Process.pid}'
else
  puts 'In child #{Process.pid}'
end

# 以block方式运行fork

fork do
  puts "In child"
end

```

上面是fork的代码，因为在父进程中执行fork会返回fork出来进程的pid，所以if里面的代码会在parent进程里执行，else里的代码会在子进程里执行。

终结当前进程有很多方法，我们常见的就是代码执行完毕正常退出，这种代码其实默认会执行一个exit返回0。

所以可以在任何地方执行exit来退出当前进程，并且可以指定一个退出值，exit 22。

另外还可以加一段回调操作：

```ruby
at_exit { puts "Shutdown..." }
exit
```

同时可以使用abort，abort跟exit的主要区别是abort的默认退出值是1。

还有一个退出是exit!，这个退出默认退出值是1，同时不会执行at_exit回调。

还有一种情况就是运行过程中发生了异常并且没有进行捕捉，也会直接导致进程以非0退出。

然后来看下终结一个fork出去的子进程，在Linux下，结束一个进程会发一个SIGCHLD信号给父进程，如果父进程没有处理，那这个信号会遗留在系统内核里成为僵尸进程。

还有另一个点就是，如果父进程先于子进程结束了，那么子进程的父进程会变为init(1)，之后由init来处理子进程结束的信号。

```ruby
fork do
  sleep 10
  puts "Child end"
end

Process.wait # 等待子进程完结的信号
```

wait是一个阻塞操作，一共有四个类似方法，wait/wait2/waitpid/waitpid2。对于带2的来说，主要区别在返回值上，wait系的只会返回子进程的pid，而带2的会同时返回退出的状态码。

```ruby
fork do
  sleep 10
  puts "Child end"
  exit 22
end
puts Process.wait2
```

另外wait这个操作，如果不指定参数，那么任一子进程都会触发，所以如果有多个子进程，需要执行多次Process.wait。

```ruby
10.times do |i|
  fork do
    sleep i
    puts "Process #{Process.pid}"
  end
end

10.times do
  Process.wait
end
```

因为wait是个阻塞调用，有些时候并不是很方便，所以有三种办法来避免阻塞：

```ruby
# 1. detach
cpid = fork {...}
Process.detach(cpid) # 实际上是新建一个线程去执行wait

# 2. 当有信号的时候才执行wait
trap(:CHLD) do
  Process.wait
end

# 3. 使用非阻塞的wait

Process.wait(-1, Process::WNOHANG)
```

在上面的代码里，处理信号的有一点小问题，因为信号的并发很可能会导致丢失处理，结合2和3优化一下就是：

```ruby
trap(:CHLD) do
  while Process.wait(-1, Process::WNOHANG)
  end
end
```

这样就可以避免由于信号并发产生的问题。

## Daemon

Daemon也是我们常见的一个东西了，用人话说其实就是后台运行一个进程(不过其实后台运行有很多种方法，Daemon只是其中一种)。

Daemon其实并不复杂，在Ruby里可以直接用Process.daemon来创建一个daemon进程，不过还是详细的介绍下这个过程。

```ruby
exit if fork
Process.setsid
Dir.chdir('/')
STDIN.reopon '/dev/null' # 重定向stdin，out， err
STDERR.reopon '/dev/null', 'a'
STDOUT.reopon '/dev/null', 'a'
```

正如我们上面说的，父进程先于子进程结束，子进程的父进程会变成init，所以在上面我们fork出一个新的进程之后直接退出父进程。

然后我们新建一个session，防止当前session对新进程的影响。

把工作目录切换到根目录，防止在daemon过程中当前目录被删除导致错误。

重定向stdin，out和err，因为默认的stdin，out和err还是当前的。

大部分daemon主要流程都是这样，不管是在ruby里还是其他语言里面，这里面有些时候会在setsid之后再fork一次，主要原因是当setsid之后，这个进程就成为了session的leader，实际上还是可以给这个session attach一个终端来进行操作的，所以一般为了防止这种情况还会再fork一次。

## 通信

首先进程间可以互相传递信号，上面的SIGCHLD就是。

```ruby
trap(:USR1) do
  puts 'Signal USR1'
end
```

信号基本上就是个简单的通知机制，而且也不是那么可靠。

上面说过，fork出去的进程会继承父进程的文件描述符，所以我们可以打开一些可以通信的内容。

```ruby
r, w = IO.pipe # 创建一个新的可读写的管道，实际上是两个文件，在linux里都是文件嘛，所以会被fork出去的进程继承
fork do
  r.close
  w.write "Good for you"
end

w.close
puts r.read
```

管道是单向通信的，双向的可以用Unix Socket

```ruby
require 'socket'
sock_a, sock_b = Socket.pair(:UNIX, :DGRAM, 0)
fork do
  sock_a.close
  sock_b.send("Good for you",0)
end
sock_b.close
puts sock_a.recv(1000)
```

## Other

**修改进程名称**

```
$PROGRAM_NAME = "My Web Server"
```

这个名字就是显示在ps和top命令中的进程名字。

**执行其他命令**

```ruby
Process.spawn('ls') # fork之后exec ls
system 'ls' # 阻塞版的spawn 约等于Process.wait(Process.spawn('ls')) 返回退出值
`ls` # 对比system来说，system是共享stdin,out和err，而``会把stdout作为返回值返回，stderr共享
IO.popen('ls') # fork之后exec同时可以把stdin,out或者err的一个提取出来作为pipe使用
Open3.popen3('ls') # 对比IO.popen来说，这个是可以把stdin,out和err都重定向出来作为pipe使用
```

事实上，除非写个服务器或者sidekiq这种东西，否则一般情况下还真的很少会遇见写进程相关的代码，不过理解一下进程还是很有帮助的。
