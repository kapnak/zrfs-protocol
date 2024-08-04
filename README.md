# ZRFS - Protocol

*Version 1.0*

ZRFS is a remote file system base on POSIX with permission management. It is build on top of [ZProtocol](https://github.com/kapnak/z-protocol).

__Default port :__ 1501


## Features

- Individual file permission using the authentication of [ZProtocol](https://github.com/kapnak/z-protocol) and extend file attributes.
- Support most of the basics POSIX file systems methods and therefore is compatible with FUSE.

> [!IMPORTANT]
> The file system used under ZRFS (server-side) must support extended attributes.


## Implementation

| Language   | FUSE & tools | Library | Link                             |
| ---------- | ------------ | ------- | -------------------------------- |
| C          | ✅            | ✅       | https://github.com/kapnak/zrfs-c |
| JavaScript | ❌            | ✅       | https://github.com/kapnak/z-js   |


## Protocol

The communication starts without initialization.
The client can send a request anytime and the server must reply to it.
See the section [requests](#requests) for the details of every possible requests.


### File descriptor

Some request utilize a file descriptor.
A file descriptor can only be used by the client that opened it.


### Permissions

Each client is identify with his public key in base 32. The authentication is 
made by [ZProtocol](https://github.com/kapnak/z-protocol).

The permissions are stored in the extended attributes of the file and are used as an ACL :

**Key :** `user.z.acl.{public_key}`
- `public_key` - The client public key in base 32.

**Value :** `{level}`
- `level` - The [level](#levels) number in a single byte.

If no permission apply to the client, the server must check the parents directories
recursively and take the first permission found.
If no permission apply to the client in all parents directories the server must deny
the requests.


#### Default permission

To set a permission that will apply to all clients that don't have a permission defined
on the file, add an ACL with all the bytes of public key at 0. In base 32 its equivalent
to 52 'a' :
```
user.z.acl.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```


#### Levels

| Number | Meaning      | Description                              |
| ------ | ------------ | ---------------------------------------- |
| 0      | NOTHING      | The client can't see if the file exists  |
| 1      | REFERENCE    | The client can see the file with readdir |
| 2      | READ         | The client can read it                   |
| 3      | WRITE        | The client can edit the file             |
| 4      | ADMINISTRATE | The client can edit the permissions      |


## Requests

The following sections detail the different requests types their construction.

The first byte of a requests specifies the requests type.  
In the bytes column, `\0` indicates that the argument is a null-terminated string of variable length.

The first byte of a reply indicates whether the request was successful. If unsuccessful, it specifies the error code.


### GETATTR
*Minimum permission level :* 1

Get the attributes of a file or a directory.

#### Request
| Name | Bytes | Offset | Type   | Description                      |
| ---- | ----- | ------ | ------ | -------------------------------- |
| 10   | 1     | 0      | byte   | Request type                     |
| fd   | 8     | 1      | int    | File descriptor or 0 to use path |
| path | \0    | 9      | string | File path (unused if fd != 0)    |

#### Reply
| Name    | Bytes | Offset | Type         | Description                      |
| ------- | ----- | ------ | ------------ | -------------------------------- |
| error   | 8     | 0      | int          | 0 or errno                       |
| dev     | 8     | 8      | unsigned int | ID of device containing the file |
| ino     | 8     | 16     | unsigned int | Inode number                     |
| size    | 8     | 24     | int          | Total size in bytes              |
| blksize | 8     | 32     | unsigned int | Blocksize for file system I/O    |
| blocks  | 8     | 40     | int          | Number of 512b blocks allocated  |
| mode    | 8     | 48     | unsigned int | File type, permission removed    |


### ACCESS
*Minimum permission level :* 1

cf. [POSIX documentation for access](https://pubs.opengroup.org/onlinepubs/9799919799/functions/access.html)


#### Request
| Name | Bytes | Offset | Type   | Description  |
| ---- | ----- | ------ | ------ | ------------ |
| 12   | 1     | 0      | byte   | Request type |
| path | \0    | 1      | string | File path    |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


###  READDIR
*Minimum permission level :* 2

List files of a directory.

cf. [POSIX documentation for opendir](https://pubs.opengroup.org/onlinepubs/9799919799/functions/opendir.html), [POSIX documentation for readdir](https://pubs.opengroup.org/onlinepubs/9799919799/functions/readdir.html)

#### Request
| Name | Bytes | Offset | Type   | Description    |
| ---- | ----- | ------ | ------ | -------------- |
| 13   | 1     | 0      | byte   | Request type   |
| path | \0    | 1      | string | Directory path |

#### Reply
| Name    | Bytes | Offset | Type            | Description   |
| ------- | ----- | ------ | --------------- | ------------- |
| error   | 8     | 0      | int             | 0 or errno    |
| entries |       | 8      | [Entry](#entry) | List of files |

The `entries` argument is a list of [Entry](#entry).
To iterate over the entries : 
- Stop if the remaining message length is less than 51 (which is the minimum Entry size).
- Read the entry values (cf. [Entry](#entry)) but defer reading the name.
- Stop if the name length exceeds the remaining message length.
- Read the name.
- Continue to the next entry.

##### Entry
| Name    | Bytes | Offset | Type         | Description                                 |
| ------- | ----- | ------ | ------------ | ------------------------------------------- |
| fdp     | 1     | 0      | byte         | 1 if Dev, Size, Blksize and Blocks are set. |
| dev     | 8     | 1      | unsigned int | ID of device containing the file            |
| ino     | 8     | 9      | unsigned int | Inode number                                |
| size    | 8     | 17     | int          | Total size in bytes                         |
| blksize | 8     | 25     | unsigned int | Blocksize for file system I/O               |
| blocks  | 8     | 33     | int          | Number of 512b blocks allocated             |
| mode    | 8     | 41     | unsigned int | File type, permission removed               |
| name    | \0    | 49     | string       | File name                                   |


### MKDIR
*Minimum permission level :* 3

Create a directory.  
cf. [POSIX documentation for mkdir](https://pubs.opengroup.org/onlinepubs/9799919799/functions/mkdir.html)

#### Request
| Name | Bytes | Offset | Type   | Description        |
| ---- | ----- | ------ | ------ | ------------------ |
| 14   | 1     | 0      | byte   | Request type       |
| path | \0    | 1      | string | New directory path |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### UNLINK
*Minimum permission level :* 3

Delete a name and possibly the file it refers to.  
cf. [POSIX documentation for unlink](https://pubs.opengroup.org/onlinepubs/9799919799/functions/unlink.html)

#### Request
| Name | Bytes | Offset | Type   | Description  |
| ---- | ----- | ------ | ------ | ------------ |
| 15   | 1     | 0      | byte   | Request type |
| path | \0    | 1      | string | Name path    |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### RMDIR
*Minimum permission level :* 3

Delete an empty directory.  
cf. [POSIX documentation for rmdir](https://pubs.opengroup.org/onlinepubs/9799919799/functions/rmdir.html)

#### Request
| Name | Bytes | Offset | Type   | Description    |
| ---- | ----- | ------ | ------ | -------------- |
| 16   | 1     | 0      | byte   | Request type   |
| path | \0    | 1      | string | Directory path |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### SYMLINK
*Minimum permission level :* 3 on `from` parent directory

Create symlink.  
cf. [POSIX documentation for symlink](https://pubs.opengroup.org/onlinepubs/9799919799/functions/symlink.html)

#### Request
| Name | Bytes | Offset           | Type   | Description  |
| ---- | ----- | ---------------- | ------ | ------------ |
| 17   | 1     | 0                | byte   | Request type |
| from | \0    | 1                | string | Link path    |
| to   | \0    | 2 + sizeof(from) | string | Target       |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### RENAME
*Minimum permission level :*
- 3 on `from` file
- 3 on `to` parent directory

Rename a file and moving it between directories if required.  
cf. [POSIX documentation for rename](https://pubs.opengroup.org/onlinepubs/9799919799/functions/rename.html)

#### Request
| Name | Bytes | Offset           | Type   | Description          |
| ---- | ----- | ---------------- | ------ | -------------------- |
| 18   | 1     | 0                | byte   | Request type         |
| from | \0    | 1                | string | File path            |
| to   | \0    | 2 + sizeof(from) | string | New file name & path |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### LINK
*Minimum permission level :*
- 3 on `from` file
- 3 on `to` parent directory

Create a new link to an existing file.  
cf. [POSIX documentation for link](https://pubs.opengroup.org/onlinepubs/9799919799/functions/link.html)

#### Request
| Name | Bytes | Offset           | Type   | Description  |
| ---- | ----- | ---------------- | ------ | ------------ |
| 19   | 1     | 0                | byte   | Request type |
| from | \0    | 1                | string | Old path     |
| to   | \0    | 2 + sizeof(from) | string | New path     |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### OPEN
*Minimum permission level :*
- 3 on `path` if the flags `O_RDWR`, `O_WRONLY`, `O_APPEND` or `O_TRUNC` are set; otherwise 2
- 3 on `path` parent directory if flag `O_CREAT` is set

Create / Open a file.  
cf. [POSIX documentation for open](https://pubs.opengroup.org/onlinepubs/9799919799/functions/open.html)

#### Request
| Name | Bytes | Offset | Type   | Description                                                                            |
| ---- | ----- | ------ | ------ | -------------------------------------------------------------------------------------- |
| 21   | 1     | 0      | byte   | Request type                                                                           |
| flag | 8     | 1      | int    | cf. [POSIX open](https://pubs.opengroup.org/onlinepubs/9799919799/functions/open.html) |
| path | \0    | 9      | string | File path                                                                              |

#### Reply
| Name  | Bytes | Offset | Type | Description     |
| ----- | ----- | ------ | ---- | --------------- |
| error | 8     | 0      | int  | 0 or errno      |
| fd    | 8     | 8      | int  | File descriptor |


### READ
*Minimum permission level :* 2 on `fd`

Read a file.  
cf. [POSIX documentation for pread](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pread.html)

#### Request
| Name   | Bytes | Offset | Type         | Description               |
| ------ | ----- | ------ | ------------ | ------------------------- |
| 22     | 1     | 0      | byte         | Request type              |
| fd     | 8     | 1      | int          | File descriptor           |
| size   | 8     | 9      | unsigned int | Number of bytes to read   |
| offset | 8     | 17     | int          | Position to start reading |

#### Reply
| Name   | Bytes             | Offset | Type  | Description |
| ------ | ----------------- | ------ | ----- | ----------- |
| error  | 8                 | 0      | int   | 0 or errno  |
| buffer | sizeof(Reply) - 8 | 8      | bytes | Bytes read  |

> [!NOTE]
> The number of bytes read is equal to the buffer length and can be calculated as `sizeof(Reply) - 9`.


### WRITE
*Minimum permission level :* 3 on `fd`

Write bytes in a file.  
cf. [POSIX documentation for pwrite](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pwrite.html)

#### Request
| Name   | Bytes | Offset | Type         | Description               |
| ------ | ----- | ------ | ------------ | ------------------------- |
| 23     | 1     | 0      | byte         | Request Type              |
| fd     | 8     | 1      | int          | File descriptor           |
| size   | 8     | 9      | unsigned int | Number of bytes to write  |
| offset | 8     | 17     | int          | Position to start writing |
| buffer | size  | 25     | bytes        | Bytes to write            |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### STATVFS
*Minimum permission level :* 3 on `path`

Get file system information.  
cf. [POSIX documentation for statvfs](https://pubs.opengroup.org/onlinepubs/9799919799/functions/statvfs.html)

#### Request
| Name | Bytes | Offset | Type   | Description  |
| ---- | ----- | ------ | ------ | ------------ |
| 24   | 1     | 0      | byte   | Request type |
| path | \0    | 1      | string | File path    |

#### Reply
| Name    | Bytes | Offset | Type         | Description                                                       |
| ------- | ----- | ------ | ------------ | ----------------------------------------------------------------- |
| error   | 8     | 0      | int          | 0 or errno                                                        |
| bsize   | 8     | 8      | unsigned int | File system block size                                            |
| frsize  | 8     | 16     | unsigned int | Fundamental file system block size                                |
| blocks  | 8     | 24     | unsigned int | Total number of blocks on file system in units of f_frsize        |
| bfree   | 8     | 32     | unsigned int | Total number of free blocks                                       |
| bavail  | 8     | 40     | unsigned int | Number of free blocks available to non-privileged process         |
| files   | 8     | 48     | unsigned int | Total number of file serial numbers.                              |
| ffree   | 8     | 56     | unsigned int | Total number of free file serial numbers                          |
| favail  | 8     | 64     | unsigned int | Number of file serial numbers available to non-privileged process |
| fsid    | 8     | 72     | unsigned int | File system ID                                                    |
| flag    | 8     | 80     | unsigned int | Bit mask of f_flag values                                         |
| namemax | 8     | 88     | unsigned int | Maximum filename length                                           |


### CLOSE
*Minimum permission level :* 0

Close a file descriptor.  
cf. [POSIX documentation for close](https://pubs.opengroup.org/onlinepubs/9799919799/functions/close.html)

#### Request
| Name | Bytes | Offset | Type | Description     |
| ---- | ----- | ------ | ---- | --------------- |
| 25   | 1     | 0      | byte | Request type    |
| fd   | 8     | 1      | int  | File descriptor |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### TRUNCATE
*Minimum permission level :* 3

Truncate a file.  
cf. [POSIX documentation for truncate](https://pubs.opengroup.org/onlinepubs/9799919799/functions/truncate.html)

#### Request
| Name   | Bytes | Offset | Type   | Description                      |
| ------ | ----- | ------ | ------ | -------------------------------- |
| 28     | 1     | 0      | byte   | Request type                     |
| fd     | 8     | 1      | int    | File descriptor or 0 to use path |
| length | 8     | 9      | int    | New file length                  |
| path   | \0    | 17     | string | File path (unused if fd != 0)    |

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |


### GETPERM
*Minimum permission level :* 1

Get file permissions ACL.

#### Request
| Name | Bytes | Offset | Type   | Description  |
| ---- | ----- | ------ | ------ | ------------ |
| 26   | 1     | 0      | byte   | Request type |
| path | \0    | 1      | string | File path    |

#### Reply
| Name       | Bytes             | Offset | Type                        | Description          |
| ---------- | ----------------- | ------ | --------------------------- | -------------------- |
| error      | 8                 | 0      | int                         | 0 or errno           |
| permission | sizeof(Reply) - 8 | 8      | [[Permission](#permission)] | A list of Permission |

#### Permission
| Name  | Bytes | Offset | Type   | Description                 |
| ----- | ----- | ------ | ------ | --------------------------- |
| pk    | 52    | 0      | string | User public key in base 32  |
| level | 1     | 52     | byte   | Permission [level](#levels) |

> [!CAUTION]
> The public key is a fixed length of 52 bytes and **is not null-terminated**.


### SETPERM
*Minimum permission level :* 4

Set file permission. If the specified user already have a permission defined, it will be overwritten.

#### Request
| Name  | Bytes | Offset | Type   | Description                 |
| ----- | ----- | ------ | ------ | --------------------------- |
| 27    | 1     | 0      | byte   | Request type                |
| level | 1     | 1      | byte   | Permission [level](#levels) |
| pk    | 52    | 2      | string | User public key in base 32  |
| path  | \0    | 54     | string | File path                   |

> [!CAUTION]
> The public key is a fixed length of 52 bytes and **is not null-terminated**.

#### Reply
| Name  | Bytes | Offset | Type | Description |
| ----- | ----- | ------ | ---- | ----------- |
| error | 8     | 0      | int  | 0 or errno  |
