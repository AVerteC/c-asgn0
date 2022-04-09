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
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>




## About split

Split takes a delimiter character and a list of files as input. It replaces each instance of the delimiter character with a newline, splitting the file.

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
  * Using '-' as a filename causes it to read stdin instead of the file named '-'.
  * Dash can only be used once in the list of input files
  * split works on binary input files
  * split only supports single-character delimiters  

<p align="right">(<a href="#top">back to top</a>)</p>

## Design Overview
  
  My implementation of split has 4 parts
  1. int main(int argc, char *argv[]);
  2. int openFile(char *filename);
  3. int split(int file_descriptor, char *delimiter);
  4. int finalExit();

### main()
  ```sh
  int main(int argc, char *argv[]);
  ```
  main handles error cases and the calling of split() for every file, and also running finalExit() at the end of the program.
  finalExit() exits the program with the last error code produced by the file inputs.
  main handles the not enough arguments error by checking the count of arguments using the value of argc <=2, because you need 3 arguments in order to start split.
  main calls openFile() on each file argument, and passes the file descriptor value from openFile() to run split(), and lastly, runs finalExit().
  
### openFile()
  ```sh
  int openFile(char *filename);
  ```
  
  
  
  
  
  
  
### int split(int file_descriptor, char *delimiter); 

### int finalExit();






<p align="right">(<a href="#top">back to top</a>)</p>






<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<p align="right">(<a href="#top">back to top</a>)</p>



<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE.txt` for more information.

<p align="right">(<a href="#top">back to top</a>)</p>



<!-- CONTACT -->
## Contact

Your Name - [@your_twitter](https://twitter.com/your_username) - email@example.com

Project Link: [https://github.com/your_username/repo_name](https://github.com/your_username/repo_name)

<p align="right">(<a href="#top">back to top</a>)</p>



<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

Use this space to list resources you find helpful and would like to give credit to. I've included a few of my favorites to kick things off!

* [Choose an Open Source License](https://choosealicense.com)
* [GitHub Emoji Cheat Sheet](https://www.webpagefx.com/tools/emoji-cheat-sheet)
* [Malven's Flexbox Cheatsheet](https://flexbox.malven.co/)
* [Malven's Grid Cheatsheet](https://grid.malven.co/)
* [Img Shields](https://shields.io)
* [GitHub Pages](https://pages.github.com)
* [Font Awesome](https://fontawesome.com)
* [React Icons](https://react-icons.github.io/react-icons/search)

<p align="right">(<a href="#top">back to top</a>)</p>



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/othneildrew/Best-README-Template.svg?style=for-the-badge
[contributors-url]: https://github.com/othneildrew/Best-README-Template/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/othneildrew/Best-README-Template.svg?style=for-the-badge
[forks-url]: https://github.com/othneildrew/Best-README-Template/network/members
[stars-shield]: https://img.shields.io/github/stars/othneildrew/Best-README-Template.svg?style=for-the-badge
[stars-url]: https://github.com/othneildrew/Best-README-Template/stargazers
[issues-shield]: https://img.shields.io/github/issues/othneildrew/Best-README-Template.svg?style=for-the-badge
[issues-url]: https://github.com/othneildrew/Best-README-Template/issues
[license-shield]: https://img.shields.io/github/license/othneildrew/Best-README-Template.svg?style=for-the-badge
[license-url]: https://github.com/othneildrew/Best-README-Template/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/othneildrew
[product-screenshot]: images/screenshot.png
