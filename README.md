#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8080

int main() {
    int server_fd, client_fd;
    struct sockaddr_in address;
    socklen_t addrlen = sizeof(address);

    server_fd = socket(AF_INET, SOCK_STREAM, 0);

    if (server_fd < 0) {
        perror("Socket failed");
        return 1;
    }

    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(server_fd, (struct sockaddr *)&address,
             sizeof(address)) < 0) {
        perror("Bind failed");
        return 1;
    }

    if (listen(server_fd, 5) < 0) {
        perror("Listen failed");
        return 1;
    }

    printf("Server waiting for client...\n");

    client_fd = accept(server_fd,
                       (struct sockaddr *)&address,
                       &addrlen);

    if (client_fd < 0) {
        perror("Accept failed");
        return 1;
    }

    printf("Client connected!\n");

    char buffer[1024];

    while (1) {
        memset(buffer, 0, sizeof(buffer));

        int n = recv(client_fd, buffer,
                     sizeof(buffer) - 1, 0);

        if (n <= 0)
            break;

        printf("Received: %s\n", buffer);

        char ack[100];

        sprintf(ack, "ACK %s", buffer);

        send(client_fd, ack, strlen(ack), 0);

        printf("Sent: %s\n", ack);

        if (strcmp(buffer, "Frame 4") == 0)
            break;
    }

    close(client_fd);
    close(server_fd);

    return 0;
}
