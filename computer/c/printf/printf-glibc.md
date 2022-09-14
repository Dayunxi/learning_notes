# glibc-2.31

printf的实现位于`stdio-common/printf.c`下的`__printf`(编译器别名，实际名为`_IO_printf`)。
最终会调用`vfprintf()`，并通过宏`outstring`输出至目标文件

```c
// libio/libioP.h
/* We always allocate an extra word following an _IO_FILE.
   This contains a pointer to the function jump table used.
   This is for compatibility with C++ streambuf; the word can
   be used to smash to a pointer to a virtual function table. */
struct _IO_FILE_plus
{
  FILE file;
  const struct _IO_jump_t *vtable;
};
```

`FILE *fopen(const char *path, const char *mode);`本身返回的FILE指针就是被截断过的(见libio/iofopen.c)

```c
// ./libio/stdio.c
#undef stdin
#undef stdout
#undef stderr
FILE *stdin = (FILE *) &_IO_2_1_stdin_;
FILE *stdout = (FILE *) &_IO_2_1_stdout_;
FILE *stderr = (FILE *) &_IO_2_1_stderr_;
```

```c
// ./libio/stdfiles.c
#ifdef _IO_MTSAFE_IO
# define DEF_STDFILE(NAME, FD, CHAIN, FLAGS) \
  static _IO_lock_t _IO_stdfile_##FD##_lock = _IO_lock_initializer; \
  static struct _IO_wide_data _IO_wide_data_##FD \
    = { ._wide_vtable = &_IO_wfile_jumps }; \
  struct _IO_FILE_plus NAME \
    = {FILEBUF_LITERAL(CHAIN, FLAGS, FD, &_IO_wide_data_##FD), \
       &_IO_file_jumps};
#else
# define DEF_STDFILE(NAME, FD, CHAIN, FLAGS) \
  static struct _IO_wide_data _IO_wide_data_##FD \
    = { ._wide_vtable = &_IO_wfile_jumps }; \
  struct _IO_FILE_plus NAME \
    = {FILEBUF_LITERAL(CHAIN, FLAGS, FD, &_IO_wide_data_##FD), \
       &_IO_file_jumps};
#endif

DEF_STDFILE(_IO_2_1_stdin_, 0, 0, _IO_NO_WRITES);
DEF_STDFILE(_IO_2_1_stdout_, 1, &_IO_2_1_stdin_, _IO_NO_READS);
DEF_STDFILE(_IO_2_1_stderr_, 2, &_IO_2_1_stdout_, _IO_NO_READS+_IO_UNBUFFERED);
```


# 具体的调用栈

```c
printf()
	// ldbl_strong_alias (__printf, printf); ldbl_strong_alias (__printf, _IO_printf);
	__printf() // stdio-common/printf.c
		// # define vfprintf __vfprintf_internal
		__vfprintf_internal() 
			vfprintf()  // stdio-common/vfprintf-internal.c
				outstring // MACRO
					// #define PUT(F, S, N) _IO_sputn ((F), (S), (N))
					PUT(file_ptr, string, len)
						// #define _IO_sputn(__fp, __s, __n) _IO_XSPUTN (__fp, __s, __n)
						_IO_sputn 
							// #define _IO_XSPUTN(FP, DATA, N) JUMP2 (__xsputn, FP, DATA, N)
							_IO_XSPUTN 
								//#define JUMP2(FUNC, THIS, X1, X2) (_IO_JUMPS_FUNC(THIS)->FUNC) (THIS, X1, X2)
								JUMP2
									_IO_JUMPS_FUNC // 将指针偏移至FILE结构体后面的跳转表地址
									// 最终指向_IO_new_file_xsputn函数
```

```c
_IO_new_file_xsputn  // libio/fileops.c
	new_do_write
		#define _IO_SYSWRITE(FP, DATA, LEN) JUMP2 (__write, FP, DATA, LEN)
		_IO_SYSWRITE  // call __write in unistd.h
```

```c
/* libio/fileops.c */
versioned_symbol (libc, _IO_new_file_xsputn, _IO_file_xsputn, GLIBC_2_1);

size_t _IO_new_file_xsputn (FILE *f, const void *data, size_t n)
{
	...
	_IO_OVERFLOW()	// flush buffer // call _IO_new_file_overflow // 同new_file_write
	...
	new_do_write()	// call _IO_new_file_write 
	...
	// 不以换行符结尾的剩余buffer
	_IO_default_xsputn()  // libio/genops.c
}

// call syscall
size_t _IO_new_file_write (FILE *f, const void *data, ssize_t n)
{
	...
	__write()  // syscall write()
	...
}
```

## _IO_file_jumps 表初始化
```c
/* libio/fileops.c */
versioned_symbol (libc, _IO_new_do_write, _IO_do_write, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_attach, _IO_file_attach, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_close_it, _IO_file_close_it, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_finish, _IO_file_finish, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_fopen, _IO_file_fopen, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_init, _IO_file_init, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_setbuf, _IO_file_setbuf, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_sync, _IO_file_sync, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_overflow, _IO_file_overflow, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_seekoff, _IO_file_seekoff, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_underflow, _IO_file_underflow, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_write, _IO_file_write, GLIBC_2_1);
versioned_symbol (libc, _IO_new_file_xsputn, _IO_file_xsputn, GLIBC_2_1);

const struct _IO_jump_t _IO_file_jumps libio_vtable =
{
  JUMP_INIT_DUMMY,
  JUMP_INIT(finish, _IO_file_finish),
  JUMP_INIT(overflow, _IO_file_overflow),
  JUMP_INIT(underflow, _IO_file_underflow),
  JUMP_INIT(uflow, _IO_default_uflow),
  JUMP_INIT(pbackfail, _IO_default_pbackfail),
  JUMP_INIT(xsputn, _IO_file_xsputn),
  JUMP_INIT(xsgetn, _IO_file_xsgetn),
  JUMP_INIT(seekoff, _IO_new_file_seekoff),
  JUMP_INIT(seekpos, _IO_default_seekpos),
  JUMP_INIT(setbuf, _IO_new_file_setbuf),
  JUMP_INIT(sync, _IO_new_file_sync),
  JUMP_INIT(doallocate, _IO_file_doallocate),
  JUMP_INIT(read, _IO_file_read),
  JUMP_INIT(write, _IO_new_file_write),
  JUMP_INIT(seek, _IO_file_seek),
  JUMP_INIT(close, _IO_file_close),
  JUMP_INIT(stat, _IO_file_stat),
  JUMP_INIT(showmanyc, _IO_default_showmanyc),
  JUMP_INIT(imbue, _IO_default_imbue)
};
```