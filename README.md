**IRC(Internet Relay Chat)프로토콜을 구현한 채팅 서버** (2인 개발🙋🏻‍♀️🙋🏻‍♀️)  <br>
2023-09.20 ~ 10.19


# Internet Relay Chat
Internet Relay Chat or IRC is a text-based communication protocol on the Internet.
It offers real-time messaging that can be either public or private. Users can exchange
direct messages and join group channels.
IRC clients connect to IRC servers in order to join channels. IRC servers are connected
together to form a network.


![](https://velog.velcdn.com/images/meliesfi/post/8b3d5e35-9de2-4047-9f0b-943473f15bad/image.png)


## Mandatory part
```
다중 클라이언트 동시 처리 
TCP/IP (v4 또는 v6) 통신
non-blocking I/O 작업만 허용

*주의
- Non-blocking 설정은 이것만 허용 -> fcntl(fd, F_SETFL, O_NONBLOCK); 
- poll/kqueue/epoll 없이 직접 read/recv 사용 금지
 // 잘못된 예:
	while (recv(fd, buffer, size, 0) > 0) { ... }
- fork() 사용 금지
- poll() (또는 select, kqueue, epoll) 하나만 사용해서 모든 I/O 작업 처리
- 부분적인 데이터 수신, 낮은 대역폭 등 모든 에러 상황 처리 필요
- 패킷을 재조립하여 완전한 명령어로 처리해야 함
- 서버가 멈추지 않아야 함
```

-- --

<br>
<br>


# 💬 Server
![](https://velog.velcdn.com/images/meliesfi/post/d50d6754-4319-415c-bfd1-7f92fbb026c1/image.png)

### 1. 초기화
 ``` cpp
Copyconst int PORT = this->port;
const int BUFFER_SIZE = 1024;      // 버퍼 크기 설정
const int EVENTLIST_SIZE = 10;     // 이벤트 리스트 크기 설정
// TCP 소켓 생성
int serverSocket = socket(AF_INET, SOCK_STREAM, 0);
if (serverSocket == -1) {
    perror("Socket creation failed");
    return -1;
}
 ```
 
 ### 2. 서버 주소 설정, 바인딩
 ``` cpp
sockaddr_in serverAddress;
memset(&serverAddress, 0, sizeof(serverAddress));
serverAddress.sin_family = AF_INET;         // IPv4
serverAddress.sin_addr.s_addr = INADDR_ANY; // 모든 인터페이스에서 접속 허용
serverAddress.sin_port = htons(PORT);       // 포트 번호 설정

// 소켓에 주소 바인딩
bind(serverSocket, (struct sockaddr *)&serverAddress, sizeof(serverAddress));
 ``` 



### 3. ~~리스닝~~, Non-blocking 설정

``` cpp
listen(serverSocket, 5);  // 연결 대기열 크기 5로 설정
fcntl(serverSocket, F_SETFL, O_NONBLOCK);  // 논블로킹 모드 설정
```
 

### 4. kqueue 초기화
```cpp
int kq = kqueue();  // kqueue 생성
struct kevent event;
std::vector<struct kevent> events;

// 서버 소켓을 kqueue에 등록 (읽기 이벤트 감시)
EV_SET(&event, serverSocket, EVFILT_READ, EV_ADD, 0, 0, NULL);
events.push_back(event);
```


## 이벤트 루프 🔄

 **kqueue를 사용한 이벤트 기반 I/O 처리**
kqueue는 모든 파일 디스크립터의 이벤트를 감시
read/write 가능 상태를 효율적으로 추적
동적 메모리 할당을 통한 데이터 전달

```cpp
struct kevent eventList[EVENTLIST_SIZE];
while (1) 
{
//kqueue에서 이벤트 대기
    int nev = kevent(kq, &events[0], events.size(), eventList, EVENTLIST_SIZE, NULL);
    events.clear();
    //이벤트 처리
    for (int i = 0; i < nev; i++) 
    {    	
        if (eventList[i].flags & EV_EOF) 
        { // 연결 종료 처리
    	     close(eventList[i].ident);  // 소켓 닫기
        }
        else if (eventList[i].ident == (uintptr_t)serverSocket) 
        { // 새 클라이언트 연결 처리
         // 새 클라이언트 소켓을 읽기 이벤트로 등록
	    	EV_SET(&event, clientSocket, EVFILT_READ, EV_ADD, 0, 0, NULL);
		    events.push_back(event);
        }
        else if (eventList[i].filter == EVFILT_READ) 
        {// 클라이언트로부터 데이터 읽기
    		int clientSocket = eventList[i].ident;
		    char buffer[BUFFER_SIZE];
    		int bytesRead;
	    	std::string buf = "";
    	
    		while ((bytesRead = recv(clientSocket, buffer, sizeof(buffer) - 1, 0)) > 0) 
	        {
		        buffer[bytesRead] = '\0';
    		    buf.append(buffer);
        		if (bytesRead < BUFFER_SIZE - 1)
            		break;
	    	}	
    
    	// 쓰기 이벤트 등록
		    Udata *udata = new Udata();
    		udata->buf = buf;
		    EV_SET(&newEvent, clientSocket, EVFILT_WRITE, EV_ADD | EV_ONESHOT, 0, 0, udata);
		    events.push_back(newEvent);
		}
        else if (eventList[i].filter == EVFILT_WRITE) 
        { // 클라이언트에게 응답 보내기
            int clientSocket = eventList[i].ident;
		    Udata *tmp = static_cast<Udata *>(eventList[i].udata);
	    	communicateClient(clientSocket, tmp->buf);  // 실제 통신 처리
		    delete tmp;  // 메모리 해제
        }
    }
}
```


> 
``EVFILT_READ``: 데이터 수신 가능
``EVFILT_WRITE``: 데이터 송신 가능
``EV_EOF``: 연결 종료
``EV_ADD``: 이벤트 감시 추가
``EV_ONESHOT``: 한 번만 사용되는 이벤트 (일회성 쓰기 이벤트 처리)


<br>
<br>


## IRC 프로토콜 명령어 

### 채널 모드 지원
``` cpp
class Channel {
private:
    std::string name;
    std::string topic;
    int modes[5];  // [INVITE, TOPIC, KEY, OPER, LIMIT]
    std::map<int, Client*> clients;      // 일반 사용자
    std::map<int, Client*> oClients;     // 운영자
    std::map<int, Client*> iClients;     // 초대된 사용자
    std::string key;
    unsigned int limit;
};
```


```cpp
class Server {
public:
		/* cmds */
        void pass(MessageInfo *msg, Client *client);
        void ping(MessageInfo *msg, Client *client);
        void nick(MessageInfo *msg, Client *client);
        void user(MessageInfo *msg, Client *client);
        void join(MessageInfo *msg, Client *client);
        void part(MessageInfo *msg, Client *client);
        void names(MessageInfo *msg, Client *client);
        void topic(MessageInfo *msg, Client *client);
        void list(MessageInfo *msg, Client *client);
        void invite(MessageInfo *msg, Client *client);
        void kick(MessageInfo *msg, Client *client);
        void mode(MessageInfo *msg, Client *client);
        void privmsg(MessageInfo *msg, Client *client);
        void notice(MessageInfo *msg, Client *client);
        void who(MessageInfo *msg, Client *client);
        void quit(MessageInfo *msg, Client *client);
        void whois(MessageInfo *msg, Client *client);
};
```
### 연결 관리
`PASS` : 서버 접속 비밀번호 인증 (연결 비밀번호 설정?)
```
PASS <password>
다른 명령어보다 먼저 실행되어야 함
실패시 클라이언트 연결 즉시 종료
성공시 client->setValid(true)로 설정
```
`NICK` : 사용자 닉네임 설정/변경

```
NICK <nickname>
최대 9자 제한
허용 문자: 알파벳, 숫자, [], {}, , |
중복 닉네임의 경우 자동으로 언더스코어(_) 추가
```
`USER` : 사용자 등록 (상세 정보 등록)
```
USER <username> <hostname> <servername> :<realname>
NICK 명령어와 함께 초기 등록 과정에서 사용
realname은 공백 포함 가능
한 번만 설정 가능
```
`QUIT` : 연결 종료
```
QUIT [:<reason>]
모든 채널에서 자동 퇴장
연결된 모든 리소스 정리
종료 메시지 전송

클라이언트 소켓 종료
메모리 해제
채널 멤버 목록에서 제거
```
`PING` : 연결 상태 확인
```
PING [server]
서버는 PONG으로 응답
연결 유지 확인용
지연 시간 측정 가능
```
#### 채널 관리
`JOIN` : 채널 입장/생성
```
JOIN <channel> [password]
채널명은 '#' 또는 '&'로 시작
새 채널 생성시 생성자가 자동으로 운영자 권한 획득
한 번에 하나의 채널만 참여 가능
모드에 따른 접근 제한 (비밀번호, 초대전용, 인원제한)
```

`PART` : 사용자가 채널에서 퇴장
```
PART <channel> [:<reason>]
현재 참여중인 채널에서만 가능
채널의 마지막 사용자가 나가면 채널 삭제
모든 채널 멤버에게 퇴장 알림
```
`KICK` : 채널에서 클라이언트를 강제로 제거
```
KICK <channel> <user> [:<reason>]
채널 운영자만 사용 가능
퇴장된 사용자와 채널 멤버들에게 알림
채널의 마지막 사용자가 나가면 채널 삭제

```
`INVITE` : 채널 초대
```
INVITE <nickname> <channel>
채널 멤버만 초대 가능
초대된 사용자는 invite-only 채널에 입장 가능
초대 받은 사용자와 채널 멤버들에게 알림
```
`TOPIC` : 채널 주제 관리
#### 채널 모드
`MODE` : 채널 모드 설정
 
   `+i`: invite-only 초대 전용 모드
   `+t`: topic 주제 변경 제한 모드
   `+k`: 채널 비밀번호 모드
   `+o`: 운영자 권한 모드
   `+l`: 인원 제한 모드
    
- '+' 추가, '-' 제거
- 운영자만 변경 가능
- 각 모드별 추가 매개변수 필요 여부가 다름
#### 메시지 전송
`PRIVMSG` : 메시지 전송 / 개인 또는 채널 메시지 전송
```
PRIVMSG <target> :<message>
채널 메시지: '#'으로 시작하는 대상
개인 메시지: 닉네임으로 전송
채널 메시지는 발신자 제외 모든 멤버에게 전달
```
`NOTICE` : 응답이 필요 없는 메시지 전송 (PRIVMSG와 유사하지만 자동 응답을 발생시키지 않는 메시지 전송)
```
NOTICE <target> :<message>
서버/클라이언트 간 자동화된 메시지에 주로 사용
PRIVMSG와 달리 자동 응답 triggering 없음
채널('#'으로 시작) 또는 개인에게 전송 가능
서버 공지나 시스템 메시지에 자주 사용




```
#### 정보 조회
`WHO` : 사용자/채널 정보 조회 (대상의 기본 정보 조회)
```
WHO <channel/nickname>
채널 멤버 목록 또는 사용자 정보 표시
운영자 상태(@) 표시
"352" 응답으로 정보 전달
"315" 응답으로 목록 종료
```
`WHOIS` : 사용자 상세 정보
```
WHOIS <nickname>
닉네임, 사용자명, 호스트명 표시 (311)
서버 정보 표시 (312)
idle 시간과 접속 시간 표시 (317)
운영자인 경우 추가 정보 표시 (319)
"318"로 정보 종료
```

### 에러 처리

``` cpp
void Server::sendResponse(std::string msg, Client *client) {
    msg.append("\r\n");
    std::cout << "***send: " << msg << std::endl;
    send(client->getSocket(), msg.c_str(), msg.size(), 0);
}

void Server::noSuchNick(int fd, std::string nickname, std::string params) {
    std::string msg = ":ft_irc 401 " + nickname + " " + params + 
                     " :No such nick/channel\r\n";
    send(fd, msg.c_str(), msg.size(), 0);
    throw std::runtime_error("no such user");
}
```

**이 프로토콜들은 RFC 문서에 정의된 대로 메시지 포맷과 응답 코드를 구현**
- 401: No such nick/channel
- 403: No such channel
- 461: Not enough parameters

RFC 표준에 따른 에러 코드 구현
RFC 1459, RFC 2810-2813 참고

## 메모리 관리
```cpp
void Server::removeClient(Client *client) {
    if (client) {
        // 1. 채널에서 먼저 제거
        if (client->getCurrentchannel() != "*")
            removeClientFromChannel(client->getCurrentchannel(), client);
        
        // 2. 서버에서 제거
        int fd = client->getSocket();
        if (clients.find(fd) != clients.end()) {
            clients.erase(fd);
            nClients.erase(fd);
            close(fd);
            delete client;
        }
    }
}
```
No leaks

## 리팩토링 예정

1. while 루프로 recv를 반복 호출하는 대신, 한 번의 recv만 수행
2. EAGAIN/EWOULDBLOCK 에러 처리 추가
3. Server 클래스 책임 분리
	_네트워크, 명령어 처리, 사용자 관리, 채널 관리_
4. 안전한 문자열 처리 (버퍼 오버플로우 방지)







### Makefile을 사용하여 빌드
```
OBJS = $(SRCS:.cpp=.o)

all: $(NAME)

$(NAME) : $(OBJS)
	$(CPP) $(CFLAGS) -o $(NAME) $(OBJS)

%.o: %.cpp
	$(CPP) $(CFLAGS) -c $< -o $@ 

clean:
	rm -f $(OBJS)

fclean : clean
	rm -f $(NAME)

re: fclean all

.PHONY:	all clean fclean re
```




#### Peer evaluations
![](https://velog.velcdn.com/images/meliesfi/post/f800822f-1770-4c7a-bfa9-4b7c75ec29e0/image.png)




 





