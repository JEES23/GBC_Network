```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <pthread.h>

#define BUF_SIZE 100
#define MAX_CLNT 256

void * handle_clnt(void * arg);
void send_msg(char * msg, int len);
void error_handling(char * msg);

int clnt_cnt=0;     
int clnt_socks[MAX_CLNT];
pthread_mutex_t mutx;

int main(int argc, char *argv[])
{
	int serv_sock, clnt_sock;
	struct sockaddr_in serv_adr, clnt_adr;
	int clnt_adr_sz;
	pthread_t t_id;
	if(argc!=2) {
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
  
	pthread_mutex_init(&mutx, NULL);            // 구조체 초기화           
	serv_sock=socket(PF_INET, SOCK_STREAM, 0);

	memset(&serv_adr, 0, sizeof(serv_adr));    //모든 멤버 0으로 초기화 
	serv_adr.sin_family=AF_INET;               // 주소체계 지정
	serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);   //Ip주소 초기화
	serv_adr.sin_port=htons(atoi(argv[1]));         //PORT번호 초기화
	
	if(bind(serv_sock, (struct sockaddr*) &serv_adr, sizeof(serv_adr))==-1)  //IP, PORT 번호 부여
		error_handling("bind() error");
	if(listen(serv_sock, 5)==-1)             // 연결 요청
		error_handling("listen() error");
    
    printf("Welcome to GBC Network Server!\n");  // ""출력
	
	while(1)
	{
		clnt_adr_sz=sizeof(clnt_adr);
		clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr,&clnt_adr_sz);
		
		pthread_mutex_lock(&mutx);         // pthread 독립적으로 실행
		clnt_socks[clnt_cnt++]=clnt_sock;        //연결추가시 해당정보 등록
		pthread_mutex_unlock(&mutx);      
	
		pthread_create(&t_id, NULL, handle_clnt, (void*)&clnt_sock);    //pthread 생성
		pthread_detach(t_id);          //종료된 pthread 완전 소멸
		printf("Connected client IP: %s \n", inet_ntoa(clnt_adr.sin_addr));
	}
	close(serv_sock);
	return 0;
}
	
void * handle_clnt(void * arg)
{
	int clnt_sock=*((int*)arg);   //형변환
	int str_len=0, i;
	char msg[BUF_SIZE];
	
	while((str_len=read(clnt_sock, msg, sizeof(msg)))!=0)
		send_msg(msg, str_len);             //메세지 보내기
	
	pthread_mutex_lock(&mutx);
	for(i=0; i<clnt_cnt; i++)   // remove disconnected client
	{
		if(clnt_sock==clnt_socks[i])
		{
			while(i++<clnt_cnt-1)
				clnt_socks[i]=clnt_socks[i+1];
			break;
		}
	}
	clnt_cnt--;
	pthread_mutex_unlock(&mutx);
	close(clnt_sock);      //끝내기
	return NULL;
}
void send_msg(char * msg, int len)   // send to all
{
	int i;
	pthread_mutex_lock(&mutx);
	for(i=0; i<clnt_cnt; i++)
		write(clnt_socks[i], msg, len);
	pthread_mutex_unlock(&mutx);
}
void error_handling(char * msg)
{
	fputs(msg, stderr);
	fputc('\n', stderr);
	exit(1);
}
```