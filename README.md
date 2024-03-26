# Chatroom-in-Python

![Monty Python](https://cdn.dribbble.com/users/20368/screenshots/3949907/live_chat_anim_2.gif)


### Overview
**A chatroom is a minimal chatting application. A chatroom is a minimal server that we can create and allow several clients to connect. All the clients that are connected to the server can transfer messages and data to one another via the central server. In this project, we will broadcast our message to every client connected to the server so that everyone can see the messages of other clients. For connecting the clients, both the server and the clients need to be either on the same machine or in the same network. The chatroom can work on the LAN only.**

### Introduction

**A chatroom is nothing but a server that we can create and allow several clients to connect to the server. All the clients that are connected to the server can transfer messages and data to one another via the central server.
A chatroom is a minimal chatting application. We can use the code as a reference for creating the backend of a chatting application. We can further add frontend and any database to create a full-fledged chatting application. In our chat room, we will define a certain IP address and Port Number, and we will provide the IP address and Port number to our clients. So, a client can connect to the network using the address and port number. So, we can connect several users (as clients) and they can send messages.
In our project, we will be broadcasting our message so every user (client) connected to the network (server) can see the messages of one another. We can visualize the system as a group chat where all the group members (here all the clients) can transfer the message which is available to everyone present in that group.
Now for connecting the clients, both the server and the clients need to be either on the same machine or in the same network. Hence, the chatroom can work on the LAN only. We can further scale up our application by providing access to a remote server. But in this article, we will be using a pre-defined IP address and Port number to connect the clients on the LAN.
For creating a chatroom server and client(s), we use the concept of socket programming (Refer to the next section for detailed discussion). we will also be using the concepts of multi-threading to deal with the numerous clients. We can create a new thread every time a new client joins or requests the server. We will be learning about multi-threading in detail in the later section.
Now before learning about how to build a chatroom in Python. We should be familiar with the concepts of the client-server model as we are going to perform client-side scripting and server-side scripting in this article.
The client-server network model is one of the most widely used networking models. In a client-server network, a specific server and clients are connected to the server. Refer to the diagram below to see the basic overview of a client-server network system architecture.**

**For Learn more : https://www.scaler.com/topics/build-a-chatroom-in-python/**



### Server Side Script
```Python
# importing the necessary modules.
import socket
import select

# defining the header length.
HEADER_LENGTH = 10

# defining the IP address and Port Number.
IP = "127.0.0.1"
PORT = 1234

"""
Creating a server socket and providing the address family (socket.AF_INET) and type of connection (socket.SOCK_STREAM), i.e. using TCP connection.
"""
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

"""
Modifying the socket to allow us to reuse the address. We have to provide the socket option level and set the REUSEADDR (as a socket option) to 1 so that address is reused.
"""
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# Binding the socket with the IP address and Port Number.
server_socket.bind((IP, PORT))

# Making the server listen to new connections.
server_socket.listen()

# List of sockets for select.select()
sockets_list = [server_socket]

# A set to contain the connected clients.
clients = {}

print(f'Listening for connections on IP = {IP} at PORT = {PORT}')


# A function for handling the received message.
def receive_message(client_socket):
    try:
        """
        The received message header contains the message length, its size is defined, and the constant.
        """
        message_header = client_socket.recv(HEADER_LENGTH)

        """
        If the received data has no length then it means that the client has closed the connection. Hence, we will return False as no message was received.
        """
        if not len(message_header):
            return False

        # Convert header to an int value
        message_length = int(message_header.decode('utf-8').strip())

        # Returning an object of the message header and the data of the message data.
        return {'header': message_header, 'data': client_socket.recv(message_length)}

    except:
        return False


# running an infinite loop to accept continuous client requests.
while True:
    # Read the data using a select module from the socketLists.
    read_sockets, _, exception_sockets = select.select(
        sockets_list, [], sockets_list)

    # Iterating over the notified sockets.
    for notified_socket in read_sockets:
        """
        If the notified socket is a server socket then we have a new connection, so add it using the accept() method.
        """
        if notified_socket == server_socket:
            client_socket, client_address = server_socket.accept()

            # Else the client has disconnected before sending his/her name.
            user = receive_message(client_socket)

            # If False - client disconnected before he sent his name
            if user is False:
                continue

            # Adding the accepted socket to select.select() list.
            sockets_list.append(client_socket)

            # Also adding the username and username header.
            clients[client_socket] = user

            print('Accepted new connection from {}:{}, username: {}'.format(
                *client_address, user['data'].decode('utf-8')))

        # Else the existing socket is sending a message so handling the existing client.
        else:
            # Receiving the message.
            message = receive_message(notified_socket)

            # If no message is accepted then finish the connection.
            if message is False:
                print('Closed connection from: {}'.format(
                    clients[notified_socket]['data'].decode('utf-8')))

                # Removing the socket from the list of the socket.socket()
                sockets_list.remove(notified_socket)

                # Removing the user from the list of users.
                del clients[notified_socket]

                continue

            # Getting the user by using the notified socket, so that the user can be identified.
            user = clients[notified_socket]

            print(
                f'Received message from {user["data"].decode("utf-8")}: {message["data"].decode("utf-8")}')

            # Iterating over the connected clients and broadcasting the message.
            for client_socket in clients:
                # Sending to all except the sender.
                if client_socket != notified_socket:
                    """
                    Reusing the message header sent by the sender, and saving the username header sent by the user when he/she connected.
                    """
                    client_socket.send(
                        user['header'] + user['data'] + message['header'] + message['data'])

```

### Client Side Script
```Python
# importing the necessary modules.
import socket
import sys
import errno

# defining the header length.
HEADER_LENGTH = 10

# defining the IP address and Port Number.
IP = "127.0.0.1"
PORT = 1234

# Getting the name of the client.
my_username = input("Username: ")

"""
Creating a client socket and providing the address family (socket.AF_INET) and type of connection (socket.SOCK_STREAM), i.e. using TCP connection.
"""
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connecting the socket with the IP address and Port Number.
client_socket.connect((IP, PORT))

"""
Setting the connection to a non-blocking state so that the recv() function call will not get blocked. It will return some exceptions only.
"""
client_socket.setblocking(False)

# Setting the username and header.
username = my_username.encode('utf-8')
username_header = f"{len(username):<{HEADER_LENGTH}}".encode('utf-8')

"""
Here, we have encoded the username into bytes, counted the number of bytes, and then prepared a header of fixed size, that we have encoded to bytes as well.
"""

# sending the data.
client_socket.send(username_header + username)




# running an infinite loop to send continuous client requests.
while True:
    # Getting user input.
    message = input(f'{my_username} > ')

    # Sending the non-empty message.
    if message:
        """
        encode the message into bytes, counted the number of bytes, and then prepared a header of fixed size, that we have encoded to bytes as well.
        """
        message = message.encode('utf-8')
        message_header = f"{len(message):<{HEADER_LENGTH}}".encode('utf-8')
        # sending the message.
        client_socket.send(message_header + message)

    try:
        # looping over the received messages and printing them.
        while True:
            # getting the header.
            username_header = client_socket.recv(HEADER_LENGTH)

            # If no header is accepted then finish the connection.
            if not len(username_header):
                print('Connection closed by the server')
                sys.exit()

            # Converting the header to an int value.
            username_length = int(username_header.decode('utf-8').strip())

            # Decoding the received username.
            username = client_socket.recv(username_length).decode('utf-8')

            # Decoding the received message.
            message_header = client_socket.recv(HEADER_LENGTH)
            message_length = int(message_header.decode('utf-8').strip())
            message = client_socket.recv(message_length).decode('utf-8')

            # Printing the message.
            print(f'{username} > {message}')

    except IOError as e:
        # handling the normal error on nonblocking connections.
        if e.errno != errno.EAGAIN and e.errno != errno.EWOULDBLOCK:
            print('Reading error: {}'.format(str(e)))
            sys.exit()

        # If we did not receive anything, then continue.
        continue

    except Exception as e:
        print('Reading error: '.format(str(e)))
        sys.exit()
```
