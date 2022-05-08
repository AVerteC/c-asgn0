<div id="top"></div> 

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-httpserver">About httpserver</a>
    </li>
    <li>
      <a href="#how-to-use-httpserver">How To Use httpserver</a>
      <ul>
        <li><a href="#building-httpserver">Building httpserver</a></li>
        <li><a href="#running-httpserver">Running httpserver</a></li>
      </ul>
    </li>
    <li>
      <a href="#design-overview">Design Overview</a>
      <ul>
        <li><a href="#main">main()</a></li>
        <li><a href="#validate_uri">validate_uri()</a></li>
        <li><a href="#handle_connection">handle_connection()</a></li>
        <li><a href="#handle_get">handle_get()</a></li>
        <li><a href="#cat">cat()</a></li>
        <li><a href="#handle_put">handle_put()</a></li>
        <li><a href="#handle_append">handle_append()</a></li>
        <li><a href="#content_length_send_ok">content_length_send_ok()</a></li>
        <li><a href="#send_code">send_code()</a></li>
        <li><a href="#print_content">print_content()</a></li>
        <li><a href="#log_response">log_response()</a></li>
      </ul>
    </li>
    <li><a href="#Extra-design-considerations">Extra Design Considerations</a></li>
  </ol>
</details>


## About httpserver

httpserver takes a log filename and a port number as command line arguments. The port number is of uint16_t type.
httpserver runs indefinitely and responds to clients' HTTP requests with a blocking wait system. It supports GET, PUT, and APPEND operations. GET sends file contents to the client. PUT saves data the client sends to a file. APPEND appends data to an existing file in httpserver's directory. httpserver also supports logging responses to requests that have the Request-Id field.



## How to Use httpserver

httpserver's executable needs to be built first, then it can be run using the terminal.



### Building httpserver

httpserver is compiled with clang and C version `C99` using the flags `-Wall -Wextra -Werror -pedantic`.

* To build all executables for httpserver run:
  ```sh
  $ make
  ```
* To build httpserver only:
  ```sh
  $ make httpserver
  ```
* To clean all generated files:
  ```sh
  $ make clean
  ```

### Running httpserver

This command starts the server using the specified log file with the specified port number.
 ```sh
 $ ./httpserver -l <log_filename> <port number>
 ```
* An example port to use the server with is `8080`.
* The server will error out if the `log_filename` is forbidden, but will create a new file or concatenate the file called `log_filename` otherwise.
* Use `CTRL + C` to stop the server.
* When stopping with`CTRL + C`, the server might not properly unbind from the ports
  so connecting to a previously used port can result in this error message: `httpserver: bind error: Address already in use`
* Choose a different port if the server cannot be started with the port you want.

<p align="right">(<a href="#top">back to top</a>)</p>



## Design Overview

My implementation of httpserver has 11 parts:

1. [int main(int argc, char *argv[])](#main)
2. [bool validate_uri(char *uri)](#validate_uri)
3. [void handle_connection(int connfd)](#handle_connection)
4. [void handle_get(char *resource, int client_socket)](#handle_get)
5. [void cat(int filefd, int outputfd, size_t filesize)](#cat)
6. [void handle_put(char *resource, unsigned char *initial_body_contents, int initial_body_length,
   int content_length, int client_socket)](#handle_put)
7. [void handle_append(char *resource, unsigned char *initial_body_contents, int initial_body_length,
   int content_length, int client_socket)](#handle_append)
8. [void content_length_send_ok(int content_length, int client_socket)](#content_length_send_ok)
9. [void send_code(int error_code, int client_socket)](#send_code)
10. [void print_content(unsigned char *request_buffer, int bytes_read, char *title, bool compact)](#print_content)
11. [void log_response(char *method, char *resource, int response_code, int requestID)](#log_response)



### main()

  ```sh
  int main(int argc, char *argv[]);
  ```

main() uses the sample code to create a listen socket to detect and process client inputs. main() also handles the command-line arguments for the port number.
It calls handle_connection() on each connection created to process the HTTP input from clients,
and closes the socket after handle_connection() processes the request. 
main() has a signal handler to close the program gracefully when it gets a SIGTERM signal.
main() initializes a mutex for the logfile and stores the logfilename and FILE stream as global variables.
main() calls fopen() in write mode to create/concatenate the log file. It returns an error if it cannot create the file or access it.
main() closes the logfile afterwards so that log_response() can handle logging operations.

<p align="right">(<a href="#top">back to top</a>)</p>



### validate_uri()

  ```sh
  bool validate_uri(char *uri);
  ```

validate_uri() checks if the URI argument is valid based on the HTTP specifications.
* The URI's first character must be a `/`.
* It should not be more than 19 characters long.
The valid characters allowed are:
* Lowercase `a` through `z`
* Capital `A` through `Z`
* The numbers `0` through `9`
* The symbols `.`, and `_`.

It returns False if the URI is not valid and True otherwise.

<p align="right">(<a href="#top">back to top</a>)</p>



### handle_connection()

  ```sh
  void handle_connection(int connfd); 
  ```

handle_connections() takes the socket descriptor created in main() and recv()s from it to get the HTTP request from the client.
It does this in a loop, until it reads `\r\n\r\n` and sets a boolean `done` to true to stop looping.
handle_connection() marks the end of the header section and uses it to put the initial body detected into a buffer for the handle_METHOD functions to use.
It uses sscanf() to extract the METHOD, URI, VERSION, and Content-Length from the request.
These values are checked for validity.
* handle_connection() sends `400 Bad Request` when the METHOD is longer than 8 characters, or the VERSION is not `HTTP/1.1`.
* handle_connection() sends `501 Not Implemented` if a METHOD other than GET, PUT, or APPEND is detected.

handle_connection() uses strcmp() to determine if the METHOD is GET, PUT, or APPEND, and calls the corresponding function handle_get(), handle_put(), and handle_append() accordingly.


handle_METHOD refers to handle_GET, handle_PUT, and handle_APPEND.
handle_connection() also passes values to the handle_METHOD functions as well.
* It passes the socket descriptor, and the URI to all the methods.
* It passes Content-Length, the initial body contents received, and the initial body length to PUT and APPEND

<p align="right">(<a href="#top">back to top</a>)</p>



### handle_get()

  ```sh
  void handle_get(char *resource, int client_socket);
  ```

This function takes in the resource and the socket descriptor to process a GET request.
It also removes the first slash from the resource name to use with open().
It also checks if the resource name is valid with validate_uri(). It removes the first character of the resource name to use and calls open() in read-only mode.
If the filename is invalid, it sends a `400 Bad Request` to the client.
If it has an error opening the file and gets a file descriptor of `-1`, it checks the errno value and:
* It can send `404 Not Found` when `errno == 2`
* It can send `403 Forbidden` when `errno == 13`
Then it uses fstat() to get the file size of the resource file and calls get_send_ok() which sends a HTTP 200 response with a `Content-Length` header field with the file's length to the client.
handle_get() uses cat() to send the contents of the resource file to the client.

<p align="right">(<a href="#top">back to top</a>)</p>



### cat()

  ```sh
  void cat(int filefd, int outputfd, size_t filesize);
  ```

This function takes a source file descriptor `filefd` and a destination file descriptor `outputfd`, as well as a file size and sends the contents of `filefd` to `outputfd`.
It uses read() and recv() to transfer the data from the source to the destination. It doesn't need to open files because the file descriptors are part of this function's arguments.

<p align="right">(<a href="#top">back to top</a>)</p>



### handle_put()

  ```sh
  void handle_put(char *resource, unsigned char *initial_body_contents, int initial_body_length, int content_length, int client_socket);
  ```

This function takes in the resource, the initial body contents that come with the header portion of the HTTP request, the content-length, and the socket descriptor to process a PUT request.
The PUT method can save data to a new or existing file.
It can write the data received with the header in the initial recv()s by using the initial_body_length, and the initial_body_contents along with a counter to check the bytes written to file.


It also keeps track of whether it needs to send `201 Created` or `200 OK` by using two boolean flags that are set by checking the errno value if open() returns a negative file descriptor. If it encounters errors opening the file, it sends `201 Created`, otherwise it sends `200 OK`.

handle_put() checks if the resource name is valid, then calls open() in truncate mode to overwrite the resource file's contents.
It removes the first character of the resource name `/` to get the actual filename to call open() with.
If the resource name is invalid, it sends a `400 Bad Request` to the client.

To save data from the client, handle_put() uses a loop that counts the bytes written with write() and keeps receiving data from the client until the content length counter matches the bytes written counter.

If it encounters any errors while writing to the file, it sends a `500 Internal Server Error`.

<p align="right">(<a href="#top">back to top</a>)</p>



### handle_append()

  ```sh
  void handle_put(char *resource, unsigned char *initial_body_contents, int initial_body_length, int content_length, int client_socket);
  ```

This function takes in the resource, the initial body contents that come with the header portion of the HTTP request, the content-length, and the socket descriptor to process a APPEND request.
The APPEND method can only append to existing files that the server has write permissions for.

handle_append() checks for this by checking the errno value if open() returns a negative file descriptor.
If it has an error opening the file and gets a file descriptor of `-1`, it checks the errno value and:
* It can send `404 Not Found` when `errno == 2`
* It can send `403 Forbidden` when `errno == 13`

handle_append() also checks if the filename is valid with validate_uri(). If the filename is valid, it removes the first character of resource, `/` and calls open() on the file in append mode.
If the filename is invalid, it sends a `400 Bad Request` to the client.
It can write the data received with the header in the initial recv()s by using the initial_body_length, and the initial_body_contents along with a counter to check the bytes written to file.

To save data from the client, handle_put() uses a loop that counts the bytes written with write() and keeps receiving data from the client until the content length counter matches the bytes written counter.

If it encounters any errors while writing to the file, it sends a `500 Internal Server Error`.

<p align="right">(<a href="#top">back to top</a>)</p>



### content_length_send_ok()

 ```sh
  void content_length_send_ok(int code, int content_length, int client_socket);
  ```
content_length_send_ok() sends a custom `HTTP 200 OK` and `HTTP 201 Created` with a `Content-Length` header field whose value is set to `int content_length`.
This is different from the behavior of sending a `200 OK` with send_code, which sends a `200 OK` response with the content length of 3 because the message body is `OK\n`.

<p align="right">(<a href="#top">back to top</a>)</p>



### send_code()

 ```sh
  void send_code(int error_code, int client_socket);
  ```
This function takes in a error code and sends the proper HTTP response to the client socket.
In this function sending a `200 OK` will have a `Content-Length `of 3 because the message body for this OK response is `OK\n`.
This function uses sprintf() to format the HTTP response methods properly while changing the actual error message.
This function supports all of the error codes in this assignment. However, because this function doesn't take content-length as an argument, to send the 200 OK and 201 Created responses for PUT and APPEND, I used content_length_send_ok().
send_code() supports these HTTP responses.
* 200 OK
* 400 Bad Request
* 403 Forbidden
* 404 File Not Found
* 500 Internal Server Error
* 501 Not Implemented

<p align="right">(<a href="#top">back to top</a>)</p>



### print_content()

 ```sh
  void print_content(unsigned char *request_buffer, int bytes_read, char *title, bool compact);
  ```


This function prints out the `request_buffer` in human-readable format to stdout,
allowing you to be able to see normally unprintable characters like `\r`and `\n` and the space character.

<p align="right">(<a href="#top">back to top</a>)</p>

### log_response()

 ```sh
  void log_response(char *method, char *resource, int response_code, int requestID);
  ```


log_response is called after send_code() and content_length_send_ok() to save the server's responses to the client in the proper format to the log file.
When the server sends a response, it must be logged with this format to the logfile: `<METHOD>,<URI>,<response_code>,<RequestID header value>\n` .
I also set requestID to 0 by default so if the Request-Id header field is missing when handle_connection() is done processing the headers, Request-Id will be logged as 0 for requests that are missing the Request-Id header field.
In order to protect write atomicity, log_response() uses a mutex lock to prevent threads from writing out of order to the logfile with log_response().
When writing to the file, log_response() gets the mutex lock, fopens the file in append mode, then uses fprintf(), fflush(), fclose(), and then finishes by unlocking the mutex.

<p align="right">(<a href="#top">back to top</a>)</p>



### Extra Design Considerations
I used large buffer sizes of 2KB to read in large chunks of input and file data quickly, instead of calling read() on every byte.
Since main() handles closing the socket, none of my functions need to close the socket.
In order to avoid dynamically allocating memory for buffers in my functions, all of my functions take in buffers by reference.
To prevent the program from running out of file descriptors, my functions close the resource file descriptors when they are done using them before sending a response back to the client.
For audit logging, I used a mutex lock to give the logfile coherency.

<p align="right">(<a href="#top">back to top</a>)</p>
