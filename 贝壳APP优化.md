# 在 kCFRunLoopAfterWaiting 通知监听者要处理事情的时候就把闲时停掉。

```objective-c
void dispatch_idle_task(bool inMainThread, NSString *taskId, dispatch_block_t task)
{
if (task == nil || taskId.length == 0) { return; }
id<LJIdleTaskProtocol> idleTaskEngine = [LJTasksManager.sharedInstance idleTaskEngine];
[idleTaskEngine enqueueIdleTask:^{
task();
} requireMainThread:inMainThread taskId:taskId callBack:^(uint64_t cost, NSDictionary *context) {
}];
}
- (id)enqueueIdleTask:(LJTaskBlock)idleTask
requireMainThread:(BOOL)isMainThread
taskId:(NSString *)taskId
callBack:(LJTaskContextBlock)contextCallBackBlock {
LJWrapperIdleTask *idleTaskWrapper = [[LJWrapperIdleTask alloc] initWithIdleTaskBlock:idleTask mainThreadOnly:isMainThread taskId:taskId callBack:contextCallBackBlock];

NSValue *value = nil;
if (idleTask && idleTaskWrapper) {
// 封装为 NSValue，用于后续取消任务
value = [NSValue valueWithNonretainedObject:idleTaskWrapper];
// 将闲时任务加入到队列
dispatch_sync(self.synQueue, ^{
[self.idleTasks addObject:idleTaskWrapper];
});
if (!self.observerRef) {
__weak typeof(self) weakSelf = self;
self.observerRef = [self.runloopObserver addMainThreadRunloopIdleObserver:^(CFRunLoopActivity activity) {
if (activity == kCFRunLoopBeforeWaiting) {
[weakSelf doIdleTask];
} else if (activity == kCFRunLoopAfterWaiting) {
weakSelf.isRunloopRuning = YES;
}
}];
}
}
return value;
}
```

# 调用起来也比较简单：

```objective-c
- (void)applicationLaunchWithOptions:(NSDictionary *)launchOptions {
_status = LJLaunch_App_STATUS_FINISH_LAUNCH;
// High priority tasks
for (Class appClass in self.highOrderedLaunchArray) {
dispatch_idle_task(true, NSStringFromClass(appClass), ^{
[appClass appLaunchWithOptions:launchOptions];
});
}
// low priority tasks
for (Class appClass in self.lowOrderedLaunchArray) {
dispatch_idle_task(true, NSStringFromClass(appClass), ^{
[appClass appLaunchWithOptions:launchOptions];
});
}

if (_shouldHandleAppBecomeActiveEvent) {
[self applicationDidBecomeActive:[UIApplication sharedApplication]];
}

[self doAsyncLaunchTask];
}

```

# 另外通过 DoraemonKit 也可以直接扫描。

```objective-c
@implementation LJLoadCounter
+ (void)load {
CFTimeInterval start = CFAbsoluteTimeGetCurrent();
gcosts = [NSMutableDictionary dictionary];

//1. 获取所有已加载的 images
int imageCount = (int)_dyld_image_count();
NSString *mainExecuteName = [[[NSBundle mainBundle].executablePath componentsSeparatedByString:@"/"] lastObject];
NSString *testLoadFramework = @"LJTestLoadFramework";
NSSet *needHookImage = [NSSet setWithObjects:testLoadFramework, mainExecuteName, nil];
for(int i = 0; i < imageCount; i++) {
const char *path = _dyld_get_image_name((unsigned) i);
NSString *imagePath = [NSString stringWithUTF8String:path];
NSString *dylib = [[imagePath componentsSeparatedByString:@"/"] lastObject];
//2. 通过路径筛选出 image
if ([needHookImage containsObject:dylib]) {
uint count = 0;

const char * *classes = objc_copyClassNamesForImage(path, &count);
for (int i = 0; i < count; i++) {
NSString *className = [NSString stringWithCString:classes[i] encoding:NSUTF8StringEncoding];
Class class = object_getClass(NSClassFromString(className));

unsigned int methodCount = 0;
Method * methods = class_copyMethodList(class, &methodCount);

for (unsigned int methodIndex = 0; methodIndex < methodCount; ++methodIndex) {
Method method = methods[methodIndex];
NSString *methodName = NSStringFromSelector(method_getName(method));
if ([methodName isEqualToString:@"load"]) {
__block void (*originalLoad)(__unsafe_unretained id, SEL) = NULL;
id newloadBlock = ^(__unsafe_unretained id self, SEL _cmd) {
CFTimeInterval start = CFAbsoluteTimeGetCurrent();
originalLoad(self, _cmd);
CFTimeInterval end = CFAbsoluteTimeGetCurrent();
CFTimeInterval cost = end - start;
gcosts[[NSString stringWithUTF8String:class_getName(self)]] = @(cost);

};
IMP newloadIMP = imp_implementationWithBlock(newloadBlock);
originalLoad = (__typeof__(originalLoad))method_setImplementation(method, newloadIMP);
}
}
}
}
}

CFTimeInterval end = CFAbsoluteTimeGetCurrent();
CFTimeInterval cost = end - start;
gcosts[[NSString stringWithUTF8String:class_getName(self)]] = @(cost);
}
@end
```

# 作用就是将函数或数据放入指定名为"section_name"对应的段中。例如

```objective-c
#define LJLoadableSegmentName "__DATA"
#define LJLoadableSectionName "LJLoadable"
#define LJLoadable __attribute((used, section(LJLoadableSegmentName "," LJLoadableSectionName)))
```

# 下⾯这段代码可以把函数地址放到 LJLoadable 段中。

```objective-c
typedef int (*LJLoadableFunctionCallback)(const char *);
typedef void (*LJLoadableFunctionTemplate)(LJLoadableFunctionCallback);
#define LJLoadableFunctionBegin(functionName) \
static void LJLoadable##functionName(LJLoadableFunctionCallback LJLoadableCallback){ \
if(0 != LJLoadableCallback(#functionName)) return;
#define LJLoadableFunctionEnd(functionName) \
} \
static LJLoadableFunctionTemplate varLJLoadable##functionName LJLoadable = LJLoadable##functionName;
```

# 调⽤方可以像下面这样，把原来在 +load 中的代码移植到两个宏 (LJLoadableFunctionBegin 和 LJLoadableFunctionEnd) 之间。

```objective-c
LJLoadableFunctionBegin(xxx)
// anything here
LJLoadableFunctionEnd(xxx)
```


# (1) 通过函数获得返回值的全局常量

```objective-c
int test() {
return 1;
}
int x = test();
```

# (2) 非编译时常量

```objective-c
// 结构体的⾮编译时常量
CGRect rect = CGRectZero;
// OC⾮编译时期常量
NSDictionary *dic = @{@"key" : @"value"};
```

# (3) __attribute((constructor))

```objective-c
static __attribute__((constructor))
void ljshl_MessageCellIdentifiers(void) {
kLJSHLMessageCellIdentifiers = @[
@"LJSHLMessageBotCell",
@"LJSHLMessageSystemCell",
@"LJSHLMessageUserCell",
@"LJSHLMessageSelfCell",
@"LJSHLMessageVIPCell",
@"LJSHLMessageSystemWelcomeCell"
];
}
```

# (4) C++ 全局类对象初始化


```objective-c
class testGlobalVar {
public:
testGlobalVar() {
std::cout<<"testGlobalVar"<<std::endl;
}
};
testGlobalVar var;
```

# (5) C++ 静态类对象初始化

```objective-c
class testGlobalVar {
public:
static testGlobalVar *globalVar;
testGlobalVar() {
std::cout<<"testGlobalVar"<<std::endl;
}
};
testGlobalVar *(testGlobalVar::globalVar) = new testGlobalVar;
```

# 原理:mod_init_func 的运⾏时机晚于 +load，可以在运⾏时 hook。核心代码如下：

```objective-c
+ (void)load
{
CFTimeInterval start = CFAbsoluteTimeGetCurrent();
g_initializer = new std::vector<MemoryType>();
g_cur_index = -1;
gcosts = [NSMutableDictionary dictionary];
Dl_info info;
dladdr((const void *)myInitFunc_Initializer, &info);
#ifndef __LP64__
const struct mach_header *mhp = (struct mach_header*)info.dli_fbase;
unsigned long size = 0;
MemoryType *memory = (uint32_t *)getsectiondata(mhp, "__DATA", "__mod_init_func", & size);
#else /* defined(__LP64__) */
const struct mach_header_64 *mhp = (struct mach_header_64*)info.dli_fbase;
unsigned long size = 0;
MemoryType *memory = (uint64_t *)getsectiondata(mhp, "__DATA", "__mod_init_func", & size);
#endif /* defined(__LP64__) */
for (int idx = 0; idx < size/sizeof(void*); ++idx) {
MemoryType original_ptr = memory[idx];
g_initializer->push_back(original_ptr);
struct dl_info func_info;
dladdr((void *)original_ptr, &func_info);
g_initializer_symbol.push_back(func_info.dli_sname);
memory[idx] = (MemoryType)myInitFunc_Initializer;
}
CFTimeInterval end = CFAbsoluteTimeGetCurrent();
CFTimeInterval cost = end - start;
gcosts[[NSString stringWithUTF8String:class_getName(self)]] = @(cost);
}
```

# ⽽对应⽂件号在 Object files 是能找到所属 bundle 和⽂件名的，进⽽能找到对应负责⼈。

```objective-c
import os
linkmap = "arm64_link_map.txt"
csv_file = "symbol.csv"
with open(linkmap) as lm:
lm_lines = lm.readlines()
dir_nums = []
sym_nums = []
bundle_nums = []
clazz_nums = []
cur = 0
with open(csv_file, "r") as f:
lines = f.readlines()
fix_lines = ""
for line in lines:
line = line.strip()
for lm_line in lm_lines:
if 'literal string' in lm_line:
continue
if 'non-lazy-pointer-to-local' in lm_line:
continue
if lm_line.endswith(' _%s\n' % line) or lm_line.endswith(' %s\n' % line):
if lm_line.startswith('0x'):
lm_infos = lm_line.split()
dir_num = lm_infos[2]
dir_nums.append(dir_num)
sym_nums.append(line)
print('wait...')
print('next...')
print('序号, 符号, 文件号, bundle 名, class 名')
for index in range(len(dir_nums)):
num = dir_nums[index]
line = sym_nums[index]
for dir_line in lm_lines:
if num in dir_line:
dir = dir_line.split('/')[-1]
bundles = dir.split('(')
bundle = bundles[0]
clazz = 'Default'
if len(bundles) > 1:
clazz = bundles[1].strip(')\n')
cur = cur + 1
print('%d ,%s, %s, %s, %s' % (cur, line, num, bundle, clazz))
bundle_nums.append(bundle)
clazz_nums.append(clazz)
break
```

# 则编译器会⾃动在⽤户定义的函数中插⼊__sanitizer_cov_trace_pc_guard 函数，借⽤这个机制我们可以获取到启动阶段所有⽤户⾃定义函数的调⽤顺序，并⽣成 order⽂件。核心代码如下：

```objective-c
@implementation LJAPPOrderFiles
+ (void)load {
initImageTextStartAndEndPos();
initCurrentImageSlide();
// 此处通知标识 app 启动已完成
NSNotificationCenter *center =[NSNotificationCenter defaultCenter];
[center addObserverForName:@"LJHomeViewControllerDidAppear"
object:nil
queue:[NSOperationQueue mainQueue]
usingBlock:^(NSNotification * _Nonnull note)
{
collectFinished = YES;
[LJAPPOrderFiles getOrderFiles:^(NSString *orderFilePath) {
NSLog(@"orderFilePath = %@", orderFilePath);
}];
}];
}
+ (void)getOrderFiles:(void(^)(NSString *orderFilePath))completion {
__sync_synchronize();
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.01 * NSEC_PER_SEC)), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{

NSMutableArray <NSString *> *functions = [NSMutableArray array];
while (YES) {
PCNode *node = OSAtomicDequeue(&queue, offsetof(PCNode, next));
if (node == NULL) {
break;
}
[functions addObject:[NSString stringWithFormat:@"%lX", ((vm_address_t)node->pc - LJCurImageSlide)]];
}
if (functions.count == 0) {
if (completion) {
completion(nil);
}
return;
}
NSMutableArray<NSString *> *calls = [NSMutableArray arrayWithCapacity:functions.count];
NSEnumerator *enumerator = [functions reverseObjectEnumerator];
NSString *obj;
while (obj = [enumerator nextObject]) {
if (![calls containsObject:obj]) {
[calls addObject:obj];
}
}
NSDictionary *result = @{@"orders": calls};
NSLog(@"Order:\n%@", result);

NSString *documentsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
NSString *filePath = [documentsPath stringByAppendingPathComponent:@"binary_layout_order_file.plist"];
BOOL success = [result writeToFile:filePath atomically:YES];
if (completion) {
completion(success ? filePath : nil);
}
});
}
@end
```

#  10.4.3 核心代码

```objective-c
#!/usr/bin/python
# -*- coding: utf-8 -*-
import os
import re
import time
from biplist import readPlist
def dic_2_oc_code(config):
s = ''
s += '@{ \n'
for key, value in config.items():
s += '@"' + key + '":'
if isinstance(value, str):
s += '@"' + value.replace('\\', '\\\\') + '"'
elif isinstance(value, unicode):
s += '@"' + value.encode('utf-8') + '"'
elif isinstance(value, bool):
if value:
s += " @YES "
else:
s += " @NO "
elif isinstance(value, list):
s += list_2_oc_code(value)
elif isinstance(value, dict):
s += dic_2_oc_code(value)
elif isinstance(value, int):
s += "@" + str(value)
else:
print(type(value))
s += ', \n\t'
s += ' }'
return s
def generate_oc_code(file_path, methods):
header = """
#import <UIKit/UIKit.h>
@interface LJLaunchConfigPlatform : NSObject
@end
@implementation LJLaunchConfigPlatform
+ (void)checkAppLaunchConfigCurrectly {
NSDictionary *config = [LJLaunchConfigPlatform launchConfig];
NSString *configPath = [[NSBundle mainBundle] pathForResource:@"AppLaunchConfig" ofType:@"plist"];
NSDictionary *appLaunchConfig = [NSDictionary dictionaryWithContentsOfFile:configPath];
BOOL result = [config isEqualToDictionary:appLaunchConfig];

}
+ (void)checkCode {
[LJLaunchConfigPlatform checkAppLaunchConfigCurrectly];
}
"""
footer = """

@end
"""
with open(file_path, 'w') as f:
f.write(header)
f.write("\n".join(methods))
f.write(footer)
```

# (4) synchronize 改为无锁的 dispatch_once, 防止并发访问锁可能引起的耗时，比如：

```objective-c
@synchronized (self) {
if (_lj_keychainID.length == 0) {
_lj_keychainID = [SAMKeychain passwordForService:@"lianjia_app_udid" account:@"udid"];
}
self.tempkeyChainID = _lj_keychainID;
}
```

# 改为：

```objective-c
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
if (_lj_keychainID.length == 0) {
_lj_keychainID = [SAMKeychain passwordForService:@"lianjia_app_udid" account:@"udid"];
}
self.tempkeyChainID = _lj_keychainID;
});
```

# (5) 避免直接 I/O 的方式读取 Info.plist，系统已经提供了内存读取的 API，比如：

```objective-c
NSString* File = [[NSBundle mainBundle] pathForResource:@"Info" ofType:@"plist"];
NSMutableDictionary* dict = [[NSMutableDictionary alloc] initWithContentsOfFile:File];
```

# 改为：

```objective-c
NSDictionary *dict = [[NSBundle mainBundle] infoDictionary];
_appName = [dict objectForKey:@"CFBundleDisplayName"];
```
