---
layout: article
title:  "nginx重启丢失pid文件导致切割日志失败分析"
toc: true
disqus: true
categories: linux
image:
    teaser: /teaser/nginx.png
---

* TOC
{:toc}

>以下案例是通过strace来查看进程的系统调用分析问题，一起来体会下这个命令的强大之处。

## 1.错误现象
现象1：
nignx重启之后，发现nginx的pid文件不见了
现象2：
nginx日志切割文件失败了

## 2.模拟分析现象(分析其缘由)

1.开启一个窗口不断访问网站

{% highlight bash %}
{% raw %}
for i in `seq 1 1000`;do
sleep 1;
curl http://www.gbd.com/;curl http://www.gbd.com/;curl http://www.gbd.com/
{% endraw %}
{% endhighlight %}


2.使用strace监控nginx的master进程

{% highlight bash %}
{% raw %}
strace -tt -T -v -f -e trace=file -o /data/logs/strace_nginx.log -p 3276
监控这个进程的文件操作，详细去看strace的文档
{% endraw %}
{% endhighlight %}


3.重启nginx(使用官方的脚步-优雅关闭)

{% highlight bash %}
{% raw %}
/etc/init.nginx restart
{% endraw %}
{% endhighlight %}


4.查看进程


{% highlight bash %}
{% raw %}
[root@localhost ~]# ps uax | grep nginx   
root      3276  0.0  0.1  49420  4400 ?        Ss   23:35   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
www       3291  0.6  0.6  69432 24768 ?        S    23:35   0:00 nginx: worker process is shutting down                         
root      3334  0.0  0.1  49420  4272 ?        Ss   23:36   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
www       3335  0.2  0.6  69432 24468 ?        S    23:36   0:00 nginx: worker process                                          
www       3336  0.2  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3337  1.0  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3338  3.2  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3340  0.4  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3341  3.2  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3342  1.0  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3343  3.0  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3344  3.2  0.6  69432 24540 ?        S    23:36   0:00 nginx: worker process                                          
www       3345  2.0  0.6  69432 24520 ?        S    23:36   0:00 nginx: worker process                                          
root      3356  0.0  0.0 103256   884 pts/2    S+   23:36   0:00 grep nginx
{% endraw %}
{% endhighlight %}


分析走一波：

1. 官方使用的是优雅关闭，也就是关掉空闲连接和不接收新连接，等worker进程都结束了，则会关闭旧的master进程，这个时候就会出现上面的情况（这个时候的pid文件对应的pid号是 3334 ) 
2. 等这个旧master进程结束了之后，pid文件会顺便被干掉(但是这个时候你干掉的是新的master老大的pid文件，这么整，会没朋友的)
3. 这就是为什么会丢失pid文件的原因

5.查看strace检查系统调用情况

{% highlight bash %}
{% raw %}
 unlink("/var/run/nginx.pid") = 0 <0.000081>
{% endraw %}
{% endhighlight %}


分析：

unlink syscall 其实意义上也是删除文件

可以man下文档或者手动unlink，你就明白。


下面给出这个stop和start函数 官方的脚本

{% highlight bash %}
{% raw %}
start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

{% endraw %}
{% endhighlight %}


## 3.解决办法

解决方法1：简单粗暴，使用TERM或者INT来fast shutdown

解决办法2： 重启后，手动

echo "newpidnum" >> /var/run/nginx.pid

## 4.nginx日志切割模版

{% highlight bash %}
{% raw %}
/etc/logrotate.d/nginx
/var/log/nginx/*.log {
        #指定转储周期为每天
        daily
        missingok
        #指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份
        rotate 5
        #compress
        #delaycompress
        #如果是空文件的话，不转储
        notifempty
        #create 640 root adm
        sharedscripts
        postrotate
                [ ! -f /var/run/nginx.pid ] || kill -USR1 `cat /var/run/nginx.pid`
        endscript
}
{% endraw %}
{% endhighlight %}


切割测试

{% highlight bash %}
{% raw %}
/usr/sbin/logrotate -vf /etc/logrotate.d/nginx
{% endraw %}
{% endhighlight %}

## 5.controlling nginx

{% highlight bash %}
{% raw %}
TERM, INT	fast shutdown
QUIT	graceful shutdown
HUP	changing configuration, keeping up with a changed time zone (only for FreeBSD and Linux), starting new worker processes with a new configuration, graceful shutdown of old worker processes
USR1	re-opening log files
USR2	upgrading an executable file
WINCH	graceful shutdown of worker processes
{% endraw %}
{% endhighlight %}

## 参考
[Red Hat NGINX Init Script](https://www.nginx.com/resources/wiki/start/topics/examples/redhatnginxinit/)
[Controlling nginx](http://nginx.org/en/docs/control.html)