### Android Binder 进程间通信机制一

#### 1.Linux进程通信的方式，Android系统采用的那个方式进行进程间通信的

linux系统内核提供了丰富的进程间通信的机制：

* 管道（Pipe）
* 信号（Signal）
* 消息队列（Message）
* 共享内存（Share Memory）
* Socket

而Android 并没有采用上面这些进程间通信的机制，才是采用**Binder机制来进行通信**。从性能来考虑，除了比共享内存的方式，性能略差之外，相对其方式性能都是最好的，Binder只进行一次内存数据拷贝，提高效率，节省内存空间。



#### 2.Binder简单认识

* Android系统使用的进程间通信机制（IPC）
* 是一个驱动（/dev/binder）
* 上层是一个Binder.java 实现了IBinder接口

#### 3.多进程的应用场景以及优势

视频播放、音频播放、大图浏览、推送

系统服务：打电话、闹钟

以上这些都是跨进程的，多进程的优势：

* **开辟更多的内存**，一个进程开辟的内存是由限制的。可以通过

  ```shell
  adb shell getprop dalvik.vm.heapsize
  ```

  来查看一个进程允许分配的最大的内存空间，我得到的结论是512M，所以多进程可以分配更多的内存。

* **风险隔离**，将有风险的功能放到一个独立的进程里边，就算出现崩溃，也不会影响主进程的使用。

#### 4.Binder通信机制的优势

![image-20220710173102475](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220710173102475.png)

* 从性能的角度来看，Binder机制性能由于需要进行一次数据拷贝略低于共享内存，但是性能高于其他的IPC机制。
* 从易用性来看，Binder机制采用CS架构，方便使用易用性高，共享内存控制起来复杂，易用性差。
* 从安全性来看，Binder机制为每个APP分配UID同时支持实名或者匿名的方式，保证了通信的安全性。共享内存或者Socket依赖上层协议来访问接入点是开放的，也就是说，任何一个进程都可以访问，这样是不安全的。

#### 5.Android系统内存划分

![image-20220710174230196](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220710174230196.png)

* Android系统内存划分成两块：用户空间和内核空间，用户空间是用户程序代码运行可以使用的内存，内核空间是内核代码运行可以使用的内存。两个空间是隔离的，相互不受影响。
* 32位系统，总共可以分配的内存空间4G，内核空间占用1G，用户空间占用3G。64位系统：地位0-47位才是有效的可变地址，高位全部补0或者1，补0的表示用户空间，补1的表示内核空间。
* 还有一点需要注意，在使用的时候，内存地址的后边都会保留一块空的地址作为安全保护区，用来检测非法指针。

#### 6.Binder的数据传输过程

![image-20220710175943003](C:/Users/刘静盼/AppData/Roaming/Typora/typora-user-images/image-20220710175943003.png)

当一个进程需要将数据传递给另外一个进程的时候，Binder驱动程序就会将这些数据从用户空间复制到内核空间。然后Binder程序会将目标进程的内存池中分配出一小块内核缓冲区来保存这些数据，并将用户空间的地址告诉进程，目标进程就可以访问数据了。这样做的好处就是不需要将数据从内核空间复制到用户空间了。



#### 7.Binder设备文件内存映射的过程

函数binder_mmap在``kernel/goldfish/drivers/staging/android/binder.c``  

```c++
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
   int ret;
   struct vm_struct *area;
   struct binder_proc *proc = filp->private_data;
   const char *failure_string;
   struct binder_buffer *buffer;

   if ((vma->vm_end - vma->vm_start) > SZ_4M)
      vma->vm_end = vma->vm_start + SZ_4M;
    ...

   if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
      ret = -EPERM;
      failure_string = "bad vm_flags";
      goto err_bad_arg;
   }
   vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;

   mutex_lock(&binder_mmap_lock);
   if (proc->buffer) {
      ret = -EBUSY;
      failure_string = "already mapped";
      goto err_already_mapped;
   }

   area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
   if (area == NULL) {
      ret = -ENOMEM;
      failure_string = "get_vm_area";
      goto err_get_vm_area_failed;
   }
   proc->buffer = area->addr;
   proc->user_buffer_offset = vma->vm_start - (uintptr_t)proc->buffer;
   mutex_unlock(&binder_mmap_lock);


   proc->pages = kzalloc(sizeof(proc->pages[0]) * ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
   if (proc->pages == NULL) {
      ret = -ENOMEM;
      failure_string = "alloc page array";
      goto err_alloc_pages_failed;
   }
   proc->buffer_size = vma->vm_end - vma->vm_start;

   vma->vm_ops = &binder_vm_ops;
   vma->vm_private_data = proc;

   buffer = proc->buffer;
   INIT_LIST_HEAD(&proc->buffers);
   list_add(&buffer->entry, &proc->buffers);
   buffer->free = 1;
   binder_insert_free_buffer(proc, buffer);
    //如果是异步访问，空间大小会除以2
   proc->free_async_space = proc->buffer_size / 2;
   barrier();
   proc->files = get_files_struct(proc->tsk);
   proc->vma = vma;
   proc->vma_vm_mm = vma->vm_mm;

...
   return ret;
}

```

进程打开了/dev/binder之后，就需要调用binder_mmap()函数把这个设备文件映射到所在的进程的地址空间，映射完成之后就可以为进程分配内核缓冲区了，以便它可以用来传输进程间通信的数据。

注意：内核层分配的空间限制是4M-8K，用户空间的限制是1M-8k，intent 传值也是通过进程间通信来完成的，所以数据的传递是有限制的，这个限制服从1M-8K，如果是异步需要除以2。

内核缓存区的分配操作通过binder_alloc_buf函数来实现的。



#### 8.通过mmap进行数据的读写

```c++
int8_t *m_ptr;
int32_t m_size;
int m_fd;

extern "C"
JNIEXPORT void JNICALL
Java_com_test_MainActivity_mmapWrite(JNIEnv *env, jobject thiz) {
    std::string file = "/sdcard/mmap.txt";
    //打开文件
    m_fd = open(file.c_str(), O_RDWR | O_CREAT, S_IRWXU);
    //获得一页内存大小
    //Linux采用了分页来管理内存，即内存的管理中，内存是以页为单位,一般的32位系统一页为 4096个字节
    m_size = getpagesize();
    //将文件设置为 m_size这么大
    ftruncate(m_fd, m_size);  
    // m_size:映射区的长度。 需要是整数页个字节    byte[]
    m_ptr = (int8_t *) mmap(0, m_size,
                            PROT_READ | PROT_WRITE, MAP_SHARED, m_fd,
                            0);

    std::string data("通过mmap内存映射写入文件的数据");
    //将 data 的 data.size() 个数据 拷贝到 m_ptr
    //Java 类似的：
//        byte[] src = new byte[10];
//        byte[] dst = new byte[10];
//        System.arraycopy(src, 0, dst, 0, src.length);
    memcpy(m_ptr, data.data(), data.size());

    __android_log_print(ANDROID_LOG_ERROR, "lpf", "写入数据:%s", data.c_str());
}

extern "C"
JNIEXPORT void JNICALL
Java_com_test_MainActivity_mmapRead(JNIEnv *env, jobject thiz) {
    //申请内存
    char *buf = static_cast<char *>(malloc(120));
    memcpy(buf, m_ptr, 120);

    std::string result(buf);
    __android_log_print(ANDROID_LOG_ERROR, "lpf", "读取数据:%s", result.c_str());

    //取消映射
    munmap(m_ptr, m_size);
    //关闭文件
    close(m_fd);
}
```

使用java的方式进行文件的写入，需要经过两次copy，首先需要将数据copy到内核区，然后将内核区的数据拷贝到磁盘。但是如果首先使用mmap进行内存映射的话，就只需要进行一次数据拷贝就可以将数据写入磁盘。



