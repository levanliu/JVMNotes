
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
    printf("Data: %s
", data);
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

#define SHARED_MEMORY_NAME "/my_shared_memory"
#define SHARED_MEMORY_SIZE 4096

int main() {
    int shm_fd;
    char *data;

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

    // Perform inter-process communication using shared memory
    sprintf(data, "Hello, shared memory!");

    // Wait for other process to complete its communication
    sleep(3);

    // Read the modified data from shared memory
    printf("Data: %s", data);

    // Unmap and close the shared memory object
    if (munmap(data, SHARED_MEMORY_SIZE) == -1) {
        perror("Failed to unmap shared memory");
        shm_unlink(SHARED_MEMORY_NAME);
        return -1;
    }

    // Remove the shared memory object from the system
    if (shm_unlink(SHARED_MEMORY_NAME) == -1) {
        perror("Failed to remove shared memory");
        return -1;
    }

    return 0;
}

```

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

    // Perform inter-process communication using shared memory
    sprintf(data, "Hello, shared memory!");

    // Signal the other process that communication is complete
    sem_post(semaphore);

    // Wait for the other process to complete its communication
    sem_wait(semaphore);

    // Read the modified data from shared memory
    printf("Data: %s
", data);

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
        printf("Child process: Data: %s
", data);

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
        sem_wait(semaphore);

        // Read the modified data from shared memory
        printf("Parent process: Data: %s
", data);
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