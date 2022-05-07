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
    <li><a href="#Design-considerations">Extra Design Considerations</a></li>
  </ol>
</details>


## About httpserver

httpserver takes a log filename and a port number for its input. The port number is of uint16_t type.
httpserver runs infinitely and responds to clients' HTTP messages with a blocking wait system. It supports GET, PUT, and APPEND operations. GET sends file contents to the client. PUT saves data the client sends to a file. APPEND appends to existing files in httpserver's directory.



## How To Use httpserver

httpserver's executable needs to be built first, then it can be run with the terminal.



### Building httpserver

httpserver is compiled with clang and C version `C99` using the flags `-Wall -Wextra -Werror -pedantic`.

* To build all executables for httpserver run:
  ```sh
  $ make
  ```
* To build split only:
  ```sh
  $ make httpserver
  ```
* To clean all generated files:
  ```sh
  $ make clean
  ```

### Running httpserver

This command starts the server with the specified port number.
 ```sh
 $ ./httpserver -l <log_filename> <port number>
 ```
* An example port to use the server with is `8080`.
* The server will error out if the log_filename is forbidden, but create a new file or concatenate the log_filename file otherwise.
* Use `CTRL + C` to stop the server
* When stopping with`CTRL + C`, the server might not properly unbind from the ports
  so connecting to a previously used port can result in this error message: `httpserver: bind error: Address already in use`
* Choose a different port if the server cannot be started with the port you want.

<p align="right">(<a href="#top">back to top</a>)</p>



## Design Overview

My implementation of httpserver has 11 parts:

1. int main(int argc, char *argv[]);
2. bool validate_uri(char *uri);
3. void handle_connection(int connfd);
4. void handle_get(char *resource, int client_socket);
5. void cat(int filefd, int outputfd, size_t filesize);
6. void handle_put(char *resource, unsigned char *initial_body_contents, int initial_body_length,
   int content_length, int client_socket);
7. void handle_append(char *resource, unsigned char *initial_body_contents, int initial_body_length,
   int content_length, int client_socket);
8. void get_send_ok(int content_length, int client_socket);
9. void send_code(int error_code, int client_socket);
10. void print_content(unsigned char *request_buffer, int bytes_read, char *title, bool compact);
11. void log_response(char *method, char *resource, int response_code, int requestID);



### main()

  ```sh
  int main(int argc, char *argv[]);
  ```

main() uses the sample code to create a listen socket to detect and process client inputs. main() also handles the command-line arguments for the port number.
It calls handle_connection() on each connection created to process the HTTP input from clients,
and closes the socket when the request has been processed by handle_connection(). 
main() has a signal handler to close the program gracefully when it gets a SIGTERM signal.
main() initializes a mutex for the logfile and stores the logfilename and FILE stream as global variables.
main() calls fopen() in write mode to create/concatenate the log file. It returns an error if it cannot create the file or access it.
main() closes the logfile afterwards so that log_response() can handle logging operations.



### validate_uri()

  ```sh
  bool validate_uri(char *uri);
  ```

validate_uri() checks if the URI argument is valid based on the HTTP specifications.
* The URI's first character must be a `/`.
* It should not be more than 19 characters long.
* The valid characters allowed are :
* Lowercase `a` through `z`
* Capital `A` through `Z`
* The numbers `0` through `9`, `.`, and `_`.

It returns False if the URI is not valid and True otherwise.



### handle_connection()

  ```sh
  void handle_connection(int connfd);; 
  ```

handle_connections() takes the socket descriptor created in main() and recv()s from it to get the HTTP request from the client.
It does this in a loop, until it reads `\r\n\r\n` and sets a boolean `done` to true to stop looping.
handle_connection() marks the end of the header section and uses it to put the initial body detected into a buffer for the handle_METHOD functions to use.
It uses sscanf() to extract the METHOD, URI, VERSION, and Content-Length from the request.
These values are checked for validity.
* handle_connection() sends `400 Bad Request` when the METHOD is longer than 8 characters, or the VERSION is not `HTTP/1.1`.
* handle_connection() sends `501 Not Implemented` if a METHOD other than GET, PUT, or APPEND is detected.

handle_connection() uses strcmp() to determine if the METHOD is GET, PUT, or APPEND, and calls the corresponding function handle_get(), handle_put(), and handle_append() accordingly.

handle_connection() also passes values to the handle_METHOD functions as well.
* It passes the socket descriptor, and the URI to all the methods.
* It passes Content-Length, the initial body contents received, and the initial body length to PUT and APPEND



### handle_get()

  ```sh
  void handle_get(char *resource, int client_socket);
  ```

This function takes in the resource and the socket descriptor to process a GET request.
It also removes the first slash from the resource name to use with open().
It also checks if the resource name is valid with validate_uri(). It removes the first character of the resource name to use and calls open() in read-only mode.
If it has an error opening the file and gets a file descriptor of `-1`, it checks the errno value and:
* It can send `404 Not Found` when `errno == 2`
* It can send `403 Forbidden` when `errno == 13`
  Then, it uses fstat() to get the file size of the resource file and calls get_send_ok() which sends a HTTP 200 response with a content length header with the file's length to the client.
  It then uses cat() to send the contents of the resource file to the client.

<p align="right">(<a href="#top">back to top</a>)</p>



### cat()

  ```sh
  void cat(int filefd, int outputfd, size_t filesize);
  ```

This function takes a source file descriptor `filefd` and a destination file descriptor `outputfd`, as well as a file size and sends the contents of `filefd` to `outputfd`
It uses read() and recv() to transfer the data from the source to the destination. It doesn't need to open files because the file descriptors are part of this function's arguments.



### handle_put()

  ```sh
  void handle_put(char *resource, unsigned char *initial_body_contents, int initial_body_length, int content_length, int client_socket);
  ```

This function takes in the resource, the initial body contents that come with the header portion of the HTTP request, the content-length, and the socket descriptor to process a PUT request.
The PUT method can save data to new files, and files that already exist.
It can write the data received with the header in the initial recv()s by using the initial_body_length, and the initial_body_contents along with a counter to check the bytes written to file.
It also removes the first slash from the resource name to use with open().
It also keeps track of whether it needs to send `201 Created` or `200 OK` by using two boolean flags that are set by checking the value errno after calling open().
If it encounters errors opening the file, it sends `201 Created`, otherwise it sends `200 OK`.
If it has an error opening the file and gets a file descriptor of `-1`, it checks the errno value and:
handle_put() calls open in truncate mode to overwrite the resource file's contents.
It also checks if the resource name is valid.
It removes the first character of the resource name to use and calls open() in read-only mode.
To save data from the client, handle_put() uses a loop that counts the bytes written with write() and keeps receiving data from the client until the content length counter matches the bytes written counter.
If it encounters any errors while writing to the file, it sends a `500 Internal Server Error`.



### handle_append()

  ```sh
  void handle_put(char *resource, unsigned char *initial_body_contents, int initial_body_length, int content_length, int client_socket);
  ```

This function takes in the resource, the initial body contents that come with the header portion of the HTTP request, the content-length, and the socket descriptor to process a GET request.
The APPEND method cannot append to files that do not already exist.
handle_append() checks for this by checking the value errno after calling open().
If it detects permission errors or file not found errors returns a `404 Not Found`
If it encounters errors opening the file, it sends `201 Created`, otherwise it sends `200 OK`.
If it has an error opening the file and gets a file descriptor of `-1`, it checks the errno value and:
handle_put() calls open in truncate mode to overwrite the resource file's contents.
It also removes the first slash from the resource name to use with open().
It can write the data received with the header in the initial recv()s by using the initial_body_length, and the initial_body_contents along with a counter to check the bytes written to file.
It also checks if the resource name is valid.
It removes the first character of the resource name to use and calls open() in read-only mode.
To save data from the client, handle_put() uses a loop that counts the bytes written with write() and keeps receiving data from the client until the content length counter matches the bytes written counter.
If it encounters any errors while writing to the file, it sends a `500 Internal Server Error`.



### content_length_send_ok()

 ```sh
  void content_length_send_ok(int code, int content_length, int client_socket);
  ```
put_send_ok() sends a custom `HTTP 200 OK` and `HTTP 201 Created` with a Content-Length header field whose value is set to `int content_length`.
This is different from the behavior of sending a `200 OK` with send_code.



### send_code()

 ```sh
  void send_code(int error_code, int client_socket);
  ```
This function takes in a error code and sends the proper HTTP response to the client socket.
In this function sending a `200 OK` will have a `Content-Length `of 3 because the message body for this OK response is `OK\n`.
This function uses sprintf() to format the HTTP response methods properly while changing the actual error message.
This function supports most of the error codes required for this assignmentL
* 200 OK
* 400 Bad Request
* 403 Forbidden
* 404 File Not Found
* 500 Internal Server Error
* 501 Not Implemented



### print_content()

 ```sh
  void print_content(unsigned char *request_buffer, int bytes_read, char *title, bool compact);
  ```


This function writes out the request buffer in human-readable format,
allowing you to be able to read normally unprintable characters like `\r`and `\n` and the space character.



### log_response()

 ```sh
  void log_response(char *method, char *resource, int response_code, int requestID);
  ```


log_response is called after send_code() and content_length_send_ok() to save calls in the proper format to the log file.
When the server sends a response, it must be logged with this format to the logfile: `<METHOD>,<URI>,<response_code>,<RequestID header value>\n`
I also set requestID to 0 by default so if the Request-Id header field is missing when handle_connection() is done processing the headers, Request-Id will be logged as 0 for requests that are unlabeled with the request id.
In order to protect write atomicity, log_response() a mutex lock to prevent threads from writing out of order to the logfile with log_response().
To write to the file, log_response() opens the file in write mode, then uses fprintf(), then fflush(), and finally fclose(), then it unlocks the mutex.



### content_length_send_ok()

 ```sh
  void content_length_send_ok(int code, int content_length, int client_socket);
  ```


put_send_ok() sends a custom `HTTP 200 OK` and `HTTP 201 Created` with a Content-Length header field whose value is set to `int content_length`.
This is different from the behavior of sending a `200 OK` with send_code.



### send_code()

 ```sh
  void send_code(int error_code, int client_socket);
  ```


This function takes in a error code and sends the proper HTTP response to the client socket.
In this function sending a `200 OK` will have a `Content-Length `of 3 because the message body for this OK response is `OK\n`.
This function uses sprintf() to format the HTTP response methods properly while changing the actual error message.
This function supports most of the error codes required for this assignmentL
* 200 OK
* 400 Bad Request
* 403 Forbidden
* 404 File Not Found
* 500 Internal Server Error
* 501 Not Implemented



### Extra Design Considerations
I chose to use large buffer sizes of 2KB to read in large chunks of input from the client quickly, as well as data from files
instead of calling read() on every individual character.
Since main() handles closing the socket, none of my functions need to close the socket.
I also pass state to my functions through the arguments with content length, and especially the initial buffer contents and initial buffer length.
handle_METHODS also close file descriptors when they are done using them before sending their HTTP response back to the client.

For audit logging, I used mutexes to protect the logfile from unordered writes.

<p align="right">(<a href="#top">back to top</a>)</p>