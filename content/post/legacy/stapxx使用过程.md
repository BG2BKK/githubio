+++
date = '2016-04-01T18:38:31+08:00'
draft = true
title = 'stapxx使用过程'

+++



lj-gc
---------------------

- usage:

```bash
[zhendong@D13124441 stapxx]$ sudo ./stap++  ./samples/lj-gc.sxx -x 43376
```

- error:

```bash
Found exact match for libluajit: /usr/local/lib/libluajit-5.1.so.2.1.0
WARNING: cannot find module /usr/local/lib/libluajit-5.1.so.2.1.0 debuginfo: No DWARF information found [man warning::debuginfo]
semantic error: type definition 'lua_State' not found in '/usr/local/lib/libluajit-5.1.so.2.1.0': operator '@cast' at stapxx-QIH_WYIt/luajit.stp:162:12
        source:     return @cast(L, "lua_State", "/usr/local/lib/libluajit-5.1.so.2.1.0")->glref->ptr32
                           ^

Pass 2: analysis failed.  [man error::pass2]
```

- reason: 
    
    libluajit-5.1没有dwarf信息，需要重新编译

- resolve:

```bash
make CCDEBUG="-g -O0 -gdwarf-2" -j
# -gdwarf-2将会保持DWARF参数
```

elf一般由多个节(section)组成，不熟悉的可以看关于[elf文件格式的文章](http://blog.csdn.net/coutcin/article/details/1065470)。调试信息被包含在某几个节中，如果是用dwarf2格式编译的，这些节的名字一般是以.debug开头，如.debug_info，.debug_line，.debug_frame等，如果是用dwarf1格式编译的，这些节的名字一般是.debug，.line等。现在的编译器默认大多数是dwarf2格式编译，当然可以通过gcc的编译选项改变。

```bash
# 查看elf文件的各段头部
readelf -S elffile

# 查看elf文件的.debug_info段，关键在于info的i，所以采用 -wi参数
readelf -wi src/libluajit.so

# 查看elf文件的.debug_line段，关键在于info的l，所以采用 -wl参数
readelf -wl src/libluajit.so

```

对于LuaJIT来说，-O0和-O2(默认)对于DWARF信息来说没有区别，因为dwarf信息是调试信息。而-gdwarf-2和-g的差别在于，前者能够保留更详细一步的信息(对于.dubeg_info段而言)

```bash
#output by -gdwarf-2
 <2><134>: Abbrev Number: 8 (DW_TAG_member)
    <135>   DW_AT_name        : env	
    <139>   DW_AT_decl_file   : 2	
    <13a>   DW_AT_decl_line   : 661	
    <13c>   DW_AT_type        : <0x3b9>	
    <140>   DW_AT_data_member_location: 2 byte block: 23 2c 	(DW_OP_plus_uconst: 44)

#output by -g
 <2><11c>: Abbrev Number: 8 (DW_TAG_member)
    <11d>   DW_AT_name        : env	
    <121>   DW_AT_decl_file   : 2	
    <122>   DW_AT_decl_line   : 661	
    <124>   DW_AT_type        : <0x37e>	
    <128>   DW_AT_data_member_location: 44	
```

- correct result

```bash
[zhendong@D13124441 stapxx]$ sudo ./stap++  ./samples/lj-gc.sxx -x 13337
Found exact match for libluajit: /usr/local/lib/libluajit-5.1.so.2.1.0
Start tracing 13337 (/usr/local/nginx/sbin/nginx)
Total GC count: 128828 bytes

```

- extrainfo

在压测时，采用-c10000的压力，发现压力较大时报错

```bash
[zhendong@D13124441 stapxx]$ sudo ./stap++  ./samples/lj-gc.sxx -x 13337
Found exact match for libluajit: /usr/local/lib/libluajit-5.1.so.2.1.0
ERROR: read fault [man error::fault] at 0x000000000077d8e8 (addr) near operator '@var' at stapxx-j8srK_I0/nginx.config.stp:7:80
Start tracing 13337 (/usr/local/nginx/sbin/nginx)
WARNING: Number of errors: 1, skipped probes: 0
WARNING: /usr/bin/staprun exited with status: 1
Pass 5: run failed.  [man error::pass5]

```

重启解决，很无奈


ngx-lua-shdict-info
---------------------------
- 采用openresty才能正常使用也是够了

[zhendong@D13124441 stapxx]$ sudo ./stap++ ./samples/ngx-lua-shdict-info.sxx -x 5545 --arg dict=kv_api_root_upstream
Start tracing 5545 (/data1/zhendong/openresty-1.9.7.4/build/nginx-1.9.7/objs/nginx)
shm zone "api_root_sysConfig"
shm zone "kv_api_root_upstream"
shm zone "kv_api_root_upstream"
    owner: ngx_http_lua_shdict
    total size: 102400 KB
    free pages: 96392 KB (24098 pages, 29 blocks)
    rbtree black height: 10


