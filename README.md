# 平行的世界
实现一个有用的网络需要解决大量不同的问题

        1.TCP/IP编程原理
        典型的TCP客户的通信涉及如下4个基本步骤:
        (1)使用socket()创建TCP套接字
        (2)使用connect()建立到达服务器的连接
        (3)使用send()和recv()通信
        (4)使用close()关闭连接
        client.c
        #include <stdio.h>
        #include <stdlib.h>
        #include <string.h>
        #include <unistd.h>
        #include <sys/types.h>
        #include <sys/socket.h>
        #include <netinet/in.h>
        #include <arpa/inet.h>
        #include "Practical.h"
        
        int main(int argc, char* argv[]){
            if(argc < 3 || argc >4)
                DieWithUserMessage("Parameter(s)", "<Server Address> <Echo Word> [<Server Port>]");
            char* servIP = argv[1];
            char* echoString = argv[2];
            
            in_port_t servPort = (argc == 4) ? atoi(argv[3]) : 7;
            
            int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
            if(sock < 0)
                DieWithSystemMessage("socket() failed");
                
            struct sockaddr_in servAddr;
            memset(&servAddr, 0, sizeof(servAddr));
            servAddr.sin_family = AF_INET;
            
            int rtnVal = inet_pton(AF_INET, servIP, &servAddr.sin_addr.s_addr);
            if(rtnVal == 0)
                DieWithUserMessage("inet_pton() failed", "invalid address string");
            else if(rtnVal < 0)
                DieWithSystemMessage("inet_pton() failed", "invalid address string");
            servAddr.sin_port = htons(servPort);
            
            if(connect(sock, (struct sockaddr*) &servAddr, sizeof(servAddr)) < 0)
                DieWithSystemMessage("connect() failed");
            
            size_t echoStringLen = strlen(echoString);
            
            ssize_t numBytes = send(sock, echoString, echoStringLen, 0);
            if(numBytes < 0)
                DieWithSystemMessage("send() failed");
            else if(numBytes != echoStringLen)
                DieWithUserMessage("send()", "sent unexpected number of bytes");
                
            unsigned int totalBytesRcvd = 0;
            fputs("Received: ", stdout);
            while(totalBytesRcvd < echoStringLen){
                char buffer[BUFSIZE];
                numBytes = recv(sock, buffer, BUFSIZE - 1, 0);
                if(numBytes < 0)
                    DieWithSystemMessage("recv() failed");
                else if(numBytes == 0)
                    DieWithUserMessage("recv()", "connection closed prematurely");
                totalBytesRcvd += numBytes;
                buffer[numBytes] = '\0';
                fputs(buffer, stdout);
            }
            fputc('\n', stdout);
            
            close(sock);
            exit(0);
        }
        
        DieWithMessage.c
        #include <stdio.h>
        #include <stdlib.h>
        
        void DieWithUserMessage(const char* msg, const char* detail){
            fputs(msg, stderr);
            fputs(": ", stderr);
            fputs(detail, stderr);
            fputc('\n', stderr);
            exit(1);
        }
        
        void DieWithSystemMessage(const char* msg){
            perror(msg);
            exit(1);
        }
        
        对应上面的TCP服务器涉及如下4个基本步骤:
        (1)使用socket()创建TCP套接字
        (2)利用bind()给套接字分配端口号
        (3)使用listen()告诉系统允许对该端口建立连接
        (4)反复执行以下操作:
            调用accept()为每个客户连接获取新的套接字
            使用send()和recv()通过新的套接字与客户通信
            使用close()关闭客户连接
            
        #include <stdio.h>
        #include <stdlib.h>
        #include <string.h>
        #include <sys/types.h>
        #include <sys/socket.h>
        #include <netinet/in.h>
        #include <arpa/inet.h>
        #include "Practical.h"
        
        static const int MAXPENDING = 5;
        
        int main(int argc, char* argv[]){
            if(argc != 2)
                DieWithUserMessage("Parameter(s)", "<Server Port>");
            in_port_t servPort = atoi(argv[1]);
            
            int servSock;
            if((servSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
                DieWithSystemMessage("socket() failed");
                
            struct sockaddr_in servAddr;
            memset(&servAddr, 0, sizeof(servAddr));
            servAddr.sin_family = AF_INET;
            servAddr.sin_addr.s_addr = htonl(INADDR_ANY);
            servAddr.sin_port = htons(servPort);
            
            if(bind(servSock, (struct sockaddr*)&servAddr, sizeof(servAddr)) < 0)
                DieWithSystemMessage("bind() failed");
                
            if(;;){
                struct sockaddr_in clntAddr;
                socklen_t clntAddrLen = sizeof(clntAddr);
                
                int clntSock = accept(servSock, (struct sockaddr*)&clntAddr, &clntAddrLen);
                if(clntSock < 0)
                    DieWithSystemMessage("accept() failed");
                    
                char clntName[INET_ADDRSTELEN];
                if(inet_ntop(AF_INET, &clntAddr.sin_addr.s_addr, clntName, sizeof(clntName)) != NULL)
                    printf("Handling client %s/%d\n", clntName, ntohs(clntAddr.sin_port));
                else
                    puts("Unable to get client address");
                    
                HandleTCPClient(clntSock);
            }
        }
        
        TCPServerUtility.c
        void HandleTCPClient(int clntSocket){
            char buffer[BUFSIZE];
            
            ssize_t numBytesRcvd = recv(clntSocket, buffer, BUFSIZE, 0);
            if(numBytesRcvd < 0)
                DieWithSystemMessage("recv() failed");
            while(numBytesSent > 0){
                ssize_t numBytesSent = send(clntSocket, buffer, numBytesRcvd, 0);
                if(numBytesSent < 0)
                    DieWithSystemMessage("send() failed");
                else if(numBytesSent != numBytesRcvd)
                    DieWithUserMessage("send()", "sent unexpected number of bytes");
                    
                numBytesRcvd = recv(clntSocket, buffer, BUFSIZE, 0);
                if(numBytesRcvd < 0)
                    DieWithSystemMessage("recv() failed");
            }
            
            close(clntSocket);
        }
        
        手工编码整数
        BruteForceCoding.c
        #include <stdint.h>
        #include <stdlib.h>
        #include <stdio.h>
        #include <limits.h>
        #include "Practical.h"
        
        const uint8_t val8 = 101;
        const uint16_t val16 = 10001;
        const uint32_t val32 = 100000001;
        const uint64_t val64 = 1000000000001L;
        const int MESSAGELENGTH = sizeof(uint8_t) + sizrof(uint16_t) + sizeof(uint32_t) + sizeof(uint64_t);
        
        static char stringBuf[BUFSIZE];
        
        char* BytesToDecString(uint8_t* byteArray, int arrayLength){
            char* cp = stringBuf;
            size_t bufSpaceLeft = BUFFSIZE;
            for(int i = 0; i < arrayLength && bufSpaceLeft > 0; i++){
                int strl = snprintf(cp, bufSpaceLeft, "%u ", byteArray[i]);
                bufSpaceLeft -= strl;
                cp += strl;
            }
            return stringBuf;
        }
        
        int EncodeIntBigEndian(uint8_t dst[], uint64_t val, int offset, int size){
            for(int i = 0; i < size; i++){
                dst[offset++] = (uint8_t)(val >> ((size - 1) - 1)* CHAR_BIT);
            }
            return offset;
        }
        
        uint64_t DecodeIntBigEndian(uint8_t val[], int offset, int size){
            uint64_t rtn = 0;
            for(int i = 0; i < size; i++){
                rtn = (rtn << CHAR_BIT) | val[offset + i];
            }
            return rtn;
        }
        
        int main(int argc char* argv[]){
            uint8_t message[MESSAGELENGTH];
            
            int offset = 0;
            offset = EncodeIntBigEndian(message, val8, offset, sizeof(uint8_t));
            offset = EncodeIntBigEndian(message, val16, offset, sizeof(uint16_t));
            offset = EncodeIntBigEndian(message, val32, offset, sizeof(uint32_t));
            offset = EncodeIntBigEndian(message, val64, offset, sizeof(uint64_t));
            printf("Encode message:\n%s\n", BytesToDecString(message, MESSAGELENGTH));
            
            uint64_t value = DecodeIntBigEndian(message, sizeof(unit8_t), sizeof(uint16_t) + sizeof(uint32_t) + sizeof(uint64_t));
            printf("Decoded 2-byte integer = %llu\n", value);
            
            offset = 4;
            int iSize = sizeof(size32_t);
            value = DecodeIntBigEndian(message, offset, iSize);
            printf("Decoded value(offset %d, size %d) = %lld\n", offset, iSize, value);
            int signedVal = DecodeIntBigEndian(message, offset, iSize);
            printf("...same as signed value %d\n", signedVal);
        }
        
        2.Linux下C的几种网络编程模型(后端)
        
        3.Windows下C#下的几种网络编程模型(前端)
        
        4.统一用于通讯的数据结构
        
        5.Linux下C服务模块的实现(后端)
        
        6.Windows下C#服务模块的实现(前端)
        
