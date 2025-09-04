# Makefile常用语法

- wildcard

```makefile
SRC := $(wildcard *.c) # 提取出当前目录下的c文件
```

- patsubst

```makefile
OBJ := $(patsubst %c, %o, $(SRC)) 		# 格式化修改变量内容，.c->.o
```

- filter，filter-out

```makefile
cur = $(filter %.c, path/$(SRC)) 		# 匹配保留
cur = $(filter-out %.c, path/$(SRC))	# 匹配排除
```

- dir, notdir, basename, suffix

```makefile
$(dir src/main.c) 		# -> src/
$(nodir src/main.c) 	# -> main.c
$(basename src/main.c) 	# -> src/main
$(suffix src/main.c) 	# -> .c
```

- if, or, and

```makefile
$(if $(Judge), yes, no)
$(or $(A),$(B),default)	# 返回第一个非空
$(and $(A),$(B))		# 所有非空，返回最后一个
```

- foreach

```makefile
$(foreach var, $(LIST), $(info var=$(var)))
```

## 项目中的makefile

- 顶层Makefile

顶层目录中的Makefile文件定义了编译器相关的变量。

每个dir下的Makefile定义当前文件夹的`obj-y +=` `.o`文件和路径。

`all`规则：进入`Makefile.build`执行该文件的规则。执行完成之后链接顶层`built-in.o`文件为可执行文件。

```makefile
all : 
	make -C ./ -f $(TOPDIR)/Makefile.build
	$(CC) $(LDFLAGS) -o $(TARGET) built-in.o
```

- Makefile.build

1. 伪目标build，目的是执行规则`$(subdir-y)`和`built-in.o`

```makefile
__build : $(subdir-y) built-in.o
```

2. $(subdir-y)规则，实现递归

```makefile
$(subdir-y):
	make -C $@ -f $(TOPDIR)/Makefile.build
```

3. built-in.o 把`cur_objs`和`subdir_objs`链接成当前文件中的`build-in.o`文件。

```makefile
built-in.o : $(cur_objs) $(subdir_objs)
	$(LD) -r -o $@ $^
```

4. .o文件的编译

```makefile
%.o : %.c
	$(CC) $(CFLAGS) -Wp,-MD,$(dep_file) -c -o $@ $<
```

- 关键变量的提取

```makefile
__subdir-y	:= $(patsubst %/,%,$(filter %/, $(obj-y)))
subdir-y	+= $(__subdir-y)

subdir_objs := $(foreach f,$(subdir-y),$(f)/built-in.o)

cur_objs := $(filter-out %/, $(obj-y))
```

