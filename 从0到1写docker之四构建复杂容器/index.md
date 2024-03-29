# 从0到1写docker之四构建复杂容器


<!--more-->

# (四)构建复杂容器

##### 1.容器后台运行

增加detach标签，并不允许创建tty和detach同时存在

~~~go
if tty && detach{
	log.Println("the tty and detach can't exist at the same time")
	return
}
~~~

并且在Run命令中更改：

~~~go
if tty {
	cmd.Wait() //父进程等待子进程
}
~~~

- 因为cmd.Wait()是让父进程等待子进程，如果我们要实现让容器后台运行，那么就不需要父进程等待子进程，子进程此时就会被init进程控制

##### 2.保存容器信息

在创建容器的同时，将容器信息写入到宿主机的文件中，我们保存到/var/run/gocker目录中

~~~go
type ContainerInfo struct {
	Pid           string `json:"pid"`
	ContainerName string `json:"container_name"`
	ContainerId   string ` json:"container_id"`
	CreateAtTime  string `json:"create_at_time"`
	Command       string `json:"command"`
	Status        string `json:"status"`
}

var (
	RUNING              = "runing"
	STOP                = "stopped"
	EXIT                = "exited"
	DefaultInfoLocation = "/var/run/gocker/%s/"
	ConfigName          = "config.json"
)

func RecordContainerInformation(containerName string, containerPid int, command []string) (string, error) {
	id := randStringBytes(10)
	createTime := time.Now().Format("2006-01-02 15:04:05")
	if containerName == "" {
		containerName = id
	}
	cmd := strings.Join(command, " ")
	info := &ContainerInfo{
		Pid:           strconv.Itoa(containerPid),
		ContainerName: containerName,
		ContainerId:   id,
		CreateAtTime:  createTime,
		Status:        RUNING,
		Command:       cmd,
	}
	infoByte, err := json.Marshal(info)
	if err != nil {
		log.Println("can't marshal the information")
		return "", err
	}

	location := fmt.Sprintf(DefaultInfoLocation, containerName)
	log.Println("the location is ", location)
	_, err = os.Stat(location)
	if err != nil && !os.IsNotExist(err) {
		log.Println("the status of the config file can't judge")
		return "", err
	}
	if os.IsNotExist(err) {
		if err := os.MkdirAll(location, 0755); err != nil {
			return "", err
		}

	}
	file, err := os.Create(path.Join(location, ConfigName))
	if err != nil {
		log.Println("can't create the config file")
		return "", err
	}
	defer file.Close()

	_, err = file.Write(infoByte)
	if err != nil {
		log.Println("can't write the byte of information into the target file")
		return "", err
	}
	log.Println("record the informaiton successful!")
	return containerName, nil
}
func randStringBytes(n int) string {
	num := "01234567890123456789"
	by := make([]byte, n)
	r := rand.New(rand.NewSource(time.Now().Unix()))
	for i, _ := range by {
		by[i] = num[r.Intn(n)]
	}
	return string(by)
}

func DeleteContainerInfo(containerName string){
	location := fmt.Sprintf(DefaultInfoLocation,containerName)
	dirPath := location+ConfigName
	if err := os.RemoveAll(dirPath);err != nil{
		log.Println("can't delete the config file")
	}

}
~~~

##### 3.实现ps命令

ps命令就是去前面保存容器信息的目录中遍历所有的文件，拿到所有的信息

~~~go
func ListContainers() {
	dir := fmt.Sprintf(DefaultInfoLocation, "")
	dir = dir[:len(dir)-1]
	files, err := os.ReadDir(dir)
	if err != nil {
		log.Println("can't read the all directory")
		return

	}
	var containers []*ContainerInfo
	for _, file := range files {
		info, _ := file.Info()
		data, err := readInfoFromFile(info)
		if err != nil {
			log.Println("read container information failed")
			continue
		}
		containers = append(containers, data)
	}
	w:=tabwriter.NewWriter(os.Stdout,12,1,3,' ',0)
	fmt.Fprintf(w,"Container_Id\tContainer_Name\tPid\tStatus\tCommand\tCreate_At_Time\n")
	for _,con := range containers {
		fmt.Fprintf(w,"%s\t%s\t%s\t%s\t%s\t%s\t\n",con.ContainerId,con.ContainerName,con.Pid,con.Status,con.Command,con.CreateAtTime,
	)
	}
	if err := w.Flush();err != nil{
		log.Println("can't flush to stdout")
		return
	}
}
func readInfoFromFile(file os.FileInfo) (*ContainerInfo, error) {
	containerName := file.Name()
	dir := fmt.Sprintf(DefaultInfoLocation, containerName)
	dir = dir + ConfigName
	info, err := os.ReadFile(dir)
	if err != nil {
		log.Println("can't read the file")
		return nil, err
	}
	var data = &ContainerInfo{}
	err = json.Unmarshal(info, data)
	if err != nil {
		log.Println("can't unmarshal the information ")
		return nil, err
	}
	return data, nil
}
~~~

##### 4.实现logs命令

当使用了-d标签，此时的后台运行的容器，我们是无法知道运行情况的，所以就需要logs来记录下后台运行容器的标准输出

需要先将detach容器的标准输出流定向到log文件中，在创建父进程的函数newparentProcess中：

~~~go
if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	} else {
		dir := fmt.Sprintf(DefaultInfoLocation,containerName)
		if err := os.MkdirAll(dir,0622);err!= nil{
			log.Println("can't make all directory")
			return nil ,nil
		}
		 file,err := os.Create(dir+ContainerLogFile)
		 if err != nil{
			log.Println("can't create the log file")
			return nil, nil
		}
		cmd.Stdout=file
	}
~~~

然后新建一个logs命令，命令的执行函数：

~~~go
var (
	ContainerLogFile = "container.logs"
)
func ShowLogs(containerName string){
	dir := fmt.Sprintf(DefaultInfoLocation,containerName)
	path := dir + ContainerLogFile
	file ,err := os.Open(path)
	if err != nil{
		log.Println("can't open the logs file")
		return
	}
	defer file.Close()
	data,err := io.ReadAll(file)
	if err != nil{
		log.Println("can't read from thelogs file")
		return
	}
	fmt.Fprint(os.Stdout,string(data))
}
~~~

同样也是依靠读取相应文件，获取到输出数据

##### 5.实现exec命令

go语言本身的限制，使得我们如果仅用go是无法实现进入指定namespace，所以我们需要使用到cgo

~~~go
package setns
/*
#define _GNU_SOURCE
#include "errno.h"
#include "string.h"
#include "stdlib.h"
#include "stdio.h"
#include "sched.h"
#include "fcntl.h"
#include "unistd.h"
//__attribute__((constructor)) 相当于init，让函数提前运行
__attribute__((constructor))  void enter_namespace(void ){
    char *pid;
    pid  = getenv("gocker_pid");
    if (pid){
        fprintf(stdout,"get the gocker_pid %s\n",pid);
    }else {
        fprintf(stdout,"get the gocker_pid failed\n");
        return;
    }
    char * cmd;
    cmd = getenv("gocker_cmd");
    if (cmd){
        fprintf(stdout,"get the gocker_cmd %s\n",cmd);
    }else {
        fprintf(stdout,"get the gocker_cmd failed\n");
        return;
    }
    int i ;
    char nspath [1024];
    char *namespace []={"pid","net","ipc","uts","mnt"};
    for (i =0;i<5;i++){
        sprintf(nspath,"/proc/%s/ns/%s",pid,namespace[i]);
        int fd =open(nspath,O_RDONLY);
        if( (setns(fd,0))== -1){
            fprintf(stderr,"setns on %s namespace failed,error:%s\n",namespace[i],strerror(errno));
        }else {
            fprintf(stdout,"setns on %s namespace successful\n",namespace[i]);
        }
        close(fd);
    }
    int res = system(cmd);
    exit(0);
    return;
    
}
*/
import "C"
~~~

__attribute__((constructor)) 会使这段c代码会在所有的go代码前执行，所以为了避免影响之前的项目，让c代码通过获取环境变量的方式，来限制执行的时机，仅让exec命令部分加入指定环境变量。

**要注意这里的namespace执行顺序，mnt namespace应该再最后执行**

在exec命令的执行过程中，就比较复杂，需要让exec命令执行两次，第一次设置环境变量，然后调用自己，fork一个新的进程，此时的环境变量已经设置，执行cgo代码，进行setns系统调用

~~~go
const ENV_EXEC_PID = "gocker_pid"
const ENV_EXEC_CMD = "gocker_cmd"

func ExecContainer(containerName string, command []string) {
	pid, err := getPidByContainerName(containerName)
	if err != nil {
		log.Println("can't get pid by container name")
		return
	}
	cmdstr := strings.Join(command, " ")
	log.Printf("pid :%s,cmd :%s\n", pid, cmdstr)
	cmd := exec.Command("/proc/self/exe", "exec")
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	os.Setenv(ENV_EXEC_PID, pid)
	os.Setenv(ENV_EXEC_CMD, cmdstr)
	cmd.Env = append( os.Environ(),(pid))
	cmd.Dir = "/root/overlay/merged"

	if err := cmd.Run(); err != nil {
		log.Println("exec container failed, error:", err)
	}
}
func getPidByContainerName(name string) (string, error) {
	dir := fmt.Sprintf(DefaultInfoLocation, name)
	allDir := dir + ConfigName
	data, err := os.ReadFile(allDir)
	if err != nil {
		log.Println("can't read the config informaiton from the file")
		return "", err
	}
	var info = &ContainerInfo{}
	err = json.Unmarshal(data, info)
	if err != nil {
		log.Println("unmalshal the information failed")
		return "", err
	}
	return info.Pid, nil
}
func getEnvByPid(pid string)[]string {
	dir := fmt.Sprintf("/proc/%s/environ", pid)
	data, err := os.ReadFile(dir)
	if err != nil {
		log.Println("can't read the environ file")
		return nil
	}
	return strings.Split(string(data), "\u0000")

}

~~~

这里的exec.Command仅仅只是再次调用了自己这个进程，并执行exec命令，不需要再进行其他的namespace隔离。

这里还要主要需要c代码所在的包导入，这样才能执行cgo

~~~go
	_"gocker/setns"
~~~

##### 6.实现停止容器

- 1.通过容器获取容器信息
- 2.发送kill信号
- 3.修改容器信息
- 4.将信息写入config文件

~~~go
func StopContainer(containerName string) {
	info, err := getContainerInfoByName(containerName)
	if err != nil {
		log.Println("can't get contaienr information by container name")
		return
	}
	pid, _ := strconv.Atoi(info.Pid)
	if err = syscall.Kill(pid, syscall.SIGTERM); err != nil {
		log.Println("can't send the kill sigt,error:", err)
		return
	}
	info.Status = STOP
	info.Pid = ""

	data, err := json.Marshal(info)
	if err != nil {
		log.Println("can't marshal the information ")
		return
	}

	dir := fmt.Sprintf(DefaultInfoLocation, containerName)
	allDir := dir + ConfigName
	if err = os.WriteFile(allDir, data, 0622);err != nil{
		log.Println("can't write the stoped container's information into the config file")
		return
	}
	log.Printf("%s stopping ...\n",info.ContainerName)
	
}
~~~

##### 7.实现删除容器

- 1.通过容器名获取信息
- 2.判断容器是否停止
- 3.删除容器的全部文件

~~~go
func RemoveContainer(contaienrName string){
	info,err := getContainerInfoByName(contaienrName)
	if err != nil{
		log.Println("can't get the information ,error:",err)
		return
	}
	if info.Status != STOP{
		log.Println("can't remove the running container")
		return
	}
	
	dir := fmt.Sprintf(DefaultInfoLocation, contaienrName)
	if err = os.RemoveAll(dir);err != nil{
		log.Println("can't remove all the file and directory")
		return
	}
	log.Printf("remove %s successful",info.ContainerName)

}
~~~

##### 8.实现通过容器创建镜像

现在我们的所有创建的容器都是挂载的同一个overlayFS，容器之间都会相互影响，所以我们需要给每个容器单独的文件系统
更改的地方较多，这里我们不进行更改部分的展示，基本都是在原来的overlayFS的基础上：

- 将每个容器对应的文件系统路径中增加容器名，例如/root/overlay/container1/merged
- 将镜像层从原本的文件系统中抽离，实现镜像与容器的解耦， 删除文件系统就可以直接将整个目录删除，删除更彻底
- 单独增加镜像层目录存放位置

##### 9.实现上传环境变量

增加了-e标签，来上传容器的环境变量
newparentProcess函数中增加的：

~~~go
cmd.Env = append(os.Environ(),environment... )
~~~

os.Environ()为默认的环境变量，创建的容器也需要继承宿主机上的环境变量

在后台容器的模式下，exec进程的环境变量继承了父进程的环境变量，所以直接在environ文件中获取

~~~go
func ExecContainer(containerName string, command []string) {
...
cmd.Env = append( os.Environ(),getEnvByPid(pid)...)
...
}

func getEnvByPid(pid string)[]string {
	dir := fmt.Sprintf("/proc/%s/environ", pid)
	data, err := os.ReadFile(dir)
	if err != nil {
		log.Println("can't read the environ file")
		return nil
	}
	return strings.Split(string(data), "\u0000")

}
~~~

