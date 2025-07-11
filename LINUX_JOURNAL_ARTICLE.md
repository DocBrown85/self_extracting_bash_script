# Original Article from Linux Journal

There are 3 parts to the Base Self-Extracting Script:
- The Payload
- The Decompression Script
- The Builder Script
Lets begin by creating our working directory and all the script files we will need.

```bash
jeff:~$ mkdir -vp installer/payload
mkdir: created directory `installer'
mkdir: created directory `installer/payload'
jeff:~$ cd installer/
jeff:~/installer$
```
## The Payload
The payload directory will contain....just that, the payload of your installer. This is the location where you'll put all your tar files, scripts, binaries, etc that you'll want installed onto the new system. For this example I have a tar file containing some text files that I'll want to install into a folder that I create into my home directory. Here is the listing of the tar file.

```bash
jeff:~$ tar tvf files.tar
-rw-r--r-- jeff/jeff        40 2007-12-06 07:53 ./File1.txt
-rw-r--r-- jeff/jeff        92 2007-12-06 07:55 ./File2.txt
jeff:~$
```
Now we must create the installation script that will handle the payload. This script contains any actions that you'd wish to be performed on the installation system, make directories, uncompress files, run system commands, etc. For the example I will create a directory and untar the files into it.

```bash
#!/bin/bash
echo "Running Installer"
mkdir $HOME/files
tar ./files.tar -C $HOME/files
```
Now that we have filled our payload directory with all the files we'd like to install and created the installation script to the the files in their correct location, our directory structure should look like this:

```bash
jeff:~$ find installer/
installer/
installer/payload
installer/payload/installer
installer/payload/files.tar
jeff:~$
```
## The Decompression Script
The Decompression Script does most of the work. The compressed archive of your payload directory will actually be appended onto this script. When ran, the script will remove the archive, decompress it and execute the install script we had created in the previous section.

```bash
#!/bin/bash
echo ""
echo "Self Extracting Installer"
echo ""

export TMPDIR=`mktemp -d /tmp/selfextract.XXXXXX`

ARCHIVE=`awk '/^__ARCHIVE_BELOW__/ {print NR + 1; exit 0; }' $0`

tail -n+$ARCHIVE $0 | tar xzv -C $TMPDIR

CDIR=`pwd`
cd $TMPDIR
./installer

cd $CDIR
rm -rf $TMPDIR

exit 0

__ARCHIVE_BELOW__
```
Ok, now lets go through this script step by step. After the bit of printout at the begining, the first real line of work creates a temporary directory for us to decompress our payload into initially before we install it.
```
 export TMPDIR=`mktemp -d /tmp/selfextrac.XXXXXX` 
```
The
```
-d
```
flag tells mktemp to create a directory rather than a file. The parameter at the end,
```
/tmp/selfextract.XXXXXX
```
is a template of the name of the directory we are going to create. The X's are replaced with random characters to generate a random name, just incase we happen to be installing two things at the same time. The next line in the script,
```
ARCHIVE=`awk '/^__ARCHIVE_BELOW__/ {print NR + 1; exit 0; }' $0`
```
find the line number where the archive starts in the script. The first parameter given to awk
```
/^__ARCHIVE_BELOW__/
```
tells it to search for a regular expression "A line starting with the characters '__ARCHIVE_BELOW__'". Each line is read and until the regular expression is satisfied. The next parameter
```
{print NR + 1; exit 0; }
```
tells awk to print out the number of records (number of lines read) plus 1, then quit. The third parameter
```
$0
```
is the first argument when running this script, which happens to be the script's name ($0 = ./decompress). We will now seperate the archive from the script and decompress it into the temporary directory we have created.
```
tail -n+$ARCHIVE $0 | tar xzv -C $TMPDIR
```
tail prints out the end of a file. The parameter
```
-n+$ARCHIVE
```
tells tail to start at line number we just read in the previous command, and print til the end of the file.
```
$0
```
again is the name of this script file. This output is then piped to tar where it is
```
z
```
ungzipped, and
```
x
```
unarchived into
```
-C $TMPDIR
```
the temporary directory. The next section, we remember our current directory,
```
CDIR=`pwd`
```
and step into the temporary directory:
```
cd $TMPDIR
```
From here we run the script we created in the Payload section. After the script has been executed, we revert back to our previous directory
```
cd $CDIR
```
and remove the temporary directory
```
rm -rf $TMPDIR
```
The last two lines in this script are very important. First the line
```
exit 0
```
causes the script to stop executing. If we forget this line BASH would try to execute the binary archive attached at the bottom and would cause problems. The very last line
```
__ARCHIVE_BELOW__
```
tells awk that the binary archive starts on the next line. Make sure that this is the last line, no extra empty lines below this one. Now that we have finished creating the Decompression script our directory should looke like this:
```bash
jeff:~$ find installer/
installer/
installer/build
installer/payload
installer/payload/installer
installer/payload/files.tar
installer/decompress
jeff:~$
```
## The Builder Script
The last section of this installer is the script that builds the self-extracting script. This script compresses the payload and then adds the decompresion script to the archive.
```
#!/bin/bash
cd payload
tar cf ../payload.tar ./*
cd ..

if [ -e "payload.tar" ]; then
    gzip payload.tar

    if [ -e "payload.tar.gz" ]; then
        cat decompress payload.tar.gz > selfextract.bsx
    else
        echo "payload.tar.gz does not exist"
        exit 1
    fi
else
    echo "payload.tar does not exist"
    exit 1
fi

echo "selfextract.bsx created"
exit 0
```
This script archives the payload directory
```
tar cf ../payload.tar ./*
```
and compresses it using gzip
```
gzip payload.tar
```
Next the script concatenates the decompress script with the compressed payload
```
cat decompress payload.tar.gz > selfextract.bsx
```
With all the scripts completed our directory should look like this:
```
jeff:~$ find installer/
installer/
installer/build
installer/payload
installer/payload/installer
installer/payload/files.tar
installer/decompress
jeff:~$
```
## Test it out
Now that we have created everything, we can run the scripts and test it all out. Firstly run the build script. Your output should look as follows:
```
jeff:~/installer$ ./build
selfextract.bsx created
jeff:~/installer$ ls
build  decompress  payload  payload.tar.gz  selfextract.bsx
jeff:~/installer$
```
Now we have our bash script with the archive attached (selfextract.bsx). Run this script and you should see the following output:
```
jeff:~/installer$ ./selfextract.bsx

Self Extracting Installer

./files.tar
./installer
Running Installer
./File1.txt
./File2.txt
jeff:~/installer$ find /home/jeff/files
/home/jeff/files
/home/jeff/files/File1.txt
/home/jeff/files/File2.txt
jeff:~/installer$
```
If you would like the installer to install something else, just place the files in the payload directory and modify the installer script to perform the proper action.


