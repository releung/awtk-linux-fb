自己使用的步奏记录:
# 1. 总体编译
根据 `https://github.com/releung/awtk-linux-fb/README.md` 操作

先整体编译一次, 没问题后, 再继续.

编译完成后, 在 awtk-linux-fb/build 中会生成 demo 的可执行文件,
这些文件是可以放到对应板子上运行的, 但是除了 copy awtk-linux-fb/build 到板子上外,
还需要 copy tslib 的 lib ts.conf 以及 awtk/demos/assets(inc目录可以不用 copy) 目录到 awtk-linux-fb/build/bin 里面,
因为 demo 运行的时候要在当前目录去查找 assets 里面的资源

中间可能会设计交叉编译 tslib 的, 我使用 tslib-1.19.tar.xz 版本没问题.具体的交叉编译就不写了.

# 2. 板子上 tslib 的环境设置
## 1) tslib 相关 export
```bash
export export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/tslib/ #tslib 库路径添加到 LD_LIBRARY_PATH 路径中
export TSLIB='/usr/local/tslib/'
export TSLIB_CONFFILE='/usr/local/tslib/etc/ts.conf'
export TSLIB_FBDEVICE='/dev/fb0'			# lcd panel 的设备
export TSLIB_GRAB_ENABLE='0'
export TSLIB_PLUGINDIR='/usr/local/tslib/lib/ts'
export TSLIB_TSDEVICE='/dev/input/event3'	# 触摸屏的 event 设备
```
## 2) ts.conf
直接用这个就行:
```conf
# Access plugins
################

module_raw input
module pthres pmin=1
module dejitter delta=100

#module linear
```
## 3) 测试 tslib
最好先在板子上测试 ts_calibrate 没问题后, 再继续
# 3. 应用程序 demo 编译
```bash
cd awtk-linux-fb/
git clone https://github.com/zlgopen/awtk-examples
```

编译 `awtk-examples/Chart-Demo`

修改 awtk-linux-fb/SConstruct
```python
APP_NAME = 'awtk-examples/Chart-Demo'
```

修改 `awtk-linux-fb/release.sh`
```python
APP_ROOT='awtk-examples/Chart-Demo'
```

这个时候执行 awtk-linux-fb$ scons -j24 会遇到错误:
```c
awtk-examples/Chart-Demo/src/window_pie.c: In function 'on_close':
awtk-examples/Chart-Demo/src/window_pie.c:325:3: error: 'for' loop initial declarations are only allowed in C99 mode
   for (int i = 0; i < ARRAY_SIZE(save_pie_exploded); i++) {
```
去修改 `awtk-linux-fb/SConstruct` 就好了 :
```python
OS_FLAGS='-g -Wall -Os -std=gnu99 '
```
生成的可执行文件在 awtk-examples/Chart-Demo/bin/ 中的 demo 文件,
将 awtk-examples/Chart-Demo/bin/demo 和 awtk-examples/Chart-Demo/res_800_480/ copy 到板子的同一个目录下, 就可以执行了

awtk-examples 其他的也是一样的操作

# 4. 让 awtk/demos/ 程序可以触摸操作
之前编译到  build/bin 目录中的 awtk/demos/ 程序是不能用触摸操作的, 需要以下操作才能使用触摸:

```bash
# awtk-hello 是 demo 程序, 用来参考
awtk-linux-fb$git clone https://github.com/zlgopen/awtk-hello.git

awtk-linux-fb$ awtk/demos/ awtk_demos
awtk_demos$mkdir src res
awtk_demos$mv SConscript assets.h  common.inc *.c demoui_web.json vg_common.inc src/
awtk_demos$cp ../awtk-hello/SConstruct ./
```
然后执行:
修改 `awtk-linux-fb/SConstruct`
```python
APP_NAME = 'awtk_demos'
```
修改 awtk-linux-fb/release.sh
```python
APP_ROOT='awtk_demos'
```
```bash
awtk-linux-fb$ scons -j24
```

然后会从新在 build/bin/ 中生成新的 demoui demovg demo_animator demo_desktop preview_ui demo1 demotr demo_thread
这些可执行文件都是可以使用触摸的

# 5. 打印设置
## 1)
	// 在 SConstruct 中的 COMMON_CCFLAGS 去掉 -DHAS_STDIO 就可以去掉 log_debug 的打印信息
	// 上句在编译的时候，会有错误产生，暂时用下面方法去掉

	log_debug 的设置宏在 awtk/src/tkc/types_def.h 中, 可以自己修改去掉打印
## 2)

触摸的时候会有打印信息出来, 可以去修改下面的打印函数, 将其去掉
```c
awtk-port/input_dispatcher.c
24:ret_t input_dispatch_print(void* ctx, const event_queue_req_t* e) {
```

# 附录
## 添加头文件路径方法
```python
env.Append(CCFLAGS = '-I.../awtk-linux-fb/demos/res/')
```
