# 从0到1写docker之二构建简易容器


<!--more-->

# (二)创建简易容器

### 实现namespace和cgroup

通过实现一个简易的run命令，来构建容器
run命令的实现流程：

- 通过newparentProcess函数，构建一个父进程，此时已经进行了namespace
- 使用cgroup来对资源的限制，此时容器就已经创建完毕
- 创建init子进程(容器内第一个进程)，mount到/proc文件系统(方便ps命令)，同时使用syscall.Exec来覆盖之前的进程信息，堆栈信息(保证第一个进程是我们规定的进程)。

在容器中简单实现namespace和cgroup：
namespace的实现之间进行系统调用：

~~~go
cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWNS |
			syscall.CLONE_NEWIPC |
			syscall.CLONE_NEWPID |
			syscall.CLONE_NEWNET |
			syscall.CLONE_NEWUTS,
		//syscall.CLONE_NEWUSER,
	}
~~~

我们这里暂时不实现user namespace，因为较复杂会牵涉到权限等问题
cgroup则是通过读取/proc/self/mountinfo文件，获取当前进程的mount 情况，再根据我们需要限制的subsystem来获取到cgroup的挂载点，例如：/sys/fs/cgroup/memory，此时的subsystem为memory。
(本项目中仅支持了memory,cpuset ,cpushare进行了资源限制，本质上都是对文件进行读写操作)

~~~go
// FindCgroupMountPoint 获取cgroup挂载点
func FindCgroupMountPoint(subsystem string) string {
	file, err := os.Open("/proc/self/mountinfo")
	if err != nil {
		log.Println("can't open /proc/self/mountinfo")
		return "error :can't open file"
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		txt := scanner.Text()
		fileds := strings.Split(txt, " ")
		for _, opt := range strings.Split(fileds[len(fileds)-1], ",") {
			if opt == subsystem {
				log.Println("the mount point : ",fileds[4])
				return fileds[4]
			}
		}
	}
	if err := scanner.Err(); err != nil {
		return ""
	}
	return ""
}
// GetCgroupAbsolutePath 获取cgroup绝对路径
// 找到对应subsystem挂载的 hierarchy 相对路径对应的 cgroup 在虚拟文件系统中的路径,
// 然后通过这个目录的读写去操作 cgroup
// Cgroups 的 hierarchy 的虚拟文件系统是通过 cgroup类型文件系统的 mount挂载上去的,
// option 中加上 subsystem，代表挂载的 subsystem 类型 ,
// 这样就可以在 mountinfo 中找到对应的 subsystem 的挂载目录了，比如 memory。
func GetCgroupAbsolutePath(subsys string, cgroupPath string, autoCreate bool) (string, error) {
	 cgroupRoot := FindCgroupMountPoint(subsys)
	if _, err := os.Stat(path.Join(cgroupRoot, cgroupPath)); err == nil || (autoCreate && os.IsNotExist(err)) {
		if os.IsNotExist(err) {
			err := os.Mkdir(path.Join(cgroupRoot, cgroupPath), 0755)
			if err != nil {
				return "", errors.New("cgroup path error :" + err.Error())
			}
			return path.Join(cgroupRoot, cgroupPath), nil
		}
		return path.Join(cgroupRoot, cgroupPath), nil

	} else {
		return "", errors.New("cgroup path error :" + err.Error())
	}

}
~~~

找到了cgroup挂载的绝对路径，就可以通过操作文件来进行资源限制,这里以memory为例：
通过set方法，将设置的内存资源限制写入memory.limit_in_bytes文件中；

通过apply方法，把pid写入到tasks，目标进程加入到该cgroup中(！！！这里必须要注意，如果你在set中写入的数据格式不争取，是无法将pid写入tasks的)；

通过remove方法，则是取消该cgroup

~~~go
// 设置对应的cgroup memory限制
func (m *MemorySubsystem) Set(cgroupPath string, resource *ResouceConfig) error {
	//获取subsystem路径(cgroup绝对路径)
	absolutePath, err := GetCgroupAbsolutePath(m.Name(), cgroupPath, true)
	if err != nil {
		return err
	}
	//设置内存限制
	if resource.MemoryLimit != ""{
		err = os.WriteFile(path.Join(absolutePath, "memory.limit_in_bytes"), []byte(resource.MemoryLimit), 0644)
		if err != nil {
			return errors.New("set memory failed,error:" + err.Error())
		}
	}
	
	return nil

}
func (m *MemorySubsystem) Apply(cgroupPath string, pid int) error {
	absolutePath,err := GetCgroupAbsolutePath(m.Name(),cgroupPath,false)
	if err != nil{
		return errors.New("apply cgroup faield ,error:"+err.Error())
	}
	err = os.WriteFile(path.Join(absolutePath, "tasks"), []byte(strconv.Itoa(pid)), 0644)
	if err != nil {
		return errors.New("apply cgroup faield ,error:" + err.Error())
	}
	return nil

}
func (m *MemorySubsystem) Remove(cgroupPath string) error {
	absolutePath ,err := GetCgroupAbsolutePath(m.Name(),cgroupPath,false)
	if err != nil{
		return err
	}
	err = os.Remove(absolutePath)
	if err != nil{
		return err
	}
	return nil
}
~~~

## 参考文档

《自己动手写docker》

https://tech.meituan.com/2015/03/31/cgroups.html

https://www.cnblogs.com/charlieroro/p/10281469.html

https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/resource_management_guide/index

