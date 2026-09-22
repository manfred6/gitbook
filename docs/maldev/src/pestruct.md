# PE Structure

The PE file structure encopasses at least the basic information required to load and execute a process. This includes its own executable code, dlls required, and other data as well as where to find it.

![](https://github.com/corkami/pics/blob/master/binary/pe101/pe101.png?raw=true)

## DOS Header

DOS header starts with the well-known two-byte sequence:

```hex
4D 5A
```
> `MZ`

followed by the DOS string:

```text
This program cannot be run in DOS mode
```
> optional

Overall, the MS-DOS stub structure looks as follows:

```c
//0x40 bytes (sizeof)
struct _IMAGE_DOS_HEADER
{
    USHORT e_magic;                                                         //0x0
    USHORT e_cblp;                                                          //0x2
    USHORT e_cp;                                                            //0x4
    USHORT e_crlc;                                                          //0x6
    USHORT e_cparhdr;                                                       //0x8
    USHORT e_minalloc;                                                      //0xa
    USHORT e_maxalloc;                                                      //0xc
    USHORT e_ss;                                                            //0xe
    USHORT e_sp;                                                            //0x10
    USHORT e_csum;                                                          //0x12
    USHORT e_ip;                                                            //0x14
    USHORT e_cs;                                                            //0x16
    USHORT e_lfarlc;                                                        //0x18
    USHORT e_ovno;                                                          //0x1a
    USHORT e_res[4];                                                        //0x1c
    USHORT e_oemid;                                                         //0x24
    USHORT e_oeminfo;                                                       //0x26
    USHORT e_res2[10];                                                      //0x28
    LONG e_lfanew;                                                          //0x3c
};
```

Required to actually execute are only the `e_magic` and `e_lfanew` members. 
The latter is a `4-byte little endian` value which points to the offset of the next header (`PE header`).


## PE Header

The windows sdk defines two structs for the PE header, `IMAGE_NT_HEADERS` for 32-bit systems and `IMAGE_NT_HEADERS64` for 64-bit systems.
The PE header starts with the `4-byte` sequence:

```hex
50 45 00 00
```
> `PE\0\0`


The `IMAGE_NT_HEADERS*` structure contains the following members:


| Structure | Member | Content |
|:----------|:------:|:-------:|
| PE Header | PE Signature | 4-byte `PE\0\0` |
| PE Header | File Header | `IMAGE_FILE_HEADER` struct |
| PE Header | Optional Header | `IMAGE_OPTIONAL_HEADER` struct |
| File Header | Machine Number | Arch PE is compiled for |
| File Header | Number of sections | Number of PE secsions |
| File Header | SizeofOptionalHeader | byte size of opt header | 
| File Header | Characteristics | other attributes of pe file | 
| Optional Header | Magic | 32 or 64-bit flag | 
| Optional Header | AddressOfEntrypoint | addr of entry relative to pe base | 
| Optional Header | ImageBase | preferred image base | 
| Optional Header | NumberOfRvaAndSizes | size of datadirectory array | 
| Optional Header | DataDirectory | Array of `IMAGE_DATA_DIRECTORY` structs | 



These can be accessed/extracted as follows:


```c
#include <windows.h>
#include <stdio.h>

int main() {
    
    // getmodhandle(null) returns base addr of currently running PE
    // casting to pointer is a trick for effectively obtaining the base addr of the current PE file
    DWORD_PTR base = (DWORD_PTR)GetModuleHandleA(NULL);
    // base is literally the start of the _IMAGE_DOS_HEADER struct, so we can just cast it to its struct 
    PIMAGE_DOS_HEADER *dos = (PIMAGE_DOS_HEADER)base
    
    printf("(i) -> Found %s at %p\n", (CHAR*)&dos->e_magic, base);
    
    CHAR *pe = (CHAR*)(base + dos->e_lfanew);
    printf("(i) -> Found %s at %p\n", pe, (void*)dos->e_lfanew);

    return(0);

}

```

## Optional Header - Data Directories

The `IMAGE_DATA_DIRECTORY` structs contain a `VirtualAddress`, which points to the start of a particular data directory structure, and a `Size`, which is the size of that data directory.
These irectories contain information utilized by the Windows loader, for example a list of dlls the executable requires to function.

## Sections

The sections of the PE contain the data and code of the program. They typically comprise the following:

| Section | Usage | 
|:-----:|:-------:|
| `.text`  | the executable code of the program |
| `.data`  | initialised data  |
| `.bss`   | uninitialised data |
| `.rdata` | read-only data |
| `.rsrc`  | resources used by the program |

Each section has the following header:

- 8-byte `Name`
- `VirtualSize` describing its size when loaded into memory
- `VirtualAddress` of the section (offset to base addr)
- `SizeOfRawData` size of section data on disk
- `Characteristics` for example memory permissions of the section


---

