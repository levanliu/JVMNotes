
### mmap的使用

1. 打开文件
2. 创建映射
3. 对映射进行读写操作
4. 关闭映射
5. 关闭文件

```cpp
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>

int main() {
    int fd;
    char *data;

    // 打开文件
    fd = open("example.txt", O_RDWR);

    // 创建映射
    data = (char *)mmap(NULL, sizeof(char) * 1024, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    
    // 对映射进行读写操作
    printf("Data: %s", data);
    sprintf(data, "Hello, mmap!");

    // 关闭映射
    munmap(data, sizeof(char) * 1024);

    // 关闭文件
    close(fd);

    return 0;
}
```

### 不使用mmap

1. 打开文件
2. 使用read 读取文件内容
3. 使用write 写入文件内容
4. 关闭文件

```cpp
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd;
    char buffer[1024];
    
    // 打开文件
    fd = open("example.txt", O_RDWR);
    if(fd == -1) {
        perror("Failed to open file");
        return -1;
    }
    
    // 读取文件内容
    ssize_t bytesRead = read(fd, buffer, sizeof(buffer));
    if(bytesRead == -1) {
        perror("Failed to read file");
        close(fd);
        return -1;
    }
    
    printf("Data: %s", buffer);
    
    // 写入文件内容
    const char* data = "Hello, read/write!";
    ssize_t bytesWritten = write(fd, data, strlen(data));
    if(bytesWritten == -1) {
        perror("Failed to write file");
        close(fd);
        return -1;
    }
    
    // 关闭文件
    close(fd);
    
    return 0;
}

```
### 使用mmap进行进程间共享

1. 创建共享内存对象
2. 设置共享内存对象的大小
3. 将共享内存对象映射到进程的虚拟地址空间
4. 进行进程间通信
5. 等待另一个进程完成通信
6. 从共享内存中读取数据
7. 解除映射
8. 关闭共享内存对象
9. 删除共享内存对象


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <semaphore.h>

#define SHARED_MEMORY_NAME "/my_shared_memory"
#define SHARED_MEMORY_SIZE 4096
#define SEMAPHORE_NAME "/my_semaphore"

int main() {
    int shm_fd;
    char *data;
    sem_t *semaphore;

    // Create and open shared memory object
    shm_fd = shm_open(SHARED_MEMORY_NAME, O_CREAT | O_RDWR, 0666);
    if (shm_fd == -1) {
        perror("Failed to create shared memory");
        return -1;
    }

    // Set the size of the shared memory object
    if (ftruncate(shm_fd, SHARED_MEMORY_SIZE) == -1) {
        perror("Failed to set shared memory size");
        shm_unlink(SHARED_MEMORY_NAME);
        return -1;
    }

    // Map the shared memory object into the virtual address space
    data = (char *)mmap(NULL, SHARED_MEMORY_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
    if (data == MAP_FAILED) {
        perror("Failed to map shared memory");
        shm_unlink(SHARED_MEMORY_NAME);
        return -1;
    }

    // Create or open the semaphore
    semaphore = sem_open(SEMAPHORE_NAME, O_CREAT, 0666, 0);
    if (semaphore == SEM_FAILED) {
        perror("Failed to create or open semaphore");
        munmap(data, SHARED_MEMORY_SIZE);
        shm_unlink(SHARED_MEMORY_NAME);
        return -1;
    }

    // Fork the process
    pid_t pid = fork();

    if (pid == -1) {
        perror("Failed to fork");
        return -1;
    } 

    // Child process
    if (pid == 0) {
        // Wait for the parent process to complete its communication
        sem_wait(semaphore);

        // Read the data from shared memory
        printf("Child process: Data: %s", data);

        // Modify the data in shared memory
        strcpy(data, "Modified data by child process");

        // Signal the parent process that communication is complete
        sem_post(semaphore);
    }
    // Parent process
     else {
        // Perform inter-process communication using shared memory
        sprintf(data, "Hello, shared memory!");

        // Signal the child process that communication is complete
        sem_post(semaphore);

       // Wait for the child process to complete its communication
        waitpid(pid, NULL, 0);
        // Wait for the child process to complete its communication
        sem_wait(semaphore);

        // Read the modified data from shared memory
        printf("Parent process: Data: %s", data);
    }

    // Unmap and close the shared memory object
    if (munmap(data, SHARED_MEMORY_SIZE) == -1) {
        perror("Failed to unmap shared memory");
        shm_unlink(SHARED_MEMORY_NAME);
        sem_close(semaphore);
        sem_unlink(SEMAPHORE_NAME);
        return -1;
    }

    // Close and unlink the semaphore
    sem_close(semaphore);
    sem_unlink(SEMAPHORE_NAME);

    // Remove the shared memory object from the system
    if (shm_unlink(SHARED_MEMORY_NAME) == -1) {
        perror("Failed to remove shared memory");
        return -1;
    }

    return 0;
}

```

CAP理论是分布式系统设计中的一个基本原则，它指出在一个分布式系统中，一致性（Consistency）、可用性（Availability）和分区容错性（Partition Tolerance）这三个目标是无法同时满足的。

在CAP理论中，P代表分区容错性（Partition Tolerance），即一个分布式系统要能够在各个节点之间进行通信，即使某些节点无法通信或者出现网络分区，仍然能够继续工作。

分布式系统在C（Consistency，一致性）和A（Availability，可用性）两个目标之间需要进行权衡。C意味着在系统中的所有节点上的数据副本保持一致，而A意味着系统能够在任何时间响应客户端的请求。

根据CAP理论，由于分区容错性已经是分布式系统设计的必要条件，所以在C和A之间只能选择其中一个，并且不同的应用场景可能会偏向其中之一。

对于CP（Consistency and Partition Tolerance）模型，系统在面对网络分区时，会牺牲可用性，即请求可能会受到延迟或者无法响应，但是系统保证了数据的一致性，适用于对数据一致性要求较高的应用场景，如金融系统或者电商系统。

对于AP（Availability and Partition Tolerance）模型，系统在面对网络分区时，仍然保留了可用性，即能够对客户端请求进行响应，但是无法保证数据的一致性，适用于对系统可用性要求较高的应用场景，如社交网络或者新闻网站。

根据实际应用场景，我们可以根据对一致性和可用性的需求进行权衡，并选择CP或者AP模型。例如，对于一个有严格事务要求的金融系统，可能更倾向于选择CP模型，而对于一个高并发的社交网络，可能更倾向于选择AP模型。