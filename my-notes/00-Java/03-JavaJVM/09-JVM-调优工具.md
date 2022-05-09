### [**jstat 命令**](https://blog.csdn.net/zhaozheng7758/article/details/8623549)

​		对 Java 应用程序的资源和性能进行实时的命令行的监控，可以详细查看堆内各个部分的使用量，以及加载类的数量。包括了对 Heap size 和垃圾回收状况的监控。

> jstat -options								 # 可以列出当前 JVM 版本支持的选项
>
> jstat –class \<pid\>				 	 	# 显示加载 class 的数量，及所占空间等信息。
>
> jstat -compiler \<pid\>		   		# 显示 VM 实时编译的数量等信息。
>
> jstat -gc \<pid\>					   		# 可以显示 gc 的信息，查看 gc 的次数，及时间。
>
> **jstat -gccapacity \<pid\>**	 		# 可以显示，VM 内存中三代（young,old,perm）对象的使用和占用大小
>
> jstat -gcutil \<pid\>				 		# 统计 gc 信息
>
> jstat -gcnew \<pid\>						# 年轻代对象的信息。
>
> jstat -gcnewcapacity \<pid\>  		# 年轻代对象的信息及其占用量。
>
> jstat -gcold \<pid\>						  # old代对象的信息。
>
> jstat -gcoldcapacity \<pid\>		   # old代对象的信息及其占用量。
>
> jstat -gcpermcapacity \<pid\>		# perm对象的信息及其占用量。
>
> jstat -printcompilation \<pid\>	  # 当前VM执行的信息。



### jmap 命令

> jmap pid																	# 查看进程的内存映像信息
>
> jmap -heap pid														# 显示 Java 堆详细信息，打印堆的摘要信息，包括使用的GC算法、堆配置信息和各内存区域内存使用信息
>
> jmap -histo:live pid												 # 显示堆中对象的统计信息，其中包括每个Java类、对象数量、内存大小(单位：字节)、完全限定的类名。
>
> jmap -clstats pid													  # 打印类加载器信息
>
> jmap -finalizerinfo pid											# 打印等待终结的对象信息
>
> jmap -dump:live,format=b,file=dump.hprof pid								# 生成堆转储快照dump文件。
>

### jhat 命令

分析内存快照，内置了web容器，可以通过浏览器分析堆内存快照。

> jhat -J-Xmx512m dump.hprof				# 有时 dump 出来的堆很大，在启动时会报堆空间不足的错误，可加参数：-J-Xmx512m 

