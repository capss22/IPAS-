# command.c
```
#include "common.h"

int init(struct sockaddr_in *sin, int *lfd, int sock_opt)
{
	int tfd;
	
	bzero(sin, sizeof(struct sockaddr_in));
     sin->sin_family = AF_INET;
     sin->sin_addr.s_addr = INADDR_ANY;
     sin->sin_port = htons(PORT);
    
     if( (tfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
          perror("fail to creat socket");
          return -1;
     }
     
	setsockopt(tfd, SOL_SOCKET, SO_REUSEADDR, &sock_opt, sizeof(int));

   
     if( (bind(tfd, (struct sockaddr *)sin, sizeof(struct sockaddr_in))) == -1){
          perror("fail to bind");
          return -1;
     }
	
     if( (listen(tfd, 20)) == -1){
          perror("fail to listen");
	  return -1;
     }
	
	*lfd = tfd;

	return 0;

}


int do_put(int cfd, char *file)
{
     struct stat statbuf; 
     int n, fd;
	int res = -1; 
	char buf[1024];
     
	if( (fd = open(file, O_RDONLY)) == -1){ 
	  	my_write(cfd, "ERR open server file\n", strlen("ERR open server file\n"));  
		
		return res; /* 傳回-1 */
	}

 	if( (fstat(fd, &statbuf)) == -1){ 
	  	my_write(cfd, "ERR stat server file\n", strlen("ERR stat server file\n"));
		goto end;
     	}
     
	if(!S_ISREG(statbuf.st_mode)){
	  	if(my_write(cfd, "ERR server path should be a regular file\n",
		 	strlen("ERR server path should be a regular file\n")) == -1)
			goto end;
		
		res = 0; 
		goto end;
     }
	
     sprintf(buf, "OK %d", statbuf.st_size); 
     if(my_write(cfd, buf, strlen(buf)) == -1) 
		goto end;
     if ( (my_read(cfd, buf, MAX_LINE)) <= 0)
		goto end;
     
	while(1){ 
	 	n = read(fd, buf, MAX_LINE); 

		if(n > 0)
	       		if(my_write(cfd, buf, n) == -1) 
				goto end;
	  	else if(n == 0) { 
	       	printf("OK\n");  
	       	break;
	  	}else	{ 
	  		perror("fail to read");
			goto end;	 
		}
    	}
	res = 0; 
end:
     close(fd); 
	return res;
}

int do_get(int cfd, char *file)
{
     struct stat statbuf; /* 檔案狀態緩衝區 */
     int n, fd;
	int res = -1;
		char buf[1024];
	int len;
 
	if( (fd = open(file, O_WRONLY|O_CREAT|O_TRUNC, 0644)) == -1){ 
		if(errno == EISDIR){
	       	if(my_write(cfd, "ERR server has a dir with the same name\n",
		     	strlen("ERR server has a dir with the same name\n")) == -1) 
				goto end;
              
			res = 0; 
			goto end;
		}else{
	       	my_write(cfd, "ERR open server file\n", strlen("ERR open server file\n")); 
			goto end;
          }
	}
	
	if( (fstat(fd, &statbuf)) == -1){ 
	  	my_write(cfd, "ERR stat server file\n", strlen("ERR stat server file\n")); 
	  	goto end;
     }

	len = statbuf.st_size;
     
	if(!S_ISREG(statbuf.st_mode)){ 
	  	if(my_write(cfd, "ERR server path should be a regular file\n",
			strlen("ERR server path should be a regular file\n")) == -1)
			goto end;
		
		res = 0;
		goto end;
     }
     
	if(my_write(cfd, "OK", 2) == -1) 
		goto end;

     while(1){
	  	n = my_read(cfd, buf, MAX_LINE);
	  	
		if(n > 0){
	       	write(fd, buf, n);
	       	len -= n;
	  	}else if(len == 0) {
		    	printf("OK\n");
	       		break;
	  	}else
	      		goto end;
     }
	
	res = 0;
end:
     close(fd); 

	return res;
}

int do_cd(int cfd, char *path)
{
	if(chdir(path) == -1){ 
		perror("fail to change directory\n");
		my_write(cfd, "ERR can't change directory\n", strlen("ERR can't change directory\n"));
	  	
		return -1;
	}

	my_write(cfd, "OK\n", strlen("OK\n"));
	
	return 0;
}

int do_ls(int cfd, char *path)
{
	char cmd[128];
	char buf[NAME_LEN];
	struct stat statbuf; 
    	int n, fd;
	int res = -1;

	sprintf(cmd, "ls %s > temp.txt ",path);
	system(cmd);

	if( (fd = open("temp.txt", O_RDONLY)) == -1){ 
	  	my_write(cfd, "ERR ls server file\n", strlen("ERR ls server file\n"));  
		return res;
	}

 	if( (fstat(fd, &statbuf)) == -1){
	  	my_write(cfd, "ERR stat server file\n", strlen("ERR stat server file\n"));
		goto end;
     }

     if(!S_ISREG(statbuf.st_mode)){
	  	if(my_write(cfd, "ERR server path should be a regular file\n",
		 	strlen("ERR server path should be a regular file\n")) == -1)
			goto end;
		res = 0;
		goto end;
     }
	
     sprintf(buf, "OK %d", statbuf.st_size); 
     if(my_write(cfd, buf, strlen(buf)) == -1) 
		goto end;
     
     if ( (my_read(cfd, buf, MAX_LINE)) <= 0)
		goto end;
     
	while(1){
	 	n = read(fd, buf, MAX_LINE); 
	  	if(n > 0)
	       	if(my_write(cfd, buf, n) == -1) 
				goto end;
	  	else if(n == 0) { 
	       	printf("OK\n"); 
	       	break;
	  	}else	{ 
	  		perror("fail to read");
			goto end;	 
		}
    	}

	res = 0;
end:
     close(fd); 
	
	return res;
}

```

# common.h
```
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <ctype.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <errno.h>
#include "iolib.h"

#define MAX_LINE 1024
#define PORT 8000
#define ADDR_LEN 17
#define NAME_LEN 256

extern int init(struct sockaddr_in *sin, int *lfd, int sock_opt); 
extern int do_put(int cfd, char *file); 
extern int do_get(int cfd, char *file); 
extern int do_cd(int cfd, char *path); 
extern int do_ls(int cfd, char *path); 

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
     struct sockaddr_in sin; 
     struct sockaddr_in cin; 
     int lfd, cfd;
     socklen_t len = sizeof(struct sockaddr_in); 

     char buf[MAX_LINE];
     char str[ADDR_LEN]; 
     int sock_opt = 1;
     int n; 
	pid_t pid; 
    	
	if(init(&sin, &lfd, sock_opt) == -1) 
		exit(1);
	
	printf("waiting connections ...\n");
     
     while(1){ 
         if( (cfd = accept(lfd, (struct sockaddr *)&cin, &len)) == -1){ 
			perror("fail to accept");
	       	exit(1);
	  	}

		if( (pid = fork()) < 0){
			perror("fail to fork");
			exit(1);
		}else if(pid == 0){ 
			close(lfd); 
			
			while(1){ 
				if(my_read(cfd, buf, MAX_LINE) == -1) 
					exit(1);

	  			if(strstr(buf, "GET") == buf)
	       				if(do_put(cfd, &buf[4]) == -1)
						printf("error occours while putting\n");
	  			else if(strstr(buf, "PUT") == buf)
					if(do_get(cfd, &buf[4]) == -1) 
						printf("error occours while getting\n");
				else if(strstr(buf, "CD") == buf) 
					if(do_cd(cfd, &buf[4]) == -1)
						printf("error occours while changing directory\n");
				else if(strstr(buf, "LS") == buf)
					if(do_ls(cfd, &buf[4]) == -1)
						printf("error occours while listing\n");
				else if(strstr(buf, "BYE") == buf) 
					break; 
				else{ 
					printf("wrong command\n");
					exit(1);
				}
			}
	  
			close(cfd);
	
			exit(0);
     	}else
			close(cfd);
   	}
	
  	return 0; 
}

```
