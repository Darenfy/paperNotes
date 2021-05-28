## Fuzzing Android System Services by Binder Call to Escalate Privilege

> author：Guang Gong
> conference： Black Hat USA 2015

### 攻击面
![](../Fuzzing%20Android%20System%20Services%20by%20Binder%20Call%20to%20Escalate%20Privilege/截屏2020-11-19%20下午4.55.42.png)

### 接口
- 顶层接口
```
service list

```
- 下一层接口
```
IXXXXService.h
```

### 执行流程
![](../Fuzzing%20Android%20System%20Services%20by%20Binder%20Call%20to%20Escalate%20Privilege/截屏2020-11-19%20下午5.29.58.png)

### 利用
**控制 binder 服务线程数目**
线程使用不同的 arena 分配地址

不同线程分配的小内存会存在不同的 chunk 中

线程很多时，无法控制通过 binder 调用所分配的内存地址

使用 attachBuffer 使 binder server 处于等待状态

**泄漏 heap 内容**
由于 ASLR 的存在，需要从 mediaserver 泄漏地址信息

je_malloc 相同大小的内存将会分配在邻接区域


**地址泄漏**
- 泄漏栈地址
查询 pthread_internal_t 结构
由于 NX 的存在，需要使用 ROP 绕过
- 泄漏共享数据库地址
- 泄漏堆地址
  - 查询被泄漏堆内容的堆节点

**绕过 SELinux加载 so**
**利用 surfaceflinger & system_server**
