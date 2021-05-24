---
title: "Solve 'MemoryError: Unable to allocate XXXGiB for an array' by setting Overcommit and Swap"
classes: wide
categories:
  - Code
tags:
  - overcommit
  - swap
last_modified_at: 2021-05-23T16:00:52-04:00
---
### Problem Description
In my recent RL project, I need to generate a multidimensional Numpy array for a Q-table.

```python
self.qtable = np.zeros((2, 2,2,2,2,2,2,2,2,2,10, 2, 10, 2,100000))
```

However, as the array has a really big size,  the terminal sends out an error 

```
MemoryError: Unable to allocate 305. GiB for an array with shape (2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 10, 2, 100000) and data type float64
```

### Step1: Overcommit handling

Firstly, it exposes the problem of my Ubuntu system's [overcommit handling](https://www.kernel.org/doc/Documentation/vm/overcommit-accounting) mode.

The default overcommit handling is set to `0` , which means:

```
Heuristic overcommit handling. 
Obvious overcommits of address space are refused. Used for a typical system. 
It ensures a seriously wild allocation fails while allowing overcommit to reduce swap usage. 
The root is allowed to allocate slightly more memory in this mode. 
This is the default.
```

As ` 305. GiB `  is much greater than my actual physical memory space `32. GiB` . This obvious overcommit of address space is refused.

To check the current overcommit mode, you can run:

```bash
$ cat /proc/sys/vm/overcommit_memory
```

In my situation, as the array should be sparse, actually it will not occupy `305. GiB` memory space as what it claimed.  So It is fine to allow the overcommit. 

To enable the overcommit, you can run:

```bash
$ sudo -i
$ echo 1 > /proc/sys/vm/overcommit_memory
```

Changing this to `1` means:

```
Always overcommit. 
Appropriate for some scientific applications.
A classic example is a code using sparse arrays and just relying on the virtual memory consisting almost entirely of zero pages.
```

Now the system will allow you to declare a large array without worrying about how large it is. The system will only allocate physical memory pages for those explicit data in your sparse array.



### Step2: Swap (Virtual memory)

After setting the 'Overcommit handling mode' to `1`, I can start training my Q-learning model.

However,  the 'sparse' array becomes denser and denser when the training in progress.  Finally, it runs out of my `32.GiB` physical memory space at around epochs of 4750000 (Total: 8000000).`

As a result, the system has to terminate the training before it is finished.   :(

I need more available memory space!! I realize that increasing the `Swap` size would be helpful.

`Swap` is a special file located on your hard disk, which the system could use as an additional virtual memory space.

The default size of the `Swap` on my `Ubuntu20.04` system is `2.GiB`, which is insufficient. 

To extend the `Swap` space, 

1. Turn off all `Swap processes` by:

```bash
$ sudo swapoff -a
```



2. Resize the `Swap`:

(In my situation, I estimate an additional `32GiB` virtual memory should be sufficient)

```bash
$ sudo dd if=/dev/zero of=/swapfile bs=1G count=32
```

Note that:

```
if = input file
of = output file
bs = block size
count = multiplier of blocks
```



3. Change its permission:

```bash
$ sudo chmod 600 /swapfile
```

(`600` means that the owner has full read and write access to the file, while no other user can access the file)



4. Set the file as `Swap` and activate it:

```bash
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
```



5. Check the file `/etc/fstab` by

```bash
$ sudo vi /etc/fstab
```

Check whether the following command is there. If not, add it to the bottom.  

```
/swapfile none swap sw 0 0
```



6. Check the amount of `Swap` available by:

```bash
$ grep SwapTotal /proc/meminfo
```

Or,

```
$ free -t -m
```



After finishing these two steps, I can run my RL model without any error related to the memory space.



### Reference:

1. Unable to allocate array with shape and data type. [https://stackoverflow.com/questions/57507832/unable-to-allocate-array-with-shape-and-data-type](https://stackoverflow.com/questions/57507832/unable-to-allocate-array-with-shape-and-data-type)

2. Change swap size in Ubuntu 18.04 or newer. [https://bogdancornianu.com/change-swap-size-in-ubuntu/](https://bogdancornianu.com/change-swap-size-in-ubuntu/)

