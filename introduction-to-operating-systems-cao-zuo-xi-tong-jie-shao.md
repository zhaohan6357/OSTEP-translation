# Introduction to Operating Systems 操作系统介绍

如果你如果你正在上操作系统课程，你应该已经对计算机运行程序时发生了什么有所了解。如果没有，这本书（以及相应课程）可能对你来说有些困难——因此你应该停止阅读本书，去最近的书店购买相应的背景资料。\(此处作者推荐CSAPP\)

那么在程序运行时究竟发生了什么呢？

其实一个运行中的程序做的事情很简单：执行指令。处理器以每秒数百万次（甚至数十亿次）的速度从内存取指令，译码（得到具体指令功能），执行指令（就是实现其所指功能，比如将两个数相加、访问内存、检查条件，跳转到某一个函数等等）。一条指令执行完毕后，处理器会紧接着处理下一条，如此反复，直至程序完成（现代处理器为提高程序运行速度做出了很多努力，如一次执行多条指令，甚至是指令可以不按顺序完成，但此处只考虑最简单的情形，即一次执行一条指令，且按顺序执行）。

如上所述，我们描述的是冯诺依曼（Von Neumann）模型的基本状况。听起来很简单不是吗？但在这门课中，我们将会学习在程序执行过程中许多其他的手段，这些手段的目的统一，都是为了让系统更易于使用。

在计算机运行时有一个软件实体帮助我们更容易地执行程序（甚至让你能够在同一时间运行多个程序），还能让程序共享内存，使程序能够和设备交互，以及许许多多有趣的功能。这个软件实体就被称为操作系统，让整个程序系统正确并且有效率的执行是操作系统的职责所在。

操作系统通过一个通用的技术实现这些，我们称之为**虚拟化。**就是说，操作系统将物理资源（处理器，内存或磁盘）转换为一种更加通用有效且易于使用的虚拟形式，因此，我们有时也将操作系统称为**虚拟机**。

```text
问题关键：如何将资源虚拟化？

我们在本书中将要回答的一个中心问题很简单：操作系统如何虚拟化资源？这是关键问题。操作系统为什么要这么做
并不是主要问题，因为答案是显然的：为了让系统更易于使用。因此我们将聚焦于如何实现：操作系统采用了哪些机制
和策略来实现虚拟化？如何有效率地实现这些？以及需要哪些硬件支持。
```

最后，由于虚拟化使得许多程序同时运行（共享CPU），并且同时访问其各自的指令和数据（共享内存），同时访问设备（共享磁盘等等），因此操作系统又可以看作是**资源管理器。**CPU，内存，磁盘等都可以看作是系统资源，操作系统充当资源管理者的角色，使得系统运行的更为高效，公平，并完成特定功能。为了更好地理解操作系统的角色，下面来看几个例子。

```cpp
1 #include <stdio.h>
2 #include <stdlib.h>
3 #include <sys/time.h>
4 #include <assert.h>
5 #include "common.h"
6
7 int
8 main(int argc, char *argv[])
9 {
10     if (argc != 2) {
11         fprintf(stderr, "usage: cpu <string>\n");
12         exit(1);
13     }
14     char *str = argv[1];
15     while (1) {
16     Spin(1);
17     printf("%s\n", str);
18     }
19     return 0;
20 }

Figure 2.1: Simple Example: Code That Loops and Prints (cpu.c)
```

**2.1 CPU虚拟化**

图2.1是我们的第一个程序，它的功能很简单，主要功能是Spin\(\)函数，即不断地检查时间，满一秒即返回，随后输出用户在命令行传递的字符串，如此重复不断。

我们将此文件存作cpu.c，在单个处理器上编译运行，会得到如下结果：

```text
prompt> gcc -o cpu cpu.c -Wall
prompt> ./cpu "A"
A
A
A
A
ˆC
prompt>
```

运行结果不出所料，每秒钟输出一次用户传递的字符串（此处为"A"），值得注意的是，这个程序会永无休止地运行下去，因此我们使用Ctrl—C进行终止。

下面我们对这个程序使用一种不同的运行方式，如下图2.2（在tcsh shell中，使用&关键字可以将当前任务设置为后台任务，并且用分号连接可以实现同时运行多个任务）：

```text
prompt> ./cpu A & ; ./cpu B & ; ./cpu C & ; ./cpu D &
[1] 7353
[2] 7354
[3] 7355
[4] 7356
A
B
D
C
A
B
D
C
A
C
B
D
...
Figure 2.2: RunningMany Programs At Once
```

可以看出这时的情况很有意思，尽管我们只有一个处理器，但是好像多个程序在同时运行，到底发生了什么？

是操作系统，它凭借硬件的支持，造就了这样一种”假象“，一种系统具有很多虚拟CPU的假象。将一个或一组有限数量的CPU转化为看上去无限多的CPU，从而“表面上”使很多程序在同时运行，这种技术我们称之为CPU虚拟化，是本书内容的第一部分。

当然还需要一些应用程序接口（APIs）来向操作系统传达你的操作意向，比如运行和终止程序，或者告诉操作系统应该运行哪一个程序。我们在本书中也会讨论这些接口，这些接口也是人机交互的主要方式。

你或许也已经意识到，同时运行多道程序的能力也带来了一些问题，比如如果两个程序在某一时刻同时要运行，应该运行哪一个呢？这由操作系统的策略（policy）决定，操作系统在很多地方使用了一些策略来解决这类调度问题，当我们学习操作系统底层机制（mechanism）的时候也会学习这些策略（比如多道程序同时运行的能力）。从这个角度都说，操作系统可以看作是资源管理器。

```cpp
1 #include <unistd.h>
2 #include <stdio.h>
3 #include <stdlib.h>
4 #include "common.h"
5
6 int
7 main(int argc, char *argv[])
8 {
9      int *p = malloc(sizeof(int)); // a1
10     assert(p != NULL);
11     printf("(%d) memory address of p: %08x\n",
12     getpid(), (unsigned) p); // a2
13     *p = 0; // a3
14     while (1) {
15         Spin(1);
16         *p = *p + 1;
17     printf("(%d) p: %d\n", getpid(), *p); // a4
18     }
19     return 0;
20 }

Figure 2.3: A Program that Accesses Memory (mem.c)
```

**2.2内存虚拟化**

现在我们来考虑内存。现代机器的物理内存模型很简单。内存仅仅是一个字节序列。读取内存应该指明数据存储的地址，向内存写入时那个样也要指明将要写入的具体数据和写入地址。

程序运行时，内存时刻都在被访问。一个程序将其所有的数据结构都保存在内存中，并且通过不同的指令去访问，比如加载，存储等等。同时不要忘记指令同样存储在内存中，取指令的时候也是在访问内存。

看一下2.3中的程序，通过malloc\(\)函数分配内存，程序输出结果如下：

```text
prompt> ./mem
(2134) memory address of p: 00200000
(2134) p: 1
(2134) p: 2
(2134) p: 3
(2134) p: 4
(2134) p: 5
ˆC
```

这个程序首先分配了一些内存（a1行），然后将分配空间的地址输出（a2行），然后将数字0存储在新分配空间的首部，最后，循环，每间隔1秒将地址p中存储的值加1，在每个输出的语句中，还输出了正在执行的程序的进程号（PID），对于每个正在运行的程序，PID唯一。

当然，第一个运行的结果完全在意料之中，下面来看看如果同时运行这个程序的多个实例会发生什么：

```text
prompt> ./mem &; ./mem &
[1] 24113
[2] 24114
(24113) memory address of p: 00200000
(24114) memory address of p: 00200000
(24113) p: 1
(24114) p: 1
(24114) p: 2
(24113) p: 2
(24113) p: 3
(24114) p: 3
(24113) p: 4
(24114) p: 4
...
Figure 2.4: Running TheMemory ProgramMultiple Times
```

如图2.4所示，两个进程在相同的地址00200000分配了空间，但是却各自独立地更新存储在该地址的值，这看上去好像是每个进程都有独立的地址空间。

其实这就是操作系统实现的内存虚拟化，每个进程访问其私有的虚拟地址空间（或简称地址空间），操作系统负责将虚拟地址映射到具体的物理地址。一个进程访问其地址空间中的地址时并不影响其他进程（也包括操作系统本身），从每个进程的角度，好像其本身独占所有内存空间，然而事实是，物理内存是被所有程序共享的，由操作系统管理，如何管理内存也是本书第一部分讨论的内容：虚拟化。

**2.3并发**

本书的另一个重要的主题是**并发**，我们使用这个概念来描述一个程序在同时执行多项任务时所发生的必须被处理的问题。操作系统本身也会产生并发的问题，正如同我们在上述虚拟化的例子中看到的，操作系统需要同时应付许多事情，首先执行一个进程，紧接着下一个，如此类推，因此这种情形下会产生许多有趣的问题。

```text
1 #include <stdio.h>
2 #include <stdlib.h>
3 #include "common.h"
4
5 volatile int counter = 0;
6 int loops;
7
8 void *worker(void *arg) {
9     int i;
10     for (i = 0; i < loops; i++) {
11         counter++;
12     }
13     return NULL;
14 }
15
16 int
17 main(int argc, char *argv[])
18 {
19     if (argc != 2) {
20     fprintf(stderr, "usage: threads <value>\n");
21     exit(1);
22     }
23     loops = atoi(argv[1]);
24     pthread_t p1, p2;
25     printf("Initial value : %d\n", counter);
26
27     Pthread_create(&p1, NULL, worker, NULL);
28     Pthread_create(&p2, NULL, worker, NULL);
29     Pthread_join(p1, NULL);
30     Pthread_join(p2, NULL);
31     printf("Final value : %d\n", counter);
32     return 0;
33 }

Figure 2.5: AMulti-threaded Program (threads.c)
```

并发性的问题也不仅仅存在于操作系统本身中，实际上，现在流行的**多线程程序**也存在同样的问题，如图2.5就是一个多线程的例子。

或许你现在可能还不能完全理解这个例子，在本书后续部分会详细讨论。这个程序通用Pthread.creat\(\)（实际调用的是小写形式pthread.creat\(\)，首字母大写的函数是一种封装后的形式，比如封装了异常处理等，在本书后面有许多类似的用法，这应该也是一种优良的编码风格，值得学习）。你可以将线程看作是和其他函数运行在同一内存空间内的一种函数，在同一时刻有多个线程处于活动状态，在这个例子中，每个线程都运行了一个worker\(\)函数，该函数的功能是循环并且使用counter变量记录循环的次数。

下面是我们将loops变量设置为1000时的运行结果，loops变量决定了两个线程中各自执行循环并增加counter变量的次数，那么当loops为1000时，你期望最终counter的值是多少呢？

```text
prompt> gcc -o thread thread.c -Wall -pthread
prompt> ./thread 1000
Initial value : 0
Final value : 2000
```

你可能已经猜到，最终值是2000，因为两个线程各自都执行了1000次counter++操作。实际上，当我们将loops设置为N时，我们会默认最终的counter值应该是2N，然而事实果真如此？其实事情并不简单，让我们看看当把loops值设置大些时会发生什么：

```text
prompt> ./thread 100000
Initial value : 0
Final value : 143012 // huh?? 蛤？
prompt> ./thread 100000
Initial value : 0
Final value : 137298 // what the?? 什么鬼？？
```

当loops值设置为100000时，得到的结果不是期望中的200000，而是143012，更奇怪的是当我们重新执行一遍时居然又得到了另外一个结果，当你不断执行这个程序，你会得到许多不同的结果，或许偶尔会有你期望的答案200000，但是，为何如此？究竟发生了什么？

实际上，这种奇怪的输出结果和指令的执行过程有关，如前假设，计算机一次执行一条指令

，而counter++语句虽然只是一条语句，但却包含三条指令：一条将counter的值从内存加载到寄存器；一条将寄存器中的counter值增加1；最后一条将其从寄存器转存到内存中。由于这三条指令并不是原子化地执行（所谓原子化是指所有指令紧接着执行，中间不被其他指令打断），从而发生了这些奇怪的现象，我们将在本书的第二部分详细讨论。

```text
问题的关键：如果构建正确的并发程序？
当多个线程在同一内存空间内并发执行时，如何构建正确的程序？操作系统需要提供哪些原语支持？（原语，
primitives，是由若干条指令组成的,用于完成一定功能的一个过2程）硬件方面需要提供哪些机制？我们如何
使用这些来解决并发问题？
```

**2.4 持久化**

第三个重要的主题就是**持久。**由于在动态随机存储器（DRAM）中数据具有不稳定性，因此在系统内存中，数据具有易失性；因此当断电或系统崩溃时，内存中的数据会丢失。从而我们需要软件和硬件配合来存储数据，并使数据具有持久性，数据至上，因此这种存储对任何系统来说都很重要。

其中硬件以I/O设备的形式展现；在现代系统中，（磁头式）硬盘是常用的长时间存储设备，尽管目前固态硬盘也发展的很快。

操作系统中管理磁盘的软件称作**文件系统；**它负责将用户创建的文件可靠且高效地存储到磁盘上。

不像CPU和内存那样虚拟化的机制，操作系统并不为每一个应用创建私有且虚拟的磁盘。它假设用户经常需要在文件之间共享信息，比如当写一个C程序时，需要一个编辑器来创建和编辑源文件，然后需要编译器将源代码变成可执行文件。当一切完成可以使用命令来运行该程序。可以看出文件在各个过程中共享，编辑器创建的文件作为编译器的输入，编译器产生的可执行文件提供给系统运行。

为了更好地理解，下面来看一段代码，如图2.6，该代码实现的功能是创建了一个文件\(/temp/file\)，文件中包含了字符串"hello world"：

```text
1 #include <stdio.h>
2 #include <unistd.h>
3 #include <assert.h>
4 #include <fcntl.h>
5 #include <sys/types.h>
6
7 int
8 main(int argc, char *argv[])
9 {
10     int fd = open("/tmp/file", O_WRONLY | O_CREAT | O_TRUNC, S_IRWXU);
11     assert(fd > -1);
12     int rc = write(fd, "hello world\n", 13);
13     assert(rc == 13);
14     close(fd);
15     return 0;
16 }
Figure 2.6: A Program That Does I/O (io.c)
```

为了完成这一任务，该程序进行了三次系统调用，首先使用open\(\)，打开并创建文件，然后使用write\(\)向文件中写入数据，最后使用close\(\)关闭文件表明系统不会再写入任何数据。操作中和这些系统调用关联的部分称为文件系统，文件系统负责处理这些请求并返回结果或错误代码给用户。

你可能会疑惑操作系统在向磁盘中写入时究竟做了什么，这其实有些复杂：首先文件系统会算出数据在磁盘上将要存放的位置，然后用文件系统中不同的数据结构记录下来。这些需要对底层存储设备发起I/O请求。任何写过设备驱动的人都知道，让具体的硬件设备完成特定的任务是一项负责且充满细节的工作，它需要对底层硬件设备和其运行机理有着深刻的理解。所幸的是，操作系统通过系统调用为我们操纵硬件设备提供了简明标准的方式，从这个层面，操作系统可以看作是一种标准库。

当然，对于硬件设备如何访问以及文件系统如何保持数据持久化方面有着许许多多的细节。从性能方面说，大多数文件系统对于写操作采取延迟处理策略，以期通过批处理的方法提高效率。为了处理写入时系统崩溃的问题，许多文件系统采取了一系列复杂的写入协议，比如日志（journaling）或写时复制（copy—on-write），通过谨慎地处理写入的顺序从而在写入过程中发生系统崩溃时能够正确的恢复而保证文件的完整性和一致性。为了提高操作效率，文件系统还采用了许多数据结构和访问方法，从简单的链表到复杂的B-树均有所涉及。我们将在本书的第三部分详细讨论这些，包括设备、I/O、以及磁盘、RAID，最后到文件系统。

```text
问题的关键：如何持久化存储数据
文件系统是操作系统中负责管理持久化数据的部分。做到这些需要哪些技术呢？要想达到高性能需要哪些机制
和策略？在软件和硬件都有可能崩溃的情况下，如果保证数据的可靠性？
```

**2.5 设计目标**

现在你应该已经理解了一些操作系统究竟做了些什么：将物理资源，如CPU、内存、磁盘等虚拟化；处理和并发有关等复杂而有挑战的问题；还要持久性地保存文件，保证其安全长效。如果我们想构建这样一个系统，我们需要设立一些目标来帮助我们集中设计和实现，并且在必要时做出适当的**权衡**；构建一个系统最终要就是在各项目标之间权衡。

首先最基本的目标就是设立一些**抽象**来使得系统更加易于使用。抽象是计算机科学中的核心概念，抽象使得构建大型程序成为可能。通过抽象，我们将大型程序分解为小且易于理解的程序片；通过抽象，我们可以使用高级语言编写程序而不用考虑汇编细节；通过抽象，我们可以使用汇编语言而不用考虑逻辑门；通过抽象，我们可以通过逻辑门设计处理器而不用考虑晶体管的原理，如此等等，抽象为我们提供了一种将复杂问题简单化层次化的方法，在计算机科学中极为重要，在操作系统的学习中更要深刻体会这个概念。

<<<<<<< HEAD
另一个目标就是要使得操作系统具有**高性能**，换句话说，要使得操作**系统的开销**最小化。虚拟化是一项方便且值得去实现的技术，但并不是不惜一切地去实现，因此我们必须努力使系统在实现虚拟化的同时不产生过大的开销，开销体现在各种各样的方面：额外的时间（过多的指令），额外的空间（内存或磁盘）等。我们将采取各种方式以降低时间空间复杂性。我们还必须明白，完美的解决方案未必存在，我们要学会让步和折中。

<<<<<<< HEAD
还有一个目标是提供应用程序之间以及操作系统和应用程序之间的保护机制。由于我们希望多道程序同时运行,因此我们需要一个程序的崩溃和错误不影响其他程序，我们更不希望某个程序能够损坏操作系统。因此，保护机                                                                                                                                                               制是设计一个操作系统的主要原则，即隔离机制：将进程之间相互隔离是操作系统必须实现的任务。

 
=======
另一个目标就是要使得操作系统具有**高性能**，换句话说，要使得操作**系统的开销**最小化。虚拟化是一项方便且值得去实现的技术，但并不是不惜一切地去实现，因此我们必须努力使系统在实现虚拟化的同时不产生过大的开销，开销体现在各种各样的方面：额外的时间（过多的指令），额外的空间（内存或磁盘）等。我们将采取各种方式以降低时间空间复杂性。我们还必须明白，完美的解决方案未必存在，我们要学会妥协和折中。
>>>>>>> 1cbee4b... Updates introduction-to-operating-systems-cao-zuo-xi-tong-jie-shao.md
=======
还有一个目标是提供应用程序之间以及操作系统和应用程序之间的**保护机制**。由于我们希望多道程序同时运行,因此我们需要一个程序的崩溃和错误不影响其他程序，我们更不希望某个程序能够损坏操作系统。因此，保护机制是设计一个操作系统的主要原则，即隔离机制：将进程之间相互隔离是操作系统必须实现的功能。
>>>>>>> 2164b8f... Updates introduction-to-operating-systems-cao-zuo-xi-tong-jie-shao.md
