# BOF Dev

## Compile

### Windows

```
# native windows
cl.exe /c /GS- hello-inject.c /Fohello-inject.o

# mingw windows
x86_64-w64-mingw32-gcc.exe -c hello-inject.c -o hello-inject.o
```

### Linux

```
x86_64-w64-mingw32-gcc -c hello-inject.c -o hello-inject.o
```

## Hello Inject
Packing args is wyird...

- BOF uses `go(char * args, int len)`
- Args are packed in binary format using `bof_pack` function
- Args are unpacked via functions (eg `BeaconDataParse` and `BeaconDataExtract`) exported by `beacon.h`
- Args are unpacked in the same order they were packed

### Unpacking Arguments in BOF

```
#include <windows.h>
#include "beacon.h"

/*
 * hello-inject c:\\windows\\system32\\svchost.exe /path/to/local.bin 1234
 */

void go(char * args, int len)
{
    datap parser;
    BeaconDataParse(&parser, args, len);

    char * pe_name;
    unsigned char * sc_bin;
    SIZE_T sc_bin_len; // calculated, not provided as arg
    int ppid;

    // arg1 - the program name
    pe_name = BeaconDataExtract(&parser, NULL);
    
    // arg2 - the local bin file
    sc_bin_len = BeaconDataLength(&parser);  // doesn't increment the unpack (double check)
    sc_bin = BeaconDataExtract(&parser, NULL);

    // arg3 - the parent process pid
    ppid = BeaconDataInt(&parser);

    BeaconPrintf(CALLBACK_OUTPUT, "pe: %s | bin size: %d | ppid: %d", pe_name,sc_bin_len,ppid);
}
```

### Packing Arguments in CNA

Packing in CNA only losely applies to Sliver extension manifest. For porting BOF originally written for CS, the following info is relevent. 

- Separated by whitespace
- `$1` is current beacon
- `$2`, `$3`, `$4`, ... is input
- `z` is data format (see Data Formats Table below)

Packing a single arg in CNA would look like:

```
$args = bof_pack($1, "z", $2);
```

Packing multiple args of multiple types in CNA:

```
# args = string, string, int
# type = z     , z     , i
$args = bof_pack($1, "zzi", "the", "likes", 1337);
```

#### Beacon Data Formats Table
Provided by CobaltStrike documentation

|Format| Description | Unpack Function |
|---|----------|----------|
| b | Binary data | BeaconDataExtract |
| i | 4-byte integer (int) | BeaconDataInt|
| s | 2-byte integer (short) | BeaconDataShort |
| z | zero-terminated+encoded string | BeaconDataExtract |
| Z | zero-terminated wide string | (wchar_t *)BeaconDataExtract |

## Sliver
Most existing BOFs designed for CS can be ported. Review the Sliver wiki for details.

### Extensions

Sliver extensions are JSON manifest files that detail the BOF's entry point, the BOF filenames, and the expected arguments:

```
{
    "name": "hello-inject",
    "version": "0.0.0",
    "command_name": "hello-inject",
    "extension_author": "thelikes",
    "original_author": "thelikes",
    "repo_url": "https://github.com/sliverarmory/hello-inject",
    "help": "Sliver extension hello world",
    "long_help": "",
    "depends_on": "coff-loader",
    "entrypoint": "go",
    "files": [
        {
            "os": "windows",
            "arch": "amd64",
            "path": "hello-inject.x64.o"
        },
        {
            "os": "windows",
            "arch": "386",
            "path": "hello-inject.x86.o"
        }
    ],
    "arguments": [
        {
            "name": "pe",
            "desc": "Target program",
            "type": "string",
            "optional": false
        },
        {
            "name": "bin",
            "desc": "local bin",
            "type": "file",
            "optional": false
        },
        {
            "name": "ppid",
            "desc": "PID of desired parent process",
            "type": "int",
            "optional": false
        }
    ]
}
```

#### Sliver Data Formats Table
Provided by Sliver [wiki](https://github.com/BishopFox/sliver/wiki/BOF-&-COFF-Support)

| Type | Description | Sliver Type |
|---|----------|----------|
|b | binary data | 	file (path to binary data) |
|i | 4-byte integer | 	int or integer |
|s | 2-byte short integer | 	short |
|z | zero-terminated+encoded string | 	string |
|Z | zero-terminated wide-char string | 	wstring

### Sliver Extension Packaging

- Sliver expects a `tar.gz` file as input
- At minimum, the archive needs to include the compiled BOF and the JSON manifest
- **The archive name needs to match the** `command_name`

Drop the compiled BOF and the manifest into a directory and archive it:

```
$ tree 
.
├── extension.json
└── hello-inject.o

0 directories, 2 files

$ tar czvf ../hello-inject.tar.gz .  
./
./hello-inject.o
./extension.json
```

### Execution
Steps include installing and loading the extension (who knows):

```
# install
[server] sliver > extensions install /tmp/hello-inject.tar.gz

[*] Installing extension 'hello-inject' (0.0.0) ... done!

# load
[server] sliver > extensions load /root/.sliver-client/extensions/hello-inject

[*] Added hello-inject command: Sliver extension hello world

[server] sliver > hello-inject -h

Sliver extension hello world

Usage:
======
  hello-inject [flags] pe bin ppid

Args:
=====
  pe    string    Target program
  bin   string    local bin
  ppid  int       PID of desired parent process

Flags:
======
  -h, --help           display help
  -t, --timeout int    command timeout in seconds (default: 60)

# run
[server] sliver (CRUCIAL_BRICK) > hello-inject c:\\windows\\systmem32\\svchost.exe /opt/sliver/SPLENDID_EMERGENCE.bin 4343

[*] Successfully executed hello-inject (coff-loader)
[*] Got output:
pe: c:\windows\systmem32\svchost.exe | bin size: 12681444 | ppid: 4343
```

## Win32 API
The Dynamic Function Resolution (DFR) can be used to to dynamically resolve the API

- declare the API with `DECLSPEC_IMPORT INT WINAPI USER32$MessageBoxW(HWND, LPCWSTR, LPCWSTR, UINT);`
- execute it with `USER32$MessageBoxW(NULL, L"gr33tz", L"Message Box", 0);`

## COFF Loader
[trustedsec/COFFLoader](https://github.com/trustedsec/COFFLoader) can be used to execute BOF without c2 framework. This is useful for debugging purposes.

## Resources
- [Writing Beacon Object Files: Flexible, Stealthy, and Compatible](https://www.coresecurity.com/core-labs/articles/writing-beacon-object-files-flexibie-stealthy-and-compatible)
- [beachon.h](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/beacon.h)
- [Beacon Object Files](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/beacon-object-files_main.htm)
- [Direct Syscalls in Beacon Object Files](https://outflank.nl/blog/2020/12/26/direct-syscalls-in-beacon-object-files/)
- [Process Creation is Dead, Long Live Process Creation — Adding BOFs Support to PEzor](https://iwantmore.pizza/posts/PEzor4.html)
- [Malware Development: Leveraging Beacon Object Files for Remote Process Injection via Thread Hijacking ](https://connormcgarr.github.io/thread-hijacking/)
- [outflanknl/InlineWhispers](https://github.com/outflanknl/InlineWhispers)