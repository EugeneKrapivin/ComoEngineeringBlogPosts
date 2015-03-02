Lately, on a project I work on, arose the need to delete files and folders that are nested in deep heirarchies of folders.

##### Maximum path length limitation
Windows limits the length of any path to 260 characters, more than that will cause file/folder creation problems. Such deep heirarchies (or just long named files) can not be created in a simple manner. However, sometimes they get created, for example in the process of compilation.

##### Scenario
You have to automate a compilation of some java project. Once compilation is done, you have to delete all the by-products of the compilation that you do not need anymore.
You try to delete the folder after extracting what you need, but it doesn't delete.

```
D:\Products\Web\Conduit.Mobile.PHP\tmp\427618da-bfc8-4a44-88d2-b939fadfcde0\2_20150302132731\Library\build\intermediates\classes\release\com\conduit\app\pages\branches\data\BranchesPageDataImpl$BranchesFeedDataImpl$BranchesFeedItemDataImpl$BranchesedItemOpeningHoursDataImpl$BranchesFeedItemOpeningHoursDayDataImpl$BranchesFeedItemOpeningHoursDayHoursDataImpl.class
```

Let me assure you, this file name is 194 characters long, the whole path is 367 characters long, which is way past the microsoft windows limit. This file could not be deleted neither, from windows explorer, nor by command line del. Since the folders are not empty, they could not be deleted as well.

Trying to delete such folders with the `del` command with `/s` (include sub-directories and files) `/q` (quite mode) will return this error:
```
D:\Products\Web\Conduit.Mobile.PHP>del /q /s tmp
The path D:\Products\Web\Conduit.Mobile.PHP\tmp\427618da-bfc8-4a44-88d2-b939fadfcde0\2_20150302141756\Library\build\intermediates\classes\release\com\conduit\app\pages\branches\data\BranchesPageDataImpl$BanchesFeedDataImpl$BranchesFeedItemDataImpl$BranchesFeedItemImagesDataImpl.class is too long.
```
Same behaviour is seen when trying to delete this folder using C#.

##### Unicode paths
Many of the windowsAPI functions provide variations for unicode paths that actaully allow longer paths. Such paths allow up to 32,767 characters.

##### Win32API function
The Windows API, informally WinAPI, is Microsoft's core set of application programming interfaces (APIs) available in the Microsoft Windows operating systems. WinAPI holds various functions for manipulating files and directories, in our solution we'll use three of those:

```C
BOOL WINAPI RemoveDirectory(
  _In_  LPCTSTR lpPathName
);
```
```C
BOOL WINAPI DeleteFile(
  _In_  LPCTSTR lpFileName
);
```
These function have unicode counter-parts that are suffixed with 'W', those we will use.  
Another function we will use is:
```C
DWORD WINAPI GetLastError(void);
```
This function will return us the error code, incase such errors arise during the api calls.

##### Solution
First of all we will have to import these functions to out code from the `kernel32.dll`
```C#
[DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
static extern bool DeleteFileW(string lpFileName);

[DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
static extern bool RemoveDirectoryW(string lpPathName);

[DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
static extern long GetLastError();
```
Now that we can use these functions in out C# code, we will write a simple recursive method called DelTree that will receive a path to a folder, and recursively delete all files and sub-folders, deleting the root folder in the end.
```C#
public static void DelTree(String dir) {
    var files = Directory.EnumerateFileSystemEntries(dir);
    foreach (var file in files) {
        if (Directory.Exists(file)) {
            DelTree(file);
        }
        else {
            if (!DeleteFileW(@"\\?\" + file)) {
                Console.WriteLine("Failed to delete " + (@"\\?\" + file) + " because " + GetLastError());    
            }
        }
    }
    if (!RemoveDirectoryW(@"\\?\" + dir)) {
        Console.WriteLine("Failed to delete " + (@"\\?\" + dir) + " because " + GetLastError());
    }
} 
```
Notice that all paths were prefixed with the `\\?\` string. This prefix signifies that the path is an extended length path.

Using this code as a command-line tool, we've managed to successfully and efficiently remove the trouble making files and folders.

```C#
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.InteropServices;


namespace DelTree
{
    class Program
    {
        [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
        static extern bool DeleteFileW(string lpFileName);

        [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
        static extern bool RemoveDirectoryW(string lpPathName);

        [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
        static extern long GetLastError();

        public static void DelTree(String dir)
        {
            var files = Directory.EnumerateFileSystemEntries(dir);

            foreach (var file in files)
            {
                if (Directory.Exists(file))
                {
                    DelTree(file);
                }
                else
                {
                    if (!DeleteFileW(@"\\?\" + file))
                    {
                        Console.WriteLine("Failed to delete " + (@"\\?\" + file) + " because " + GetLastError());    
                    }
                }
            }

            if (!RemoveDirectoryW(@"\\?\" + dir))
            {
                Console.WriteLine("Failed to delete " + (@"\\?\" + dir) + " because " + GetLastError());
            }
        } 

        static void Main(string[] args)
        {
            if (args.Length != 1)
            {
                Console.WriteLine(Process.GetCurrentProcess().ProcessName + " <dir>");
                return;
            }

            DelTree(Path.GetFullPath(args[0]));
        }
    }
}

```
