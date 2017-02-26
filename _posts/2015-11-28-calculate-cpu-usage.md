---
layout: post
title: "Technology blog -101"
date: 2015-11-28
---


---

How to calculate CPU Usage -/proc/stat vs top


Why top reports a different value for CPU usage compared to SeaLion?
This is one of the questions that SeaLion(a monitoring tool I used to develop) team is asked every now and then. It is because most users schedule top command as top -b -n1 where the option -n specifies number of iterations over which top should summarize. But to summarize effective CPU utilization you should have at least two iterations as -n1 shows the total CPU utilization since the system boot. To do so, one can use -n2 which will let top sample the output over two iterations. But that’s not it, one should know that this sampling is done over a default time interval of 3 seconds or the value configured in toprc which means it gives effective utilization over a period of 3 seconds by default. However, this value could be given explicitly using the option -d in top command. So, running top -b -n2 -d1 will give you the effective CPU utilization, sampling the contents of two iterations over a period of 1 second.
Like top, SeaLion agent also samples contents of /proc/stat, at an interval of 1 second. So, it is due to this difference of sample intervals and missing out on correct number of iterations that there is no correspondence between the values reported by top and SeaLion.
To confirm the same, we ran an experimental Python script which accepts two arguments.The argument -p specifies the sample interval for /proc/stat and -t specifies the sample interval for top command. The script basically starts two subprocesses in parallel. One of them calculates directly from /proc/stat and the other one uses the output from top. It also creates multiple threads to simulate CPU usage. The script is run with Jython as it has no GIL which allows it to take full advantage of multithreading to simulate CPU usage. One can also use option -s to see details of process in top output. The script can be found here.
Let’s run the script by specifying the sample interval only for /proc/stat.
As you can see, CPU utilization reported by /proc/stat and that reported by top does not match. This happened because we didn’t specify the interval for top, it runs with the default sampling interval of 3 seconds. But in case of /proc/stat it was specified to run with sampling interval of 1 second. The time taken by script which is 3.169 seconds verifies that top command took around 3 seconds to finish.
Let’s see what happens when the intervals are same.
In this case utilization reported by /proc/stat and top is correspondent. As it is clear from the screenshot, top command runs with option -d using the argument provided to the script thereby using it as sampling interval. Since, both commands use the same interval, the outputs are correspondent.
Why some processes show CPU usage more than 100% in top output?
CPU usage for some processes, as reported by top, sometimes shoots higher than 100%. It happens only when a multithreaded process is running on a machine with multiple processors/cores. Effective CPU utilization for a process is calculated as a percentage of number of ticks elapsed by CPU being in user mode or kernel mode to the total number of ticks elapsed. If it is a multithreaded process, other cores of processor are also utilized summing the total utilization percentage to be more than 100.
To verify the same, the above mentioned script was executed on a dual core machine using option -s, showing the effective CPU utilization for a process as reported by top output.
In this case 458 is the total number of ticks for the process in user mode and kernel mode combined and that was over 3 seconds. Since 1 tick equals 10 ms, so 458 ticks equals 4.58 seconds and calculating percentage as 4.58/3 * 100 will give you 152.67, which is almost equal to the value reported by top.
So, how do I find actual CPU usage?
CPU usage refers to the time taken by CPU cycles when it is in user mode or kernel mode. Niced processes also belong to user mode whereas software and hardware interrupts are handled in kernel mode. If it is not doing any of these then either the CPU is idle or it is waiting for an I/O to complete.
From the above results, effective utilization can be calculated as sum of %us, %sy, %ni, %hi and %si, which in this case is 68.71%. Although some people may question as to why iowait is not included in CPU usage when very high value of iowait could be a reason for bad performance. It is because CPU usage by definition includes only the time spent by CPU cycles in running some processes or handling interrupts. When any process is waiting for an I/O to complete, it is not contributing to CPU cycles. So, ideally iowait is considered as a subset of idle CPU but it is true that high iowait is a bad sign and one needs to be aware of it. If a process is in uninterruptible sleep doing a disk read/write, it will result in high iowait and in that case Load Average will start spiking. However, if a process is in interruptible sleep for example waiting for a socket connection, it will contribute to high iowait but that will occur in small bursts. Since the situation of processes in uninterruptible sleep is not something which will frequently occur, SeaLion does not include iowait in CPU usage while calculating effective CPU utilization.




---

