Has this ever happened to you? I was working on a project recently where I had to delete files and folders that are nested in deep heirarchies of folders.

##### Maximum Path Length Limitation
Windows limits the length of any path to 260 characters. Any more than that will cause file/folder creation problems. It's not simple to create deep heirarchies (or just long-named files). Sometimes, though, they can be created during the compilation process.

##### Scenario
To do this, first automate the compilation of a Java project. When the compilation is done, delete any by-products of the compilation that you don't need. If you try to delete the folder after extracting what you need, it won't delete.

`D:\Products\Web\Conduit.Mobile.PHP\tmp\427618da-bfc8-4a44-88d2-b939fadfcde0\2_20150302132731\Library\build\intermediates\classes\release\com\conduit\app\pages\branches\data\BranchesPageDataImpl$BranchesFeedDataImpl$BranchesFeedItemDataImpl$BranchesFeedItemOpeningHoursDataImpl$BranchesFeedItemOpeningHoursDayDataImpl$BranchesFeedItemOpeningHoursDayHoursDataImpl.class`

Let me assure you, this file name is 194 characters long. The whole path is 367 characters long, which is way past the Windows limit. You can't delete this file from Windows Explorer or command line del, either. And since the folders aren't empty, they can't be deleted. Problem.

##### Unicode Paths
Many of the Windows API functions provide variations for unicode paths that actually allow longer paths, up to 32,767 characters.

##### Win32API Function
Windows API, informally WinAPI, is Microsoft's core set of APIs, available in the Microsoft Windows operating systems. WinAPI has various functions for manipulating files and directories. In our solution, we'll use three:

```language-c
BOOL WINAPI RemoveDirectory(
  _In_  LPCTSTR lpPathName
);
```
```language-c
BOOL WINAPI DeleteFile(
  _In_  LPCTSTR lpFileName
);
```
We'll use the unicode counterparts of these functions that are suffixed with 'W'.  
Another function we will use is:
```language-c
DWORD WINAPI GetLastError(void);
```
If errors arise during the API calls, this function will return the error code.

##### Solution
First of all, we import these functions to our code from the `kernel32.dll`
```language-csharp
[DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
static extern bool DeleteFileW(string lpFileName);

[DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
static extern bool RemoveDirectoryW(string lpPathName);

[DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
static extern long GetLastError();
```
Now that we can use these functions in our C# code, we'll write a simple recursive method called DelTree, which will receive a path to a folder and recursively delete all files and subfolders, deleting the root folder in the end.
```language-csharp
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
