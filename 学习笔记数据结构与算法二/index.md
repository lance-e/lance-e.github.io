# 学习笔记：数据结构与算法(二)


<!--more-->

# 数据结构与算法入门(二)

### 第三章（续）

##### 静态链表（游标实现法）：

让数组的元素都是由两个数据域组成，data和cur。也就是 说，数组的每个下标都对应一个data和一个cur。数据域data，用来存 放数据元素，也就是通常我们要处理的数据；而cur相当于单链表中的 next指针，存放该元素的后继在数组中的下标，我们把cur叫做游标。

~~~c++
#define MAXSIZE 1000
typedef struct {
    ElemType data;
    int cur;//游标Cursor,为0表示无指向
}Component,
StaticLinkList[MAXSIZE];
//对于不提供结构struct的程序设计语言，可以使用一对并行数据data和cur来处理
~~~

初始化静态链表：

~~~c++
//初始化数组状态
Status InitList(StaticLinkList space){
    int i ;
    for (i = 0 ; i <MAXSIZE-1;i++){
        space[i].cur = i +1;
    }
    space[MAXSIZE -1 ].cur = 0;//最后一个元素的cur存放第一个有数值的元素的下标，相当于单链表中的头结点作用
    return OK;
}
~~~

###### 静态链表的插入操作：

为了辨明数组中哪些分量未被使用，解决的办法是将所有未被使用过 的及已被删除的分量用游标链成一个备用的链表，每当进行插入时， 便可以从备用链表上取得第一个结点作为待插入的新结点。

~~~~c++
int Malloc_SLL(StaticLinkList space ){
    //返回第一个备用空闲的下标
    int i = space[0].cur;
    if (space[0].cur){
        //因为把第一个空闲的下标返回，所以需要把下一个分量拿来备用
        space[0].cur = space[i].cur;
    }
    return i;
}
~~~~

具体实现：

~~~c++
//在第i个元素前插入新的数据元素e
Status ListInsert(StaticLinkList L , int i ,ElemType e){
    int j , k , l;
    //k是最后一个元素的下标
    k = MAXSIZE -1;
    if (i<1 || i > ListLength(L)+1){
        return ERROR;
    }
    j = Malloc_SLL(L);//返回空闲分量的游标
    if (j ){
        L[j].data = e;
        //循环找到第i个元素之前的位置
        for ( l = 1; l <i -1 ; l++){
            k = L[k].cur;//k = 第i-1个元素的游标
        }
        L[j].cur = L[k].cur;//让插入元素的cur指向第i个元素
        L[k].cur = j;//让第i-1个元素的cur指向新元素
        return OK;
    }
    return ERROR;
}
~~~

###### 静态链表的删除操作：

~~~c++
Status ListDelete(StaticLinkList L, int i ){
    int j , k ;
    if (i <1 || i >ListLength(L)){
        return ERROR;
    }
    k = MAXSIZE -1;
    for (j = 1;j<=i -1;j++){
        k = L[k].cur;
    }
    j = L[k].cur;
    L[k].cur = L[j].cur;
    Free_SSL(L,j);
    return OK;

}
//将下标为k的空闲结点回收到备用链表
Status Free_SSL(StaticLinkList space,int k ){
    //把备用链表的第一个元素的cur值赋给要删除的分量的cur
    space[k].cur =space[0].cur;
    //要删除的分量下标赋值给备用链表的第一个元素的cur
    space[0].cur = k;
}
~~~

###### 获取静态链表的元素个数：

~~~c++
//返回静态链表的数据元素的个数
int ListLength(StaticLinkList L){
    int j =0 ;
    int i = L[MAXSIZE -1 ].cur;
    while(i){
        i = L[i].cur;
        j++;
    }
    return j;
}
~~~

##### 循环链表：

将单链表中终端结点的指针端由空指针改为指向头结点，就使整个单 链表形成一个环，这种头尾相接的单链表称为单循环链表，简称循环链表（circular linked list）。

（不用头指针，而是使用指向终端结点的尾指针）

合并两个循环链表的操作：

~~~c++
/* 保存A表的头结点，即① */
p = rearA->next;
/*将本是指向B表的第一个结点（不是头结点） */
rearA->next = rearB->next->next;
/* 赋值给reaA->next，即② */
q = rearB->next;
/* 将原A表的头结点赋值给rearB->next，即③ */
rearB->next = p;
/* 释放q */
free(q);

~~~

##### 双向链表：

双向链表（double linkedlist）：是在单向链表的每个结点中，再设置以一个指向其前驱结点的指针域。所以在双向链表中的结点都有两个指针 域，一个指向直接后继，另一个指向直接前驱。

代码实现：

~~~c++
typedef struct DulNode{
    ElemType data;
    struct DulNode *prior;//直接前驱指针
    struct DulNode *next ;//直接后续指针
}DulNode,*DuLinkList;
~~~

### 第四章 栈与队列

##### 4.2.1栈的定义：

栈(stack)是限定仅在表尾进行插入和删除操作的线性表
我们把允许插入和删除的一端称为栈顶(top)，另一端称为栈底 (bottom)，不含任何数据元素的栈称为空栈。栈又称为后进先出 (LastIn First Out)的线性表，简称LIFO结构。

###### 栈的插入操作：也叫进栈，压栈，入栈（push）

###### 栈的删除操作：也叫出栈，弹栈（pop）

##### 4.3栈的抽象数据类型

~~~cpp
ADT 栈(stack) Data
同线性表。元素具有相同的类型，相邻元素具有前驱和后继关系。
Operation
    InitStack(*S): //初始化操作，建立一个空栈S。
    DestroyStack(*S): // 若栈存在，则销毁它。
    ClearStack(*S): //将栈清空。
    StackEmpty(S): // 若栈为空，返回true，否则返回false。
    GetTop(S, *e): //若栈存在且非空，用e返回S的栈顶元素。
    Push(*S, e): //若栈S存在，插入新元素e到栈S中并成为栈顶元素。
    Pop(*S, *e): // 删除栈S中栈顶元素，并用e返回其值。
    StackLength(S): // 返回栈S的元素个数。
endADT
~~~

##### 4.4栈的顺序存储结构及其实现

结构定义：

~~~cpp
//SElemType 视实际情况而定，这里假设为int
typedef int SElemType ;
const int MAXSIZE = 5;
typedef struct
{
    SElemType data[MAXSIZE];
    int top;//表示栈顶指针
}SqStack;
~~~

进栈操作(push):

~~~cpp
//push
Status Push (SqStack *S , SElemType e )
{
    //栈满
    if (S->top ==MAXSIZE-1){
        return ERROR;
    }
    //栈顶指针加一
    S->top ++;
    //将新插入的元素赋值到栈顶
    S->data[S->top]=e;
    return OK;
}
~~~

出栈操作(pop):

~~~cpp
//pop
//若栈不空，则删除S的栈顶元素，用e返回其值， 并返回OK;否则返回ERROR
Status Pop(SqStack *S,SElemType e)
{
    //空栈
    if (S ->top ==-1){
        return ERROR;
    }
    e=S->data[S->top];
    S->top--;
    return OK;
}
~~~

##### 4.5两栈共享空间

结构：

~~~cpp
//两栈共享空间
typedef struct
{
    SElemType data[MAXSIZE];
    int top1; //栈一栈顶指针
    int top2; //栈二栈顶指针
}SqDoubleStack;
~~~

插入操作：

~~~cpp
Status Push(SqDoubleStack *S,SElemType e,int StackNumber)
{
    //栈满了
    if (S->top1 +1 == S->top2)
    {
        return ERROR;
    }
    if (StackNumber == 1)
        S->data[++S->top1]=e;
    else if (StackNumber ==2)
        S->data[--S->top2]=e;
    return OK;
}
~~~

删除操作：

~~~cpp
//如果栈不为空，就用e返回删除的值，并且返回ok，失败则返回error
Status Pop(SqDoubleStack *S,SElemType *e,int StackNumber )
{
    if (StackNumber == 1){
        if (S->top1 ==-1)
        {
            //说明栈一为空
            return ERROR;
        }
        *e = S->data[S->top1--];

    }
    else if (StackNumber ==2){
        if (S->top2 ==-1)
        {
            //说明栈二为空
            return ERROR;
        }
        *e = S->data[S->top2++];
    }
    return OK;
}
~~~

##### 4.6 栈的链式存储结构及实现

栈的链式存储结构，简称链栈

~~~cpp
//链栈的结构定义
//链栈结点
typedef struct StackNode
{
    SElemType data;
    struct StackNode *next;
}StackNode,*LinkStackPtr;
//整个链栈
typedef struct LinkStack
{
    LinkStackPtr top;//栈顶指针
    int count ; //栈元素个数
}LinkStack;

~~~

进栈操作push：

~~~cpp
//进栈操作
//插入e为新元素到链栈的栈顶
Status Push(LinkStack *S,SElemType e)
{

    LinkStackPtr s = (LinkStackPtr)malloc(sizeof(StackNode));
    s->data = e;
    //接着将当前栈顶元素赋值给新结点的直接后续，
    s->next = S->top;
    //将新结点赋值给栈顶指针top
    S->top = s;
    S->count ++;
    return OK;
}
~~~

出栈操作Pop：

~~~cpp
//出栈操作
//若栈不为空，就删除栈顶元素并用e来返回删除的元素，
//假设变量p用来存储要删除的栈顶元素，将栈顶指针后移一位，再释放掉p
Status Pop(LinkStack *S,SElemType *e)
{
    LinkStackPtr p ;
    //判断栈是否为空
    if (StackEmpty(S)){
        return ERROR;
    }
    *e= S->top->data;
    //将栈顶结点指针赋值给p
    p = S->top;
    //使栈顶指针下移到后一位
    S->top = S->top->next;
    //释放结点p
    free(p);
    S->count--;
    return OK;
}
~~~

##### 4.7栈的应用：

###### 1.递归

递归函数：直接调用自己或通过一系列的调用语句间接地调用自己的函数。
递归过程退回的顺序是它前行顺序的逆序。在退回过程中，可能要执行某些动作，包括恢复在前行过程中存储起来的某些数据。
因此要实现这样的需求，编译器就是使用栈来实现递归

###### 2.四则运算表达式

1.后缀（逆波兰）表达式：所有的符号都要在运算数字的后面出现
2.后缀表达式运算结果：规则:从左到右遍历表达式的每个数字和符号，遇到是数字就进栈，遇到是符号，就将处于栈顶两个数字出栈，进行算，运算结果进栈，一直到最终获得结果
3.中缀表达式：标准的四则运算表达式
4.中缀表达式转换为后缀表达式的规则：从左到右遍历中缀表达式的每个数字和符号，若是数字就输出，即成为后缀表达式的一部分;若是符号，则判断其与栈顶符号的优先级，是右括号或优先级不高于栈顶符号(乘除优先加减)则栈顶元素依次出栈并输出，并将当前符号进栈，一直到最终输出后缀表达式为止。

##### 4.8队列的定义：

队列是只允许在一端进行插入操作，一端进行删除操作的线性表
(First in First out )简称FIFO，允许插入的一头叫做队尾，允许删除的一头叫做队头

##### 4.9队列的抽象数据类型

~~~cpp
ADT 队列(Queue) Data
//同线性表。元素具有相同的类型，相邻元素具有前驱和后继关系。
Operation
InitQueue(*Q);//初始化操作，建立一个空队列Q。 
DestroyQueue(*Q);//若队列Q存在，则销毁它。
ClearQueue(*Q);//将队列Q清空。
QueueEmpty(Q);// 若队列Q为空，返回true，否则返回false。
GetHead(Q, *e);//若队列Q存在且非空，用e返回队列Q的队头元素。
EnQueue(*Q, e);// 若队列Q存在，插入新元素e到队列Q中并成为队尾元素
DeQueue(*Q, *e);//删除队列Q中队头元素，并用e返回其值
QueueLength(Q);//返回队列Q的元素个数
endADT
~~~

##### 4.10循环队列

循序存储结构的不足:
入队列操作，就是在队尾添加一个元素，时间复杂度为O(1)
出队列操作，就是从线性表下标为0的位置出列，全部元素都要移动，时间复杂度为O(n)
当只有一个元素时，队尾和对头重叠，使得操作麻烦，所以引入两个指针，一个是front指向队头元素，一个是rear指向队尾元素
的**下一个**位置，这样当front和rear相等时，不是只剩一个元素，而是这是一个空队列
如果一个队列即在出列也在入列，使rear超出了线性表的最大元素个数，但是队列队头还有空闲，这样的现象称为“假溢出”

###### 循环队列定义：队头队尾相连的顺序存储结构

循环队列满的条件：(rear+1)%QueueSize==front
循环队列空的条件：rear = front
循环队列队列长度：(rear-front+QueueSize)%QueueSize
循环队列的结构定义:

~~~cpp
//QElemType的类型视实际情况而定，这里假设为int
typedef int QElemType ;

typedef struct 
{
    QElemType data[MAXSIZE];
    int front;
    int rear;//尾追针，指向队列的队尾元素的下一个位置
}SqQueue;
~~~

循环列表初始化操作：

~~~cpp
Status InitSqQueue(SqQueue *Q){
    Q->front=0;
    Q->rear=0;
    return OK;
}
~~~

获取循环列表的长度：

~~~cpp
int SqQueueLenth(SqQueue Q){
    return (Q.front-Q.rear+MAXSIZE)%MAXSIZE;
}
~~~

入队操作：

~~~cpp
//插入e到循环队列Q中
Status EnQueue(SqQueue *Q,QElemType e ){
    //先判断是否队列满
    if ((Q->rear +1 +MAXSIZE )%MAXSIZE == Q->front){
        return ERROR;
    }
    //将e赋值给队尾
    Q->data[Q->rear]=e;
    //rear后移一位，若到最后就移动到数组前面
    Q->rear = (Q->rear+1)%MAXSIZE;
    return OK;
}
~~~

出队操作：

~~~cpp
//删除对头元素，并用e表示该元素
Status DeQueue(SqQueue *Q,QElemType *e ){
    //判断是否队列空
    if (Q->front == Q->rear){
        return ERROR;
    }
    //将对头赋值给e
    *e = Q->data[Q->front];
    //将front后移一位，若到最后就移动到数组前面
    Q->front = (Q->front +1) %MAXSIZE;
    return OK;
}
~~~

##### 4.11链队列

队列的链式存储结构其实就是线性表的单链表，队头指针指向头结点，队尾指针指向终端结点
空队列时，front和rear都指向头结点
链队列的定义：

~~~cpp
//QElemType的类型视实际情况而定，这里假设为int
typedef int QElemType ;
//结点定义
typedef struct QNode
{
    QElemType data;
    struct QNode *next;
}QNode,*QueuePtr;
//链队列定义
typedef struct 
{
    //队头队尾指针
    QueuePtr front,rear;
}LinkQueue;
~~~

链队列的入队操作：

~~~cpp
//插入元素e到链队列的队尾
Status EnQueue(LinkQueue *Q,QElemType e){
    QueuePtr s  = (QueuePtr)malloc(sizeof(QNode));
    //分配失败
    if (!s ){
        exit(OVERFLOW);
    }
    //将e赋值给s
    s->data= e;
    s->next =NULL;
    //将拥有e的新结点s赋值给原队列后续
    Q->rear->next = s;
    //把当天s设置为队尾结点，rear指向s
    Q->rear = s;
    return OK;
}
~~~

链队列的出队操作：

~~~cpp
//若队列不为空，删除队头元素，并用e返回值
Status DeQueue(LinkQueue *Q,QElemType *e){
    QueuePtr p ;
    //判断队列是否为空
    if (Q->front == Q->rear ){
        return ERROR;
    }
    //将队头元素暂存在p中
    p = Q->front->next;
    //将要删除的元素的值赋值给e
    *e = p->data;
    //将原头结点后续的后续赋值给头结点的后续
    Q->front->next = p->next;
    //如果队头是队尾，就将队尾指针指向头结点
    if (Q->rear == p ){
        Q->rear = Q->front;
    }
    free(p);
    return OK;
}
~~~

##### 4.12 总结

栈：只允许在表尾进行插入和删除操作的线性表
队列：只允许在队尾插入元素，队头删除元素的线性表

### 第五章 串

##### 5.1串的定义：

串(string)：是指由一个或多个字符组成的有限序列
零个字符的串就是空串
子串就是该串中连续的元素组成的子序列，包含子串的串就是主串
子串的第一个字符在主串的序号就是子串在主串的位置

##### 5.2串的抽象数据类型

~~~cpp
ADT 串(string) Data
串中元素仅由一个字符组成，相邻元素具有前驱和后继关系。
Operation
StrAssign(T, *chars):
StrCopy(T, S):
ClearString(S):
StringEmpty(S): 
StrLength(S):
StrCompare(S, T):
Concat(T, S1, S2):
SubString(Sub, S, pos, len)
Index(S,T,pos):
Replace(S,T,V):
StrInsert(S,pos,T):
StrDelete(S,pos,len):
endADT
~~~

##### 5.3串的存储结构

1.串的顺序存储结构：由一组内存地址连续的存储单元进行存储串中的字符序列，一般是根据预定义的长度，为串分配一个定长存储区，一般为定长数组
2.串的链式存储结构：一个结点放一个字符或者多个字符

##### 5.4朴素的模式匹配算法

算法实现：

~~~cpp
//index
//假设S[0],和T[0]存储各字符串的长度
//返回T在S的第pos个字符之后的位置
int Index(string S,string T,int pos){
    //i用于S中当前位置下标，若pos不为1，则从pos位置开始
    int i  = pos;
    //j用于T中当前位置下标
    int j = 1;
    while(i <S[0] &&j <T[0])
    {
        //两字母相同则继续
        if (S[i] == T[j]){
            i ++;
            j++;
        }else {
            //不同就指针回退重新开始匹配
            /* i退回到上次匹配首位的下一位 */ 
            i = i - j + 2;
            /* j退回到子串T的首位 */
            j = 1;
        }
    }
    if (j == T[0]){
        return i -T[0];
    }else {
        return 0;
    }
}
~~~

##### 5.5 KMP模式匹配算法

一脸懵逼😳，单独写个博客补充吧😭


