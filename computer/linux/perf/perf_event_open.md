
# struct perf_event_attr

```c
struct perf_event_attr {
    __u32 type;                      /* Type of event */
    __u32 size;                      /* Size of attribute structure */
    __u64 config;                    /* Type-specific configuration */

    union {
         __u64 sample_period;    /* Period of sampling */
         __u64 sample_freq;      /* Frequency of sampling */
    };

    __u64 sample_type;  /* Specifies values included in sample */
    __u64 read_format;  /* Specifies values returned in read */

    __u64 disabled          : 1,     /* off by default */
            inherit         : 1,     /* children inherit it */
            pinned          : 1,     /* must always be on PMU */
            exclusive       : 1,     /* only group on PMU */
            exclude_user    : 1,     /* don't count user */
            exclude_kernel  : 1,     /* don't count kernel */
            exclude_hv      : 1,     /* don't count hypervisor */
            exclude_idle    : 1,     /* don't count when idle */
            mmap            : 1,     /* include mmap data */
            comm            : 1,     /* include comm data */
            freq            : 1,     /* use freq, not period */
            inherit_stat    : 1,     /* per task counts */
            enable_on_exec  : 1,     /* next exec enables */
            task            : 1,     /* trace fork/exit */
            watermark       : 1,     /* wakeup_watermark */
            precise_ip      : 2,     /* skid constraint */
            mmap_data       : 1,     /* non-exec mmap data */
            sample_id_all   : 1,     /* sample_type all events */
            exclude_host    : 1,     /* don't count in host */
            exclude_guest   : 1,     /* don't count in guest */
            exclude_callchain_kernel : 1,
                                         /* exclude kernel callchains */
            exclude_callchain_user   : 1,
                                         /* exclude user callchains */
            mmap2           : 1,     /* include mmap with inode data */
            comm_exec       : 1,     /* flag comm events that are
                                              due to exec */
            use_clockid     : 1,     /* use clockid for time fields */
            context_switch  : 1,     /* context switch data */
            write_backward  : 1,     /* Write ring buffer from end
                                              to beginning */
            namespaces      : 1,     /* include namespaces data */
            ksymbol         : 1,     /* include ksymbol events */
            bpf_event       : 1,     /* include bpf events */
            aux_output      : 1,     /* generate AUX records
                                              instead of events */
            cgroup          : 1,     /* include cgroup events */
            text_poke       : 1,     /* include text poke events */

            __reserved_1    : 30;

    union {
         __u32 wakeup_events;    /* wakeup every n events */
         __u32 wakeup_watermark; /* bytes before wakeup */
    };

    __u32        bp_type;            /* breakpoint type */

    union {
         __u64 bp_addr;          /* breakpoint address */
         __u64 kprobe_func;      /* for perf_kprobe */
         __u64 uprobe_path;      /* for perf_uprobe */
         __u64 config1;          /* extension of config */
    };

    union {
         __u64 bp_len;               /* breakpoint length */
         __u64 kprobe_addr;      /* with kprobe_func == NULL */
         __u64 probe_offset;         /* for perf_[k,u]probe */
         __u64 config2;          /* extension of config1 */
    };
    __u64 branch_sample_type;    /* enum perf_branch_sample_type */
    __u64 sample_regs_user;      /* user regs to dump on samples */
    __u32 sample_stack_user;     /* size of stack to dump on
                                             samples */
    __s32 clockid;                   /* clock to use for time fields */
    __u64 sample_regs_intr;      /* regs to dump on samples */
    __u32 aux_watermark;             /* aux bytes before wakeup */
    __u16 sample_max_stack;      /* max frames in callchain */
    __u16 __reserved_2;          /* align to u64 */

};
```

## type 

- PERF_TYPE_HARDWARE
- PERF_TYPE_SOFTWARE
- PERF_TYPE_HW_CACHE
- PERF_TYPE_RAW
- ...
- dynamic PMU： `/sys/bus/event_source/devices/*/type`

`/sys/bus/event_source/devices/` 下每个子目录代表一个 `PMU`

## size

size = sizeof(struct perf_event_attr)

## config

具体监控事件，不同的type会有不同的事件。
64-bits，不够的话还有 `config1` 和 `config2`。

## sample_period, sample_freq

- 每 `sample_period` 次事件触发计数器溢出，并将 `sample_type` 类型的结果写入mmap的缓冲区
- 如果设置了 `sample_freq`，内核将会自动调整 `sample_period` 以尽量达到目标采样频率

*注意*： 这两个字段在一个union中，共用同一块内存，通过 `freq` 字段决定用哪个

## freq

决定是按周期采样 `sample_period` 还是按频率采样 `sample_freq` 

## sample_type

通过不同的bits字段决定哪些值会记录在采样数据中。决定在 `struct perf_event_header` 的记录类型为 `PERF_RECORD_SAMPLE` 时，记录里面包含哪些值

## read_format

该字段决定 `read` 时返回的格式

## disabled 

填1时代表暂不开启计数，后面可以通过 `ioctl` 或者 `prctl` 命令开启计数。

如果要创建一个 event group，一般要将 group leader 的 `disabled` 置为1，然后将其他所有子事件的 `disabled` 置为0.（子事件在 group leader 被 enable 后才会开始计数）

## inherit

该bit决定是否也统计子进程的事件。（仅对计数器创建后的新的子进程有效）

*注意*：与某些 `read_format` 的值不兼容，如 `PERF_FORMAT_GROUP` 

## inherit_stat

...

## pinned

要求计数器必须在CPU上。（可能出现硬件计数器数量不够的情况）
仅对硬件计数器以及 group leader 可用

## exclusive

设置后，CPU仅允许该计数器组工作。

*注意*：有很多种情况会导致该设置失效

## exclude_user, exclude_kernel, exclude_hv, exclude_idle

不统计 用户态/内核态/虚拟化/idle进程 出现的事件

## mmap, mmap_data

该bit设置后会对所有设置了 `PROT_EXEC` 的 `mmap` 调用产生一个 `PERF_RECORD_MMAP` 采样记录

如果设置了 mmap_data，则没有设置 `PERF_RECORD_MMAP` 的 `mmap` 也会被记录

## comm

...

## enable_on_exec, task. watermark

## precise_ip

允许记录的指令与实际触发事件的指令有多大偏差

参考 `PEBS`

## sample_id_all 

...

## exclude_host, exclude_guest

虚拟机相关...

## exclude_callchain_kernel， exclude_callchain_user 

是否排除函数调用链

## mmap2, comm_exec 

...

## use_clockid (since Linux 4.1)

决定使用哪个时钟源(`clockid`)来产生时间戳。

## context_switch (since Linux 4.3)

上下文切换时产生 `PERF_RECORD_SWITCH` 记录。（也会产生 `PERF_RECORD_SWITCH_CPU_WIDE` 记录？）

## write_backward

为了支持读取可被覆写的 ring buffer，于是从末尾向开头写 ring buffer

## namespaces, ksymbol, bpf_event, auxevent, cgroup, text_poke

发生某个对应事件时产生对应格式的记录

## wakeup_events, wakeup_watermark

...

## bp_type, bp_addr, bp_len 

断点类型，断点地址


## config1, config2

`config` 的 64-bits 可能还不够，新加两个字段。

## branch_sample_type (since Linux 3.4)

`PERF_SAMPLE_BRANCH_STACK` 开启后，记录type所指定的分支至分支记录。

## sample_regs_user (since Linux 3.7)

此 mask 决定哪些寄存器的值会被记录至采样数据中。mask 的位置参考 `arch/x86/include/uapi/asm/perf_regs.h`。

## sample_stack_user (since Linux 3.7)

当 `PERF_SAMPLE_STACK_USER` 启用时，决定保存多大的栈。

## clockid (since Linux 4.1)

`use_clockid` 被设置后，通过该字段决定使用哪种计时器。如 `CLOCK_MONOTONIC`，`CLOCK_REALTIME` 等。

## aux_watermark 

## sample_max_stack (since Linux 4.1)

如果 `sample_type` 包含了 `PERF_SAMPLE_CALLCHAIN`，该字段决定最多包含多少个栈帧。


----

# Reading results

如果使用了 `PERF_FORMAT_GROUP`，则读取格式如下:
```c
struct read_format {
    u64 nr;            /* The number of events */
    u64 time_enabled;  /* if PERF_FORMAT_TOTAL_TIME_ENABLED */
    u64 time_running;  /* if PERF_FORMAT_TOTAL_TIME_RUNNING */
    struct {
        u64 value;     /* The value of the event */
        u64 id;        /* if PERF_FORMAT_ID */
    } values[nr];
};
```

否则格式如下：
```c
struct read_format {
    u64 value;         /* The value of the event */
    u64 time_enabled;  /* if PERF_FORMAT_TOTAL_TIME_ENABLED */
    u64 time_running;  /* if PERF_FORMAT_TOTAL_TIME_RUNNING */
    u64 id;            /* if PERF_FORMAT_ID */
};
```

`time_enabled` 和 `time_running` 分别代表事件被启用，和在CPU运行的总时长(ns)，一般情况下两个值一样，除非出现计数器复用。


# MMAP layout

以采样模式开启 `perf_event_open` 时，当发生计数器溢出时(或者如 `PROT_EXEC` 的 mmap 等事件被触发)，会将结果写入环形缓冲区。

mmap的size必须为 `2^n+1` 个pages，第一页用来放置一个 `struct perf_event_mmap_page`，后面的页用来放置 ring-buffer。

`perf_event_open` 的 `fd` 都是对应独立的 mmap 区域

## struct perf_event_mmap_page (第1页)

```c
struct perf_event_mmap_page {
    __u32 version;        /* version number of this structure */
    __u32 compat_version; /* lowest version this is compat with */
    __u32 lock;           /* seqlock for synchronization */
    __u32 index;          /* hardware counter identifier */
    __s64 offset;         /* add to hardware counter value */
    __u64 time_enabled;   /* time event active */
    __u64 time_running;   /* time event on CPU */
    union {
        __u64   capabilities;
        struct {
            __u64 cap_usr_time / cap_usr_rdpmc / cap_bit0 : 1,
                    cap_bit0_is_deprecated : 1,
                    cap_user_rdpmc         : 1,
                    cap_user_time          : 1,
                    cap_user_time_zero     : 1,
        };
    };
    __u16 pmc_width;
    __u16 time_shift;
    __u32 time_mult;
    __u64 time_offset;
    __u64 __reserved[120];   /* Pad to 1 k */
    __u64 data_head;         /* head in the data section */
    __u64 data_tail;         /* user-space written tail */
    __u64 data_offset;       /* where the buffer starts */
    __u64 data_size;         /* data buffer size */
    __u64 aux_head;
    __u64 aux_tail;
    __u64 aux_offset;
    __u64 aux_size;
};
```

### offset
当使用 `rdpmc` 读取时，其值必须加上 `offset` 才是实际的事件次数。

### index
`rdpmc` 的参数

### cap_usr_time / cap_usr_rdpmc

从Linux 3.4 至 Linux 3.11期间，有BUG：
两个字段在被定义时都指向了同一个bit，无法区分。

### cap_user_rdpmc (since Linux 3.12)
如果硬件支持在**用户态**读取PMC寄存器(`rdpmc`)，则可以通过以下代码读取:
```c
u32 seq, time_mult, time_shift, idx, width;
u64 count, enabled, running;
u64 cyc, time_offset;

do {
    seq = pc->lock;
    barrier();
    enabled = pc->time_enabled;
    running = pc->time_running;

    if (pc->cap_usr_time && enabled != running) {
        cyc = rdtsc();
        time_offset = pc->time_offset;
        time_mult   = pc->time_mult;
        time_shift  = pc->time_shift;
    }

    idx = pc->index;
    count = pc->offset;

    if (pc->cap_usr_rdpmc && idx) {
        width = pc->pmc_width;
        count += rdpmc(idx - 1);
    }

    barrier();
} while (pc->lock != seq);
```

### cap_user_time (since Linux 3.12)
该bit表明有一个*constant, nostop timestamp counter* (TSC)

### pmc_width

如果 `cap_usr_rdpmc`，该字段表明 `rdpmc` 读取的值的宽度，可以用来做 sign extend:
```c
pmc <<= 64 - pmc_width;
pmc >>= 64 - pmc_width; // signed shift right
count += pmc;
```

### time_shift, time_mult, time_offset

用以计算从 `time_enabled` 以来的时间(纳秒)：(接上一段代码)
```c
u64 quot, rem;
u64 delta;

quot  = cyc >> time_shift;
rem   = cyc & (((u64)1 << time_shift) - 1);
delta = time_offset + quot * time_mult +
        ((rem * time_mult) >> time_shift);

// This delta can then be added to enabled and possible running
enabled += delta;
if (idx)
    running += delta;
quot  = count / running;
rem   = count % running;
count = quot * enabled + (rem * enabled) / running;

```

### time_zero (since Linux 3.12)

该字段可用于通过 `rdtsc` 计算时间戳

### data_head

该字段表示数据段的首地址，单调递增，需要**手动**求余。
在读取该字段后，用户应当调用 `rmb()` 保证读的内存序。

### data_tail

当环形缓冲区可写时(`PROT_WRITE`)，该字段应当由用户填写，用以表明最后读取的位置。这种情况下，内核不会覆写未读取的数据。

### data_offset, data_size (since Linux 4.1)

## ring-buffer (后续2^n页)

```c
struct perf_event_header {
    __u32   type;
    __u16   misc;
    __u16   size;
};
```

有如下典型的 `type`:
```c
// PERF_RECORD_MMAP:
// The MMAP events record the PROT_EXEC mappings
struct {
    struct perf_event_header header;
    u32    pid, tid;
    u64    addr;
    u64    len;
    u64    pgoff;
    char   filename[];
};

// PERF_RECORD_READ:
// This record indicates a read event.
struct {
    struct perf_event_header header;
    u32    pid, tid;
    struct read_format values;
    struct sample_id sample_id;
};

// PERF_RECORD_SAMPLE:
// This record indicates a sample
struct {
    struct perf_event_header header;
    u64    sample_id;   /* if PERF_SAMPLE_IDENTIFIER */
    u64    ip;          /* if PERF_SAMPLE_IP */
    u32    pid, tid;    /* if PERF_SAMPLE_TID */
    u64    time;        /* if PERF_SAMPLE_TIME */
    u64    addr;        /* if PERF_SAMPLE_ADDR */
    u64    id;          /* if PERF_SAMPLE_ID */
    u64    stream_id;   /* if PERF_SAMPLE_STREAM_ID */
    u32    cpu, res;    /* if PERF_SAMPLE_CPU */
    u64    period;      /* if PERF_SAMPLE_PERIOD */
    struct read_format v;
                        /* if PERF_SAMPLE_READ */
    u64    nr;          /* if PERF_SAMPLE_CALLCHAIN */
    u64    ips[nr];     /* if PERF_SAMPLE_CALLCHAIN */
    u32    size;        /* if PERF_SAMPLE_RAW */
    char   data[size];  /* if PERF_SAMPLE_RAW */
    u64    bnr;         /* if PERF_SAMPLE_BRANCH_STACK */
    struct perf_branch_entry lbr[bnr];
                        /* if PERF_SAMPLE_BRANCH_STACK */
    u64    abi;         /* if PERF_SAMPLE_REGS_USER */
    u64    regs[weight(mask)];
                        /* if PERF_SAMPLE_REGS_USER */
    u64    size;        /* if PERF_SAMPLE_STACK_USER */
    char   data[size];  /* if PERF_SAMPLE_STACK_USER */
    u64    dyn_size;    /* if PERF_SAMPLE_STACK_USER &&
                            size != 0 */
    u64    weight;      /* if PERF_SAMPLE_WEIGHT */
    u64    data_src;    /* if PERF_SAMPLE_DATA_SRC */
    u64    transaction; /* if PERF_SAMPLE_TRANSACTION */
    u64    abi;         /* if PERF_SAMPLE_REGS_INTR */
    u64    regs[weight(mask)];
                        /* if PERF_SAMPLE_REGS_INTR */
    u64    phys_addr;   /* if PERF_SAMPLE_PHYS_ADDR */
    u64    cgroup;      /* if PERF_SAMPLE_CGROUP */
};

// PERF_RECORD_SWITCH: (since Linux 4.3)
// This record indicates a context switch has happened.
// The PERF_RECORD_MISC_SWITCH_OUT bit in the misc field
// indicates whether it was a context switch into or away
// from the current process
struct {
    struct perf_event_header header;
    struct sample_id sample_id;
};
```

---

# Overflow handling

溢出事件可以被 `poll`, `select`, `epoll` 等函数捕获。或者也可以通过 `signal handler` 捕获。仅在采样模式会产生溢出事件

---

# rdpmc instruction

自 Linux 3.4 版本以后开始支持

> Note that using rdpmc is not necessarily faster than other methods for reading event values.

如果支持 `rdpmc`，任何其他进程也可以通过该指令读取计数器。

但从 Linux 4.0 开始，只有在某个事件被注册在进程的上下文，才可以通过该指令读取。如果想恢复旧的模式，需要将 `2` 写入 `/sys/devices/cpu/rdpmc`

---

# perf_event ioctl calls

- `PERF_EVENT_IOC_ENABLE`
- `PERF_EVENT_IOC_DISABLE`
- `PERF_EVENT_IOC_RESET` 
- ...

如果以上举例的三个同时设置了 `PERF_IOC_FLAG_GROUP`，那么会对该组下的事件操作。

---

# perf_event related configuration files

## /proc/sys/kernel/perf_event_max_sample_rate 
最大采样频率

## /proc/sys/kernel/perf_event_max_stack
最大调用栈深度

## /sys/bus/event_source/devices/*/type 
可填入`perf_event_attr`中的 type 字段，参考 [dynamic PMU](#type)

## /sys/bus/event_source/devices/cpu/rdpmc
如果值为 `1`，代表可以在用户态通过 `rdpmc` 指令获取计数器的值。可以通过写 `0` 关闭。

Linux 4.0 后，`1` 代表仅允许读取进程被激活的事件，`2` 代表旧行为。

## /sys/bus/event_source/devices/*/format/ (since Linux 3.4)
该目录下的每个文件名都代表一个对应 config 的域。其中的值用以表示该域所对应的 `bits`，例如 `config1:1,6-10,44`。 [config](#config)

## /sys/bus/event_source/devices/*/events/ (since Linux 3.4)
里面定义的事件不一定包含所有PMU支持的事件