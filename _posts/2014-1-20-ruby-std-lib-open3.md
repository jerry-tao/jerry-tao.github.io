---
layout: post
title:  "Ruby标准库学习：Open3"
date:   2014-01-20 11:47:06
categories: ruby-std
image: /assets/images/post.jpg
---

Open3是Ruby标准库中的一员，主要提供对系统命令行的调用。

官方描述是，开启一个可以访问stdout，stdin，stderr三个标准流管道的子进程。

先看一个Hello World。

{% highlight ruby %}
require 'open3'
Open3.popen3("pwd"){|i,o,e,t| p o.readline }
#stdin,stdout,stderr,thread
#=> prints '/home/jerry\n'.
{% endhighlight %}

Open3的示例方法popen3：

{% highlight ruby %}
  def popen3(*cmd, &block)
    if Hash === cmd.last
      opts = cmd.pop.dup
    else
      opts = {}
    end

    in_r, in_w = IO.pipe
    opts[:in] = in_r
    in_w.sync = true

    out_r, out_w = IO.pipe
    opts[:out] = out_w

    err_r, err_w = IO.pipe
    opts[:err] = err_w

    popen_run(cmd, opts, [in_r, out_w, err_w], [in_w, out_r, err_r], &block)
  end
  module_function :popen3
{% endhighlight %}

在Open3里还有几个其他方法，跟popen3都差不多。罗列一下：

- Open3.popen3 : pipes for stdin, stdout, stderr
- Open3.popen2 : pipes for stdin, stdout
- Open3.popen2e : pipes for stdin, merged stdout and stderr
- Open3.capture3 : give a string for stdin.  get strings for stdout, stderr
- Open3.capture2 : give a string for stdin.  get a string for stdout
- Open3.capture2e : give a string for stdin.  get a string for merged stdout and stderr


基本上调用的都是popen_run,Open3的核心方法之一：

{% highlight ruby %}
  def popen_run(cmd, opts, child_io, parent_io) # :nodoc:
    pid = spawn(*cmd, opts)
    wait_thr = Process.detach(pid)
    child_io.each {|io| io.close }
    result = [*parent_io, wait_thr]
    if defined? yield
      begin
        return yield(*result)
      ensure
        parent_io.each{|io| io.close unless io.closed?}
        wait_thr.join
      end
    end
    result
  end
{% endhighlight %}

当需要进行管道重定向的时候 比如执行 ls | grep 'foo'，还可以使用Open3#pipeline*一系列方法

举例：

{% highlight ruby %}
Open3.pipeline_rw("ls","grep 'index'"){|i,o,t| p o.readline}
#=> "index.html\n"
{% endhighlight %}

管道系列方法pipeline_rw:

{% highlight ruby %}
  def pipeline_rw(*cmds, &block)
    if Hash === cmds.last
      opts = cmds.pop.dup
    else
      opts = {}
    end

    in_r, in_w = IO.pipe
    opts[:in] = in_r
    in_w.sync = true

    out_r, out_w = IO.pipe
    opts[:out] = out_w

    pipeline_run(cmds, opts, [in_r, out_w], [in_w, out_r], &block)
  end
  module_function :pipeline_rw
{% endhighlight %}

类似的方法：

- Open3.pipeline_rw : pipes for first stdin and last stdout of a pipeline
- Open3.pipeline_r : pipe for last stdout of a pipeline
- Open3.pipeline_w : pipe for first stdin of a pipeline
- Open3.pipeline_start : run a pipeline and don't wait
- Open3.pipeline : run a pipeline and wait

核心方法：

{% highlight ruby %}
 def pipeline_run(cmds, pipeline_opts, child_io, parent_io) # :nodoc:
    if cmds.empty?
      raise ArgumentError, "no commands"
    end

    opts_base = pipeline_opts.dup
    opts_base.delete :in
    opts_base.delete :out

    wait_thrs = []
    r = nil
    cmds.each_with_index {|cmd, i|
      cmd_opts = opts_base.dup
      if String === cmd
        cmd = [cmd]
      else
        cmd_opts.update cmd.pop if Hash === cmd.last
      end
      if i == 0
        if !cmd_opts.include?(:in)
          if pipeline_opts.include?(:in)
            cmd_opts[:in] = pipeline_opts[:in]
          end
        end
      else
        cmd_opts[:in] = r
      end
      if i != cmds.length - 1
        r2, w2 = IO.pipe
        cmd_opts[:out] = w2
      else
        if !cmd_opts.include?(:out)
          if pipeline_opts.include?(:out)
            cmd_opts[:out] = pipeline_opts[:out]
          end
        end
      end
      pid = spawn(*cmd, cmd_opts)
      wait_thrs << Process.detach(pid)
      r.close if r
      w2.close if w2
      r = r2
    }
    result = parent_io + [wait_thrs]
    child_io.each {|io| io.close }
    if defined? yield
      begin
        return yield(*result)
      ensure
        parent_io.each{|io| io.close unless io.closed?}
        wait_thrs.each {|t| t.join }
      end
    end
    result
  end
{% endhighlight %}

require open3的时候执行的代码：

{% highlight ruby %}
if $0 == __FILE__
  a = Open3.popen3("nroff -man")
  Thread.start do
    while line = gets
      a[0].print line
    end
    a[0].close
  end
  while line = a[1].gets
    print ":", line
  end
end
{% endhighlight %}

查看[官方API][open3]可以得到更详细的介绍。
[open3]: http://www.ruby-doc.org/stdlib-2.0/libdoc/open3/rdoc/Open3.html
