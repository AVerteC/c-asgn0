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
        <li><a href="#split">split()</a><li>
        <li><a href="#finalexit">finalExit()</a><li>
      </ul>
    </li>
    <li><a href="#performance-considerations">Performance Considerations</a></li>
  </ol>
</details>



## About split

Split takes a delimiter character and a list of files as input. It replaces each instance of the delimiter character with a newline for each of the files in the list, splitting the file contents and printing it out to STDOUT.



## How To Use split

Split's executable needs to be built first, then it can be run with the terminal.



### Building split

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

  This command runs split on all the files listed and uses the delimiter x.
  ```sh
  $ ./split x file1 file2 file3 ...
  ```
  * Using '-' as a filename causes it to read STDIN instead of the file named '-'.
  * Dash can only be used once in the list of input files
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
  
  main() handles error cases and the calling of split() for every file, and also running finalExit() at the end of the program.
  finalExit() exits the program with the last error code produced by the file inputs.
  main() handles the not enough arguments error by checking the count of arguments using the value of argc <=2, because you need 3 arguments in order to start split.
  main() calls openFile() on each file argument, and passes the file descriptor value from openFile() to run split(), and lastly, runs finalExit().
  
  
  
### openFile()

  ```sh
  int openFile(char *filename);
  ```
  
  openFile() handles the opening of the file arguments in
  
  ```sh
  $ ./split (delimiter character) file1 file2 file3 ...
  ```

  openFile() uses open() to open the files in read-only mode, and returns the file descriptor value from open().
  It also handles the error case where the file doesn't exist and the file permissions are denied, errno:2 and errno:13 from open().
  The reference split implementation skips file errors to process all of the files. I implemented this through saving the error codes that open() returns to a global int that saves the last error code for finalExit(). I also pass the file descriptor value as the minimum value of int to mark it as a  file to skip when split() runs with this file descriptor. finalExit() is called after processing all the file inputs to return the last error code to the terminal.



### split()

  ```sh
  int split(int file_descriptor, char *delimiter); 
  ```
  
  split() handles the functionality of replacing the delimiter character with a newline. If there was an error detected by openFile(), split() will read the INT_MIN marker value placed in the file descriptor by openFile() and return out of the function to skip processing the faulty file. 
  
  split() also checks if the delimiter is longer than 1 character by placing the terminal input into a buffer and checking the length of it with strlen(). 
  
  split() uses large buffers that are 4096 bytes long to read large blocks of characters from files. This reduces the amount of times that a file will be accessed, making split() more efficient. In addition, split() uses unsigned char buffers to accommodate binary file input data. I also check the character values for the delimiter replacement with a == comparator which does not rely on formatted c-strings or null characters.
  
  
  
### finalExit()
  ```sh
  int finalExit();
  ```
  This function returns the last error code detected by openFile(), and matches the reference implementation's functionality of sending the last error code detected to the console.

<p align="right">(<a href="#top">back to top</a>)</p>



### Performance Considerations

  The slowest process of my program would be the reading of files and writing to the console. I use large buffer sizes of 4KB to read in large chunks of input files quickly, instead of reading one character of the file at a time. I also use these large buffers to write the results of split() to STDOUT in the terminal.   The use of large buffers cuts down on the number of writes to the terminal. I also do not use malloc() and free() for the read buffer, I used a static unsigned char buffer array instead. This removes the possibility of memory leaks in my program.  

<p align="right">(<a href="#top">back to top</a>)</p>
