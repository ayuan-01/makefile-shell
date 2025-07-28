# Makefile常用语法

- wildcard

```makefile
SRC := $(wildcard *.c) # 提取出当前目录下的c文件
```

- patsubst

```makefile
OBJ := $(patsubst %c, %o, $(SRC)) # 格式化修改变量内容，.c->.o
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

