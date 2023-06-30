---
layout: post
title: What is the unit of work pattern & how to use it
date: 2023-06-29
---

## Why this article?

I've worked most of my career with PHP and as I started picking up C# I kept seeing articles/tutorials related to the 'Unit of work pattern'.

If I'm honest I struggled a bit to really understand it when reading online, nearly every article seemed to be just wrapping some kind of 'DbContext' class and calling functions within it, I didn't see the value and it didn't really give me a deep understanding of the pattern.

Even looking at the [Microsoft documentation](https://learn.microsoft.com/en-us/aspnet/mvc/overview/older-versions/getting-started-with-ef-5-using-mvc-4/implementing-the-repository-and-unit-of-work-patterns-in-an-asp-net-mvc-application) you can see it's just wrapping DBContext without giving you a better understanding of the unit of work pattern.

So in this post I will explain implementing your own unit of work which will hopefully give you a better understanding of the pattern along with some examples in both C# and PHP.

## Definition of unit of work pattern

The clearest definition of the unit of work pattern I found was from one of my favourite technical patterns / practices writer [Martin Fowler](https://martinfowler.com).

He explains the unit of work pattern as:

> Maintains a list of objects affected by a business transaction and coordinates the writing out of changes and the resolution of concurrency problems.
>
> -- <cite>Martin Fowler</cite>

The idea of the pattern is to prevent concurrency problems by orchestrating a list of operations once everything is completed.

Essentially aggregate all your operations then execute them all at the end of the unit of work.

This might sound familiar if you have worked with database transactions and things like 'commit' & 'rollback'.

## A simple example

Let's say we have an application that interacts with the file system.

The application can perform multiple `Add` & `Delete` throughout it's execution.

### Requirements

- Only writes/deletes to the file system at the end of execution
- You can't add a file if it already exists
- You can't delete a non existant file
- Must have rollback feature with compensating operations

### Lets build

#### Defining FileSystemUnitOfWork

Here is the interface of the unit of work.

```
Add(string fileName, string fileContents): void
Delete(string fileName): void
Execute(): void
```

Our first method allows us to add a file and define it's content.
The second deletes a given file from the file system.
The final executes everything and completes our unit of work.

In the implementation what we will do is record a buffer of the write and delete operations and essentially queue them up in a dictionary in C# or an array in PHP.

When the `Execute()` method is called it will interate over the queued writes and persist them into the file system and then do the a similar thing but rather than write it will delete.

The implemention of `Execute()` will be wrapped in a try/catch and if anything fails, then an internal `Rollback()` method will be called which will do something called a `compensation operation` basically doing the inverse of what was requested orginally, it will delete newly created files and then re-add deleted files.

#### Code example 
Here is an example the unit of work pattern for what we are trying to achieve, I have full repository examples further down.

{% tabs unitofwork %}

{% tab unitofwork php %}
```php
<?php
declare(strict_types = 1);

final class FileSystemUnitOfWork {

    private array $_fileContentBuffer = [];
    private array $_deleteBuffer = [];

    public function __construct(private FileSystem $_fileSystem)
    {
    }

    public function Add(string $fileName, string $content): void
    {
        if ($this->_fileSystem->Exists($fileName)) {
            throw new Exception("File exists already");
        }

        $this->_fileContentBuffer[$fileName] = $content;
    }

    public function Delete(string $fileName): void
    {
        if (array_key_exists($fileName, $this->_fileContentBuffer)) {
            unset($this->_fileContentBuffer[$fileName]);
            return;
        }

        $fileContent = $this->_fileSystem->TryReadFile($fileName);

        if ($fileContent === null) {
            throw new Exception("File doesn't exist or not readable");
        }

        $this->_deleteBuffer[$fileName] = $fileContent;
    }

    public function Execute(): void
    {
        try {
            foreach ($this->_fileContentBuffer as $fileName => $fileContent) {
                $this->_fileSystem->WriteFile($fileName, $fileContent);
            }

            foreach (array_keys($this->_deleteBuffer) as $fileName) {
                $this->_fileSystem->Delete($fileName);
            }

        } catch (Exception $e) {
            $this->_rollback();
            throw $e;
        }

        $this->_fileContentBuffer = [];
        $this->_deleteBuffer = [];
    }

    private function _rollback(): void
    {
        echo "Performing rollback\n";
        foreach (array_keys($this->_fileContentBuffer) as $fileName) {
            $this->_fileSystem->Delete($fileName);
        }

        foreach ($this->_deleteBuffer as $fileName => $fileContent) {
            if ($this->_fileSystem->Exists($fileName) === false) {
                $this->_fileSystem->WriteFile($fileName, $fileContent);
            }
        }
    }
}
```
{% endtab %}

{% tab unitofwork C# %}
```csharp
namespace UnitOfWorkPatternExample;

public sealed class FileSystemUnitOfWork
{
    private readonly FileSystem _fileSystem;
    
    private Dictionary<string, string> _fileContentBuffer = new();
    private Dictionary<string, string> _deleteBuffer = new();
    
    public FileSystemUnitOfWork(FileSystem fileSystem)
    {
        _fileSystem = fileSystem;
    }
    
    public void Add(string fileName, string content)
    {
        if (_fileSystem.Exists(fileName))
        {
            throw new Exception("File exists already");
        }

        _fileContentBuffer[fileName] = content;
    }

    public async Task DeleteAsync(string fileName)
    {
        if (_fileContentBuffer.ContainsKey(fileName))
        {
            _fileContentBuffer.Remove(fileName);
            return;
        }

        var fileContent = await _fileSystem.TryReadAllTextAsync(fileName);
        
        if (fileContent == null)
        {
            throw new Exception("File doesn't exist or not readable");
        }
        
        _deleteBuffer.Add(fileName, fileContent);
    }

    public async Task ExecuteAsync()
    {
        try
        {
            foreach (var fileName in _fileContentBuffer.Keys)
            {
                var fileContent = _fileContentBuffer[fileName];
                await _fileSystem.WriteAllTextAsync(fileName, fileContent);
            }

            foreach (var fileName in _deleteBuffer.Keys)
            {
                _fileSystem.Delete(fileName);
            }
        }
        catch
        {
            await _rollbackAsync();
            throw;
        }

        _fileContentBuffer = new Dictionary<string, string>();
        _deleteBuffer = new Dictionary<string, string>();
    }

    private async Task _rollbackAsync()
    {
        Console.WriteLine("Performing rollback");
        foreach (var fileName in _fileContentBuffer.Keys)
        {
            if (_fileSystem.Exists(fileName))
            {
                _fileSystem.Delete(fileName);
            }
        }

        foreach (var fileName in _deleteBuffer.Keys)
        {
            if (_fileSystem.Exists(fileName) == false)
            {
                await _fileSystem.WriteAllTextAsync(fileName, _deleteBuffer[fileName]);
            }
        }
    }
}
```
{% endtab %}

{% endtabs %}


#### Demonstration

To demonstrate this I'll create a simple CLI entry point that will do the set of actions:

    1. Create 3 files
    2. Create 2 new files + delete the previous 3 files

The first set is intended to be successful, the second set however we will simulate a failure to see the rollback happen. 

We will delete one of the files queued for deletion before we try and delete it in the unit of work, this will cause an exception in the execution.

{% tabs entrypoint %}

{% tab entrypoint php %}
```php
<?php
declare(strict_types = 1);

require_once('FileSystem.php');
require_once('FileSystemUnitOfWork.php');

$filesDirectory = dirname(__FILE__)  . "/Files";

$fileSystem = new FileSystem($filesDirectory);

// Successfully create 3 files
echo "- Creating the first three files\n";
$fileSystemUnitOfWork = new FileSystemUnitOfWork($fileSystem);
$fileSystemUnitOfWork->Add('file1', 'content');
$fileSystemUnitOfWork->Add('file2', 'content');
$fileSystemUnitOfWork->Add('file3', 'content');
$fileSystemUnitOfWork->Execute();

// Fail on deleting the last file then rollback
echo "- Creating two files and attempting to delete three files\n";
$fileSystemUnitOfWork->Add('file4', 'content');
$fileSystemUnitOfWork->Add('file5', 'content');
$fileSystemUnitOfWork->Delete('file1');
$fileSystemUnitOfWork->Delete('file2');
$fileSystemUnitOfWork->Delete('file3');

// Now lets simulate a failure that could happen (file doesnt exist for file3).
unlink($filesDirectory . "/file3");

try
{
    $fileSystemUnitOfWork->Execute();
}
catch (Exception $e)
{
    echo sprintf("Execution failed because %s\n", $e->getMessage());
}
```
{% endtab %}

{% tab entrypoint C# %}
```csharp
using UnitOfWorkPatternExample;

var filesDirectory = Directory.GetCurrentDirectory() + "/Files";
var fileSystem = new FileSystem(filesDirectory);

// Successfully create 3 files
Console.WriteLine("- Creating the first three files");
var fileSystemUnitOfWork = new FileSystemUnitOfWork(fileSystem);
fileSystemUnitOfWork.Add("file1", "content");
fileSystemUnitOfWork.Add("file2", "content");
fileSystemUnitOfWork.Add("file3", "content");
await fileSystemUnitOfWork.ExecuteAsync();

// Fail on deleting the last file then rollback
Console.WriteLine("- Creating two files and attempting to delete three files");
fileSystemUnitOfWork.Add("file4", "content");
fileSystemUnitOfWork.Add("file5", "content");
await fileSystemUnitOfWork.DeleteAsync("file1");
await fileSystemUnitOfWork.DeleteAsync("file2");
await fileSystemUnitOfWork.DeleteAsync("file3");
    
// Now lets simulate a failure that could happen (file doesnt exist for file3).
File.Delete(filesDirectory + "/file3");

try
{
    await fileSystemUnitOfWork.ExecuteAsync();
}
catch (Exception e)
{
    Console.WriteLine("Execution failed because {0}", e.Message);
}
```
{% endtab %}

{% endtabs %}

If we then run the application and take a look at the messages written to the console, we can see how the application is behaving and watch it rollback changes on the last failure.

```
- Creating the first three files
Creating file file1
Creating file file2
Creating file file3
- Creating two files and attempting to delete three files
Creating file file4
Creating file file5
Deleting file file1
Deleting file file2
Performing rollback
Deleting file file4
Deleting file file5
Creating file file1
Creating file file2
Creating file file3
Execution failed because File doesn't exist so can't delete
```

And that's it!

This pattern can be used with any infrastructure dependencies you might have including a database, cache & queues.

## Other uses

This pattern has many uses including:

- Transaction management (our example)
- Simplified data access management due to encapsulation
- Change tracking and caching
- Performance optimisations (think batching operations)


## Things to remember

If you are using dependency injection make sure the lifetime of the unit of work lasts for the whole request otherwise you may lose operations or even worse have operations shared across all requests.


## Example repositories

[PHP Example](https://github.com/caddoo/unit-of-work-pattern-example-php)

[C# Example](https://github.com/caddoo/unit-of-work-pattern-example-csharp)