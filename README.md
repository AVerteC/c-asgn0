<div id="top"></div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-split">About split</a>
    </li>
    <li>
      <a href="#how-to-use-split">Getting Started</a>
      <ul>
        <li><a href="#building-split">Building split</a></li>
        <li><a href="#running-split">Running split</a></li>
      </ul>
    </li>
    <li>
      <a href="#design-overview">Design Overview</a>
      <ul>
        <li><a href="#main">main()</a></li>
        <li><a href="#openfile">openFile()</a></li>
        <li><a href="#split">split()</a></li>
        <li><a href="#finalexit">finalExit()</a></li>
      </ul>
    </li>
    <li><a href="#performance-and-optimization-considerations">Performance and Optimization Considerations</a></li>
  </ol>
</details>



## About split

Split takes a delimiter character and a list of files as input. It replaces each instance of the delimiter character with a newline for each of the files in the list, splitting the file contents and printing it out to STDOUT.



## How To Use split

Split's executable needs to be built first, then it can be run with the terminal.



### Building split

split is compiled with C version `C99` using the flags `-Wall -Wextra -Werror -pedantic`.

* To build all executables for split run:
  ```sh
  $ make
  ```
* To build split only:
  ```sh
  $ make split
  ```
* To clean all generated files:
  ```sh
  $ make clean
  ```



### Running split

  This command uses the delimiter and runs split on all the files.
  ```sh
  $ ./split <delimiter> <file1> <file2> <file3> ... <file n>
  ```
  * You can use a dash `-` instead of a filename, allowing split to read STDIN as an input.
  * Assume dash `-` is only used once in the list of input files
  * split works on binary input files
  * split only supports single-character delimiters  

<p align="right">(<a href="#top">back to top</a>)</p>



## Design Overview
  
  My implementation of split has 4 parts:
  1. int main(int argc, char *argv[]);
  2. int openFile(char *filename);
  3. int split(int file_descriptor, char *delimiter);
  4. int finalExit();



### main()

  ```sh
  int main(int argc, char *argv[]);
  ```
  
  main() calls split() on every file, runs finalExit() at the end of the program, and handles errors from invalid inputs.
  finalExit() exits the program with the last error code produced by the file inputs.
  main() also checks if the delimiter is longer than 1 character by checking the length of it with strlen(). 
  main() handles the following input validations:
  * `Not enough arguments`: by checking the count of arguments using the value of argc <=2, because you need at least 3 arguments in order to start split. These arguments include `./split <delimiter> <file>`.


  main() calls openFile() on each file argument and passes the file descriptor value from openFile() to run split(), and lastly, runs finalExit().
  
  
  
### openFile()

  ```sh
  int openFile(char *filename);
  ```
  
  openFile() handles the opening of the file arguments in
  
  ```sh
  $ ./split (delimiter character) file1 file2 file3 ...
  ```

  openFile() uses open() to open the files in read-only mode and returns the file descriptor value from open().
  It also handles the error case where the file doesn't exist and the file permissions are denied, errno:2 and errno:13 from open().
  
  
  The reference split implementation skips file errors to process all of the files and returns the last error code detected. I implemented the same functionality by saving the error codes that open() returns to a global int that saves the last error code. I also pass the file descriptor value as the minimum value of int to mark it as a file to skip when split() runs with a file descriptor belonging to a faulty file. 



### split()

  ```sh
  int split(int file_descriptor, char *delimiter); 
  ```
  
  split() handles the functionality of replacing the delimiter character with a newline. If there was an error detected by openFile(), split() will read the INT_MIN marker value placed in the file descriptor by openFile() and return out of the function to skip processing the faulty file. 
  
  
  split() uses large buffers that are 4096 bytes long to read large blocks of characters from files. This reduces the amount of times that a file will be accessed, making split() more efficient. In addition, split() uses unsigned char buffers to accommodate binary file input data. I also check the character values for the delimiter replacement with a == comparator which allows for both signed and unsigned characters as delimiters. split() closes the file descriptor after it is done reading from the file, so that the program doesn't run out of file descriptors.
  
  
  
### finalExit()

  ```sh
  int finalExit();
  ```
  
  This function returns the last error code detected by openFile(), and matches the reference implementation's functionality of sending the last error code detected to the console.

<p align="right">(<a href="#top">back to top</a>)</p>



### Performance and Optimization Considerations

  The slowest process of my program is the reading of files and writing to the console. I chose to use large buffer sizes of 4KB to read in large chunks of input files quickly, instead of calling read() on every individual character of the file. I also use these large buffers to write the results of split() to STDOUT in the terminal. The use of large output buffers optimizes the number of writes to the terminal. Additionally, I used a static unsigned char buffer array instead of dynamically allocating memory with malloc() and free() for the read buffer. This allows my program to support binary files, while also removing the possibility of memory leaks in my program. My program also closes file descriptors after it is done processing files with split(), preventing the program from running out of file descriptors. 

<p align="right">(<a href="#top">back to top</a>)</p>
