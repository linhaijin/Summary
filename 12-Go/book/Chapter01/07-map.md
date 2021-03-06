
Map
=========

- 引用类型。与 slice 一样，在函数之间传递 map 相当廉价。
因此在函数中改变了 map，那么所有引用 map 的地方都会改变。
- 不能保证迭代次序和添加时相同，因为 map 使用哈希表实现。
- 键必须是支持相等运算符 `==, !=` 的类型，
如 number，string，pointer，array，不包含 slice 的 struct，及对应的 interface。
而 slice，map，function 和 包含 slice 的 struct 类型不可以作为 map 的键。
值可以是任意类型。

```go
m := map[int]struct {
    name string
    age int
}{
    1:  {"user1", 10}, // 可省略元素类型
    2:  {"user2", 20},
}
println(m[1].name)
```

- go 初始化可使用内建函数 make，也可使用 map 字面值：

```go
// 1. make
dict := make(map[string]int)
// 2. 字面值
dict := map[string]string{"Red": "#da1337", "Orange": "#e95a22"}
```

- 建议使用字面值初始化，避免使用 make。除非必要场合，如知道分配的大小：

```go
make(map[int]string, 3)
make([]int, 0, 5)
```

- 如果不初始化 map，会创建一个 nil map。nil map 不能存放键值对，否则会报运行时错误：

```go
// 未初始化
var colors map[string]string
colors["Red"] = "#da1337"
Runtime Error:
panic: runtime error: assignment to entry in nil map

// 字面值初始化
colors := map[string]string{}
colors["Red"] = "#da1337"
```

- 可预先给 make 函数一个合理元素数量参数，有助于提升性能，避免后续操作时频繁扩张。

```go
m := make(map[string]int, 1000)
```

- 常见操作

```go
m := map[string]int{
    "a": 1,
}

if v, ok := m["a"]; ok {    // 判断 key 是否存在
    println(v)
}

println(m["c"])             // 对于不存在的 key，直接返回 \0，不会出错
m["b"] = 2                  // 新增或修改
delete(m, "c")              // 删除，如果 key 不存在,不会出错
println(len(m))             // 获取键值对数量，cap 无效

for k, v := range m {       // 迭代，可仅返回 key
    println(k, v)
}
```

- 从 map 中取回的是一个 value 的临时拷贝，对其成员的修改没有任何意义。

```go
type user struct{ name string }
m := map[int]user{
    1: {"user1"},
}
m[1].name = "Tom"         // Error: cannot assign to m[1].name

// 正确的做法是完整替换或使用指针
u := m[1]
u.name = "Tom"
m[1] = u                // 替换 value
m2 := map[int]*user{
    1: &user{"user1"},
}
m2[1].name = "Jack"     // 返回的是指针复制品。透过指针修改原对象是允许的
```

- 可以在迭代时安全删除键值。但如果期间有新增操作，结果则无法预知。

```go
for i := 0; i < 5; i++ {
    m := map[int]string{
        0:  "a", 1:  "a", 2:  "a", 3:  "a", 4:  "a",
        5:  "a", 6:  "a", 7:  "a", 8:  "a", 9:  "a",
    }
    for k := range m {
        m[k+k] = "x"
        delete(m, k)
}
    fmt.Println(m)
}

// 输出：
map[12:x 16:x 2:x 6:x 10:x 14:x 18:x]
map[12:x 16:x 20:x 28:x 36:x]
map[12:x 16:x 2:x 6:x 10:x 14:x 18:x]
map[12:x 16:x 2:x 6:x 10:x 14:x 18:x]
map[12:x 16:x 20:x 28:x 36:x]
```
