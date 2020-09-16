# command.c
```
#include "common.h"

int do_connect(char *ip,struct sockaddr_in *sin, int *sock_fd)
{
	int sfd; 

	bzero(&sin, sizeof(struct sockaddr_in)); 
	sin->sin_family = AF_INET; 

	if(inet_pton(AF_INET, ip, &sin->sin_addr) == -1){
		perror("wrong format of ip address");
		return -1;
	}
    
	sin->sin_port = htons(PORT);
    	if((sfd = socket(AF_INET, SOCK_STREAM, 0)) == -1){ 
		perror("fail to creat socket");
		return -1;	
	}
	
	
    	if(connect(sfd, (struct sockaddr *)sin, sizeof(struct sockaddr_in)) == -1){
        	perror("fail to connect");
	   	return -1;
    	}
	
	*sock_fd = sfd; 
	
	return 0;
}

int do_get(const char *src, const char *dst, int sock_fd)
{
	char *dst_file; 
	char *p;
	struct stat statbuf; 
	int n, fd;
	char buf[MAX_LINE]; 
	int len;
	int res = -1; 
	
	if(src == NULL || dst == NULL){ 
		printf("wrong command\n"); 
		return -1;
	}
	
	if(src[strlen(src)-1]=='/'){
		printf("source file should be a regular file\n");
		return -1;
	}
	
   	if( (dst_file = (char *)malloc(strlen(dst) + strlen(src))) == NULL){ 
		perror("fail to malloc");
		return -1; 
	}
    
	strcpy(dst_file, dst); 
	if(dst_file[strlen(dst_file) - 1] != '/')
		strcat(dst_file, "/");
	p = rindex(src, '/'); 
    	strcat(dst_file, p + 1);  
	    	
	if((fd = open(dst_file, O_WRONLY | O_CREAT | O_TRUNC, 0644)) == -1){
	  	perror("fail to open dst-file");
	  	goto end2;
    	}
 
    	if(fstat(fd, &statbuf) == -1){ /* 取目標檔案的檔案狀態 */
		perror("fail to stat dst-file");
	  	goto end1;
	}
	
	if(!S_ISREG(statbuf.st_mode)){
	  	printf("dst-file should be a regular file");
	  	goto end1;
	}

    	sprintf(buf, "GET %s", src); 
    	
	if(my_write(sock_fd, buf, strlen(buf)+1) == -1) 
		goto end1;
    	
	if( (n = my_read(sock_fd, buf, MAX_LINE)) <= 0){ 
		goto end1;
    	}
    	
	if(buf[0] == 'E'){ 
		write(STDOUT_FILENO, buf, n); 
	  	res = 0;
		goto end1;
    	}
    
    	len = atoi(&buf[3]);

	if(my_write(sock_fd, "RDY", 3) == -1)
		goto end1;
   
	while(1){
	 	n = my_read(sock_fd, buf, MAX_LINE);
	  	
		if(n > 0){
	    		write(fd, buf, n);
	       	len -= n;
	  	}else if(len == 0){
			printf("OK\n"); 
	       		break;
	  	}else 
			goto end1;
	}

	res = 0; 
end1:
    	free(dst_file); 
end2:       
	close(fd); 

	return res;
}

int do_serv_cd(char *path, int sock_fd)
{
	char buf[MAX_LINE];
	int n;
	
	sprintf(buf, "CD %s", path); 
     
	if(my_write(sock_fd, buf, strlen(buf)+1) == -1) 
		return -1;
     
	if( (n = my_read(sock_fd, buf, MAX_LINE)) <= 0) 
		return -1;
     
     	if(buf[0] == 'E') 
		write(STDOUT_FILENO, buf, n); 
	 
     	return 0;
}

int do_serv_ls(char *path, int sock_fd)
{
	char buf[MAX_LINE];
	int len;
	int n;
	
	sprintf(buf, "LS %s", path); 
     
	if(my_write(sock_fd, buf, strlen(buf)+1) == -1) 
		return -1;
     
	if( (n = my_read(sock_fd, buf, MAX_LINE)) <= 0)
		return -1;
     
     	if(buf[0] == 'E'){ 
		write(STDOUT_FILENO, buf, n);
	  	return 0;
     	}
     	len = atoi(&buf[3]); 
     
	if(my_write(sock_fd, "RDY", 3) == -1)
		return -1;
     
	while(1){ 
	 	n = my_read(sock_fd, buf, MAX_LINE); 
	  	
		if(n > 0){ 
	       		write(STDOUT_FILENO, buf, n);
	       		len -= n; 
	  	}else if(len == 0){ 
		    	printf("OK\n");
	       		break; 
	  	}else
			return -1;	       
	}

	return 0;
}

int do_bye(int sock_fd)
{
	char buf[MAX_LINE];
	
	sprintf(buf, "BYE");
    	if(my_write(sock_fd, buf, strlen(buf)+1) == -1)
		return -1;
     
	return 0;
}

int do_put(const char *src, const char *dst, int sock_fd)
{
	char *dst_file; 
	struct stat statbuf;
    	int n, fd; 
	int res = -1; 
	char buf[MAX_LINE];
	char *p;

	if(src == NULL || dst == NULL){ 
		printf("wrong command\n");
		return -1;
	}

	if(src[strlen(src)-1]=='/'){ 
	  	printf("source file should be a regular file\n");
	  	return -1;
    	}
     
	if((dst_file = malloc(strlen(dst)+strlen(src)+2)) == NULL){
		perror("fail to malloc");
		return -1;
	}
	
	strcpy(dst_file, dst);
    	if(dst_file[strlen(dst_file)-1] != '/')
		strcat(dst_file, "/");
	p = rindex(src, '/');
    	strcat(dst_file, p + 1);

	if((fd = open(src, O_RDONLY)) == -1){
	  	perror("fail to open src-file");
	  	goto end1;
    	}

 	if(fstat(fd, &statbuf) == -1){
	  	perror("fail to stat src-file");
	  	goto end2;
    	}
    
	if(!S_ISREG(statbuf.st_mode)){ 
		fputs("src-file should be a regular file\n", stderr);
	  	goto end2;
    	}
    
	sprintf(buf, "PUT %d %s", statbuf.st_size, dst_file);
	if(my_write(sock_fd, buf, strlen(buf)+1) == -1) 
		goto end2;
    
	if((my_read(sock_fd, buf, MAX_LINE)) <= 0) 
	  	goto end2;
     
     	if(buf[0]=='E'){ 
	  	write(STDOUT_FILENO, buf, n);
	 	goto end2;
     	}
     
     	while(1){ 
	  	n = read(fd, buf, MAX_LINE);
	  	
		if(n > 0)
	       		if(my_write(sock_fd, buf, n) == -1)
				goto end2;
	  	else if(n == 0){
	       		printf("OK\n");
	      		break;
	  	}else{ 
	       		perror("fail to read\n");
	       		goto end2;
	  	}
	}
    
	res = 0; 
end1:
	close(fd); 
end2:
	free(dst_file); 

	return res; 
}

int do_cd(char * path)
{
	if(chdir(path) == -1){ 
		perror("fail to change directory\n");
	  	return -1;
	}
	
	return 0;
}

int do_ls(char *path)
{	
	char cmd[128];
	char buf[NAME_LEN];
	FILE *fp;

	sprintf(cmd, "ls %s > temp.txt ",path);
	system(cmd);

	fp = fopen("temp.txt", "r");
	if(fp == NULL){
		perror("fail to ls");
		return -1;
	}

	while(fgets(buf, NAME_LEN, fp) == NULL)
		printf("%s", buf);

	fclose(fp);
	return 0;
}

```

# common.h
```
#include <stdio.h>
#include <stdlib.h>
#include <regex.h>
#include <fcntl.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <errno.h>
#include "iolib.h"

#define MAX_LINE 1024
#define PORT 8000 
#define COMMAND_LINE 256
#define MAX_ARG 10 
#define MAX_LENGTH 64 
#define NAME_LEN 256 

struct command_line{
	char *name;
	char *argv[MAX_ARG];
};

extern int split(struct command_line *command, char cline[]);

extern int do_connect(char *ip,struct sockaddr_in *sin, int *sock_fd); 
extern int do_get(const char *src, const char *dst, int sock_fd);
extern int do_put(const char *src, const char *dst, int sock_fd);
extern int do_serv_cd(char *path, int sock_fd); 
extern int do_serv_ls(char *path, int sock_fd); 
extern int do_cd(char * path);
extern int do_ls(char *path);
extern int do_bye(int sock_fd);

```

# input.c
```
#include "common.h"

#define del_blank(pos, cline) { \
	while(cline[pos] != '\0' && (cline[pos] == ' ' || cline[pos] == '\t')) \
	{ \
		pos++; \
	} \
}

#define get_arg(arg, pos, cline) { \
	int i = 0; \
	while(cline[pos] != '\0' || cline[pos] != ' ' || cline[pos] != '\t'){ \
		arg[i++] = cline[pos++]; \
	} \
}

int split(struct command_line * command, char cline[ ])
{
	int i;
	int pos = 0;

	cline[strlen(cline) - 1] = '\0'; 
	del_blank(pos, cline); 

	i = 0;
	while(cline[pos] != '\0'){ 
		if((command->argv[i] = (char *)malloc(MAX_LENGTH)) == NULL){
			perror("fail to malloc");
			return -1;
		}
		
		get_arg(command->argv[i], pos, cline);

		i++;
		del_blank(pos, cline);
	}	
	
	command->argv[i] = NULL; 
	command->name = command->argv[0];

	return i;
}

```

# iolib.c
```
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>

ssize_t my_read(int fd, void *buf, size_t length) 
{ 
	ssize_t done = length; 

	while(done > 0) { 
		done = read(fd, buf, length); 
		
		if(done != length) 
			if(errno == EINTR) 
				done = length; 
			else{
				perror("fail to read"); 
				return -1; 
			}
		else
			break;
	} 

	return done; 
}

ssize_t my_write(int fd, void *buf, size_t length) 
{ 
	ssize_t done = length;

	while(done > 0) { 
		done = write(fd, buf, length);

		if(done != length) 
			if(errno == EINTR) 
				done = length; 
			else{
				perror("fail to write"); 
				return -1; 
			}
		else
			break;
	} 

	return done;
}

```

# iolib.h
```
extern ssize_t my_read(int fd, void *buffer, size_t length);
extern ssize_t my_write(int fd, void *buffer, size_t length);
```

# main.c
```
#include "common.h"

int main(void)
{
	char cline[COMMAND_LINE]; 
	struct command_line command; 
	int sock_fd;
	struct sockaddr_in sin;

	printf("myftp$: "); 
	fflush(stdout); 

	while(fgets(cline, MAX_LINE, stdin) != NULL){ 
		if(split(&command, cline) == -1)
			exit(1);
		if(strcasecmp(command.name, "get") == 0){ 
			if(do_get(command.argv[1], command.argv[2], sock_fd) == -1)
				exit(1); 
		}else if(strcasecmp(command.name, "put") == 0){ 
			if(do_put(command.argv[1], command.argv[2], sock_fd) == -1)
				exit(1);
		}else if(strcasecmp(command.name, "cd") == 0){ 
			if(do_cd(command.argv[1]) == -1)
				exit(1);
		}else if(strcasecmp(command.name, "!cd") == 0){ 
			if(do_serv_cd(command.argv[1], sock_fd) == -1)
				exit(1);
		}else if(strcasecmp(command.name, "ls") == 0){ 
			if(do_ls(command.argv[1]) == -1)
				exit(1);
		}else if(strcasecmp(command.name, "!ls") == 0){ 
			if(do_serv_ls(command.argv[1], sock_fd) == -1)
				exit(1);
		}else if(strcasecmp(command.name, "connect") == 0){ 
			if(do_connect(command.argv[1], &sin, &sock_fd) == -1)
				exit(1);
		}else if(strcasecmp(command.name, "bye") == 0){ 
			if(do_bye(sock_fd) == -1)
				exit(1);
			break;
		}else{ 
			printf("wrong command\n");
			printf("usage : command arg1, arg2, ... argn\n");
		}
		
		printf("myftp$: ");
		fflush(stdout);
	}
     
	if(close(sock_fd) == -1){ 
		perror("fail to close");
		exit(1);
	}

	return 0;
}

```
