Has this ever happened to you? I was working on a project recently where I had to delete files and folders that are nested in deep hierarchies of folders.

##### Maximum Path Length Limitation
Windows limits the length of any path to 260 characters. Any more than that will cause file/folder manipulation problems. It's not trivial to create deep hierarchies (or just long-named files). Sometimes, though, they can be created during a process, such as the compilation process.

##### Scenario
During the compilation process of an android project, some files are created in a very deeply nested hierarchy, with long names. Deleting any by-products of the compilation will be seamless in most cases. In some cases, however, windows won't let you do it.

For example, in my case, the compilation process created the following file:
`D:\Products\Web\Conduit.Mobile.PHP\tmp\427618da-bfc8-4a44-88d2-b939fadfcde0\2_20150302132731\Library\build\intermediates\classes\release\com\conduit\app\pages\branches\data\BranchesPageDataImpl$BranchesFeedDataImpl$BranchesFeedItemDataImpl$BranchesFeedItemOpeningHoursDataImpl$BranchesFeedItemOpeningHoursDayDataImpl$BranchesFeedItemOpeningHoursDayHoursDataImpl.class`

Let me assure you, this file name is 194 characters long. The whole path is 367 characters long, which is way past the Windows limit. You can't delete this file from Windows Explorer or command line `del`, either. Since the folders aren't empty, they can't be deleted.

##### Unicode Paths
Many of the Windows API functions provide variations for Unicode paths that actually allow longer paths (up to 32,767 characters).

##### Win32API Function
Windows API, informally WinAPI, is Microsoft's core set of APIs, available in the Microsoft Windows operating systems. WinAPI has various functions for manipulating files and directories. In our solution, we'll use three:

```language-c
BOOL WINAPI RemoveDirectory(
  _In_  LPCTSTR lpPathName
);
```
[(reference)](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365488%28v=vs.85%29.aspx)
```language-c
BOOL WINAPI DeleteFile(
  _In_  LPCTSTR lpFileName
);
```
[(reference)](https://msdn.microsoft.com/en-us/library/windows/desktop/aa363915%28v=vs.85%29.aspx)

We'll use the Unicode counterparts of these functions that are suffixed with 'W'. These functions accept an extended length path, prefixed with `\\?\`. For example, `\\?\D:\very long path`.

Another function we'll use is:
```language-c
DWORD WINAPI GetLastError(void);
```
If errors arise during the API calls, this function will return an [error code](https://msdn.microsoft.com/en-us/library/windows/desktop/ms681381(v=vs.85).aspx).

##### Solution
First of all, we import these functions to our code from `kernel32.dll`
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
