# Qemu Log

## Trace Log
* 文档：docs/devel/tracing.rst

### 命令行参数
```c
-trace [[enable=]<pattern>][,events=<file>][,file=<file>]
                specify tracing options
-d item1,...    enable logging of specified items (use '-d help' for a list of log items)
-D logfile      output log to logfile (default stderr)

-d help         查看支持哪些分类，分类之间用 , 分隔
-d trace:help   查看支持哪些 trace event
```
* `-d help` 对应的选项列表在源码 util/log.c `const QEMULogItem qemu_log_items[]` 数组中
* 比如 `-d tid` 对应到源码中的 `LOG_PER_THREAD` flags，设置它可以给每个 thread 产生一个带 `tid` 标识的 log 文件，此时 trace 文件名参数必须含有 `'%d'` 字符

### 编译选项

```sh
--enable-trace-backends=CHOICES
                        Set available tracing backends [log] (choices:
                        dtrace/ftrace/log/nop/simple/syslog/ust)

--with-trace-file=VALUE  Trace file prefix for simple backend [trace]
```
* 以上两个选项对应到构建脚本中的 `trace_backends` 和 `trace_file` 选项
* scripts/meson-buildoptions.py
```py
# Options whose name doesn't match the option for backwards compatibility
# reasons, because Meson gives them a funny name, or both
OPTION_NAMES = {
    "b_coverage": "gcov",
    "b_lto": "lto",
    "coroutine_backend": "with-coroutine",
    "debug": "debug-info",
    "malloc": "enable-malloc",
    "pkgversion": "with-pkgversion",
    "qemu_firmwarepath": "firmwarepath",
    "qemu_suffix": "with-suffix",
    "trace_backends": "enable-trace-backends",
    "trace_file": "with-trace-file",
}
```
### 构建
* trace/meson.build 会根据编译选项添加对应的 trace backends 源文件到目标中

#### meson.build

##### dtrace
* 如果 `trace_backends` 支持 `dtrace`，编译时查找 `dtrace` 和 `stap` 工具
```meson
if 'dtrace' in get_option('trace_backends')
  dtrace = find_program('dtrace', required: true)
  stap = find_program('stap', required: false)
  if stap.found()
    # Workaround to avoid dtrace(1) producing a file with 'hidden' symbol
    # visibility. Define STAP_SDT_V2 to produce 'default' symbol visibility
    # instead. QEMU --enable-modules depends on this because the SystemTap
    # semaphores are linked into the main binary and not the module's shared
    # object.
    add_global_arguments('-DSTAP_SDT_V2',
                         native: false, language: all_languages)
  endif
endif
```
* scripts/meson.build 会把 scripts/qemu-trace-stap 脚本安装到 `bindir` 作为 SystemTap 的前端工具
##### ftrace
* 只有 Linux 支持 `ftrace` 作为 backend
```meson
# Tracing backends
if 'ftrace' in get_option('trace_backends') and targetos != 'linux'
  error('ftrace is supported only on Linux')
endif
```
##### syslog
* 编译一个调用 `openlog()` 和 `syslog()` 的小程序测试系统的是否支持 `syslog` 开发
```meson
if 'syslog' in get_option('trace_backends') and not cc.compiles('''
    #include <syslog.h>
    int main(void) {
        openlog("qemu", LOG_PID, LOG_DAEMON);
        syslog(LOG_INFO, "configure");
        return 0;
    }''')
  error('syslog is not supported on this system')
endif
```
##### lttng
* 选项中如有 `ust` 则检测 `lttng-ust` 的支持
```meson
if 'ust' in get_option('trace_backends')
  lttng = dependency('lttng-ust', required: true, version: '>= 2.1',
                     method: 'pkg-config')
endif
```

##### build/config-host.h 头文件
* 根据 `--enable-trace-backends` 的值，在 build/config-host.h 文件中生成 `#define CONFIG_TRACE_LOG`，`#define  CONFIG_TRACE_SIMPLE` 类似的宏定义
```meson
foreach k : get_option('trace_backends')
  config_host_data.set('CONFIG_TRACE_' + k.to_upper(), true)
endforeach
```
* 根据 `--with-trace-file` 的值，在 build/config-host.h 文件中生成 `#define CONFIG_TRACE_FILE "trace"` 类似的宏定义
```meson
config_host_data.set_quoted('CONFIG_TRACE_FILE', get_option('trace_file'))
```

### Trace backends
* 构建时会根据 `--enable-trace-backends=` 的选择，利用 `scripts/tracetool.py` 脚本自动生成冗长的 trace event 代码，并且保持 trace event 的声明与 trace backend 独立
  * 在 `build/trace` 目录下可看到这些生成的文件
* Trace event 不与某个特定的 trace backend 紧耦合，例如 LTTng 或 SystemTap
* 如果多个 backend 都启用，trace 都会给它们发送
* 如果没有显示地指明 backends，默认配置为 `"log"` backend

### 运行

#### 初始化 Trace Backends
```c
//linux-user/main.c
main()
   //util/log.c
-> qemu_set_log_filename_flags(last_log_filename, last_log_mask | (enable_strace * LOG_STRACE), &error_fatal)
   -> qemu_set_log_internal(name, true, flags, errp) //-D 参数设置的 log 文件名
   //trace/control.c
-> trace_init_backends()
   -> st_init()
   -> ftrace_init()
   -> openlog(NULL, LOG_PID, LOG_DAEMON)
-> trace_init_file()
```

* `-D` 和 `-trace file=<file>` 都可以指定 trace event 输出的文件，它们对应的处理函数如下：
  * `-D` 参数直接赋给 `last_log_filename` 变量
  * `-trace` 的处理函数则是 `trace_opt_parse()` 去做进一步的解析
* linux-user/main.c
```c
static void handle_arg_log_filename(const char *arg)
{
    last_log_filename = arg;
}
...
static void handle_arg_trace(const char *arg)
{
    trace_opt_parse(arg);
}
...
static const struct qemu_argument arg_table[] = {
...
    {"d",          "QEMU_LOG",         true,  handle_arg_log,
     "item[,...]", "enable logging of specified items "
     "(use '-d help' for a list of items)"},
...
    {"D",          "QEMU_LOG_FILENAME", true, handle_arg_log_filename,
     "logfile",     "write logs to 'logfile' (default stderr)"},
...
    {"trace",      "QEMU_TRACE",       true,  handle_arg_trace,
     "",           "[[enable=]<pattern>][,events=<file>][,file=<file>]"},
...
};
```
* 使用 `-trace` 选项会让 `init_trace_on_startup = true`，该值影响到 simple backend 的文件是否创建
* 在 `trace_opt_parse()` 中，`-trace file=<file>` 参数的值构造出 `trace_opts_file` 文件名  
  * 如果不用该子参数，则会根据规则生成默认的 `trace_opts_file` 文件名，见后面的分析
```cpp
void trace_opt_parse(const char *optstr)
{
    QemuOpts *opts = qemu_opts_parse_noisily(qemu_find_opts("trace"),
                                             optstr, true);
    if (!opts) {
        exit(1);
    }
    if (qemu_opt_get(opts, "enable")) {
        trace_enable_events(qemu_opt_get(opts, "enable"));
    }
    trace_init_events(qemu_opt_get(opts, "events"));
    init_trace_on_startup = true; //用了 -trace 选项就会被设为 true
    g_free(trace_opts_file);
    trace_opts_file = g_strdup(qemu_opt_get(opts, "file"));
    qemu_opts_del(opts);
}
```
* 初始化 trace 文件
```cpp
void trace_init_file(void)
{
#ifdef CONFIG_TRACE_SIMPLE
    st_set_trace_file(trace_opts_file);
    if (init_trace_on_startup) { //用了 -trace 选项就会启用 trace_file_name
        st_set_trace_file_enabled(true);
    }
#elif defined CONFIG_TRACE_LOG
    /*
     * If both the simple and the log backends are enabled, "--trace file"
     * only applies to the simple backend; use "-D" for the log
     * backend. However we should only override -D if we actually have
     * something to override it with.
     */
    if (trace_opts_file) {
        qemu_set_log_filename(trace_opts_file, &error_fatal);
    }
#else
    if (trace_opts_file) {
        fprintf(stderr, "error: --trace file=...: "
                "option not supported by the selected tracing backends\n");
        exit(1);
    }
#endif
}
```
* 结合代码，函数中的注释的意思是说
* 如果构建时同时启用了 `simple` 和 `log` backends（`--enable-trace-backends=log,simple`），那么启动时 `"--trace file"` 选项会应用于 simple backend，`-D` 选项用于 log backend
  * 此时，trace event 会输出到 `"--trace file"` 指定的 simple 文件，原有的其他 `qemu_log()` 会输出到 `-D` 指定的文件，相互之间不影响
  * **注意**：因为 trace event 也指定了 `log` 作为 backend，因此生成的 trace event 代码中也会调用 `qemu_log()` 将 trace event 也输出到 `-D` 指定的文件
* 如果构建时只启用了 `log` 而不启用 `simple` backends（`--enable-trace-backends=log,...`），那么启动时如果有 `"--trace file"` 选项，则会覆盖 `-D logfile` 的设置

#### Simple Backend
```c
//trace/simple.c
st_init()
-> trace_thread_create(writeout_thread)
   -> g_thread_new("trace-thread", fn, NULL)
```

##### `st_set_trace_file()` 设置跟踪文件名
* `st_set_trace_file()` 设置 trace 文件名 `trace_file_name`，如果参数 `file` 为空，则是 `--with-trace-file=VALUE` 的值（默认是`"trace"`）拼接 `pid` 作为文件名
```cpp
/**
 * Set the name of a trace file
 *
 * @file        The trace file name or NULL for the default name-<pid> set at
 *              config time
 */
void st_set_trace_file(const char *file)
{   //关闭 trace 文件并返回原来 trace 文件的是否开启
    bool saved_enable = st_set_trace_file_enabled(false);

    g_free(trace_file_name);
    //修改 trace 文件名
    if (!file) {
        /* Type cast needed for Windows where getpid() returns an int. */
        trace_file_name = g_strdup_printf(CONFIG_TRACE_FILE "-" FMT_pid, (pid_t)getpid());
    } else {
        trace_file_name = g_strdup_printf("%s", file);
    }
    //恢复原来 trace 文件的开关状态
    st_set_trace_file_enabled(saved_enable);
}
```

### 实例分析

* qemu 这里以 `--enable-trace-backends=log,simple` 进行编译，以下面最左边一列的参数启动，trace event 的 backend 产生的 log 输出如下：

参数                                   | log backend 输出  | simple backend 输出
---------------------------------------|------------------|--------------------
`-d trace:kvm*`                        | stderr           | 无
`-d trace:kvm* -D qemu_log`            | qemu_log         | 无
`-d trace:kvm* --trace file=trace_log` | stderr           | trace_log
`-d trace:kvm* --trace file=trace_log -D qemu_log` | qemu_log | trace_log
`--trace trace:kvm* -D qemu_log`       | qemu_log         | trace-`<pid>`
`--trace trace:kvm* -d tid -D "qemu_log.%d"` | qemu_log.`<tids>` | trace-`<pid>`
`-d tid,trace:kvm* -D "qemu_log.%d"`   | qemu_log.`<tids>` | 无

## 监控报告
* 另外还有监控报告打印，通过 `error_report()`、`warn_report()`、`info_report()` 等接口函数打印
* glib logging 共享库也用上面的接口打印，见 `error_init() -> g_log_set_default_handler(qemu_log_func, NULL)`
* 如果配置了 `-monitor` 且不是 `-qmp`，则输出到 monitor 上，否则回退到 `stderr`

## Qemu_log
* `qemu_log()` 打印输出由 `-D logfile` 参数控制，默认输出到 `stderr`
* 另外有 `qemu_log_mask()` 和 `qemu_log_mask_and_addr()` 宏可以受 loglevel mask 控制是否输出
  * include/qemu/log.h
```cpp
/* Logging functions: */

/* log only if a bit is set on the current loglevel mask:
 * @mask: bit to check in the mask
 * @fmt: printf-style format string
 * @args: optional arguments for format string
 */
#define qemu_log_mask(MASK, FMT, ...)                   \
    do {                                                \
        if (unlikely(qemu_loglevel_mask(MASK))) {       \
            qemu_log(FMT, ## __VA_ARGS__);              \
        }                                               \
    } while (0)

/* log only if a bit is set on the current loglevel mask
 * and we are in the address range we care about:
 * @mask: bit to check in the mask
 * @addr: address to check in dfilter
 * @fmt: printf-style format string
 * @args: optional arguments for format string
 */
#define qemu_log_mask_and_addr(MASK, ADDR, FMT, ...)    \
    do {                                                \
        if (unlikely(qemu_loglevel_mask(MASK)) &&       \
                     qemu_log_in_addr_range(ADDR)) {    \
            qemu_log(FMT, ## __VA_ARGS__);              \
        }                                               \
    } while (0)
```
* 从前文中的 `struct qemu_argument arg_table[]` 数组可知 `-d` 参数的解析函数是 `handle_arg_log()`
  * linux-user/main.c
```cpp
static void handle_arg_log(const char *arg)
{
    last_log_mask = qemu_str_to_log_mask(arg);
    if (!last_log_mask) {
        qemu_print_log_usage(stdout);
        exit(EXIT_FAILURE);
    }
}
```
* `qemu_str_to_log_mask()` 函数根据 `-d` 的输入（`,` 分隔的）字符串和数组 `QEMULogItem qemu_log_items[]` 的信息，转为 log mask，赋给 `last_log_mask` 变量
* 该 log mask 在 [main() -> qemu_set_log_filename_flags() -> qemu_set_log_internal()](#初始化-trace-backends) 的调用中赋给全局变量 `qemu_loglevel`
* 这样在 `qemu_loglevel_mask()` 函数中就可以通过 `qemu_loglevel` 控制 
  * include/qemu/log-for-trace.h
```cpp
/* Private global variable, don't use */
extern int qemu_loglevel;

#define LOG_TRACE          (1 << 15)

/* Returns true if a bit is set in the current loglevel mask */
static inline bool qemu_loglevel_mask(int mask)
{
    return (qemu_loglevel & mask) != 0;
}
```