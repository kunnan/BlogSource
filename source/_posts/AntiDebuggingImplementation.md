---
title: AntiDebugging Implementation Notes
date: 2017-12-21 20:31:17
tags: LLVM
---
The current implementation of AntiDebugging is simply inject Inline Assemblies at function start and only works on iOS.
Since it's now possible to do sysent hook on (certain versions of) iOS. Inline ``ptrace()`` alone is no longer safe.
Here is a few more ideas to be implemented.
<!-- more -->
### Code Stolen From Places

```
BOOL isDebuggerPresent(){
    int name[4];             

    struct kinfo_proc info;
    size_t info_size = sizeof(info);

    info.kp_proc.p_flag = 0;

    name[0] = CTL_KERN;
    name[1] = KERN_PROC;
    name[2] = KERN_PROC_PID;
    name[3] = getpid();         

    if(sysctl(name, 4, &info, &info_size, NULL, 0) == -1){
        NSLog(@"sysctl error ...");
        return NO;
    }

    return ((info.kp_proc.p_flag & P_TRACED) != 0);
}
```

```
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_SIGNAL, SIGSTOP, 0, dispatch_get_main_queue());
dispatch_source_set_event_handler(source, ^{
    NSLog(@"SIGSTOP!!!");
    exit(0);
});
dispatch_resume(source);
```

```
struct macosx_exception_info{
    exception_mask_t masks[EXC_TYPES_COUNT];
    mach_port_t ports[EXC_TYPES_COUNT];
    exception_behavior_t behaviors[EXC_TYPES_COUNT];
    thread_state_flavor_t flavors[EXC_TYPES_COUNT];
    mach_msg_type_number_t cout;
};

struct macosx_exception_info *info = malloc(sizeof(struct macosx_exception_info));
task_get_exception_ports(mach_task_self(),
                                            EXC_MASK_ALL,
                                            info->masks,
                                            &info->cout,
                                            info->ports,
                                            info->behaviors,
                                            info->flavors);

for(uint32_t i = 0; i < info->cout; i ++){
    if(info->ports[i] != 0 || info->flavors[i] == THREAD_STATE_NONE){
        NSLog(@"debugger detected via exception ports (null port)!\n");
    }
}
```

```
if (isatty(1)) {
    NSLog(@"Being Debugged isatty");
}
```

```
if (!ioctl(1, TIOCGWINSZ)) {
    NSLog(@"Being Debugged ioctl");
}
```


### Ideas from @jmpews
Register Custom EH Pad and intentionally throw exceptions,they should be handled by our very own block.
