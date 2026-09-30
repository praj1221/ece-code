#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/ip_icmp.h>
#include <sys/socket.h>
#include <sys/time.h>

unsigned short checksum(void *data, int length)
{
    unsigned short *ptr = data;
    unsigned int sum = 0;

    while (length > 1)
    {
        sum += *ptr++;
        length -= 2;
    }

    if (length == 1)
        sum += *(unsigned char *)ptr;

    sum = (sum >> 16) + (sum & 0xffff);
    sum += (sum >> 16);

    return (unsigned short)(~sum);
}

int main(int argc, char *argv[])
{
    if (argc != 2)
    {
        printf("Usage: sudo %s <IP address>\n", argv[0]);
        return 1;
    }

    int sockfd;

    sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);

    if (sockfd < 0)
    {
        perror("socket");
        return 1;
    }

    struct sockaddr_in destination;

    memset(&destination, 0, sizeof(destination));

    destination.sin_family = AF_INET;

    if (inet_pton(AF_INET, argv[1],
                  &destination.sin_addr) <= 0)
    {
        printf("Invalid IP address\n");
        return 1;
    }

    char packet[64];

    memset(packet, 0, sizeof(packet));

    struct icmphdr *icmp =
        (struct icmphdr *)packet;

    icmp->type = ICMP_ECHO;
    icmp->code = 0;

    icmp->un.echo.id = getpid();
    icmp->un.echo.sequence = 1;

    icmp->checksum = 0;

    icmp->checksum =
        checksum(packet, sizeof(packet));

    struct timeval start, end;

    gettimeofday(&start, NULL);

    sendto(sockfd,
           packet,
           sizeof(packet),
           0,
           (struct sockaddr *)&destination,
           sizeof(destination));

    printf("PING %s\n", argv[1]);

    char buffer[1024];

    socklen_t address_length =
        sizeof(destination);

    int bytes = recvfrom(sockfd,
                         buffer,
                         sizeof(buffer),
                         0,
                         (struct sockaddr *)&destination,
                         &address_length);

    if (bytes < 0)
    {
        perror("recvfrom");
        close(sockfd);
        return 1;
    }

    gettimeofday(&end, NULL);

    double time_ms =
        (end.tv_sec - start.tv_sec) * 1000.0 +
        (end.tv_usec - start.tv_usec) / 1000.0;

    printf("%d bytes received\n", bytes);

    printf("Reply from %s: time=%.3f ms\n",
           argv[1], time_ms);

    close(sockfd);

    return 0;
}
