---
title: "go里读文件的几种方法"
date: 2018-08-13T11:49:23+08:00
---


## 简单的读写方式： io/ioutil 包

- ioutil.ReadFile() : 这个对小文件比较简单，一次性把文件读入内存，在用strings.Split()按行切割，处理数据。
- ioutil.WriteFile() 和ReadFile一样，是把内存里的数据一次性刷入到文件，适合小数据场景。

## 原始的读写方式: os 包
- os.Open() 打开一个文件句柄，然后在句柄上可以进行各种文件操作。
操作方法有： 
```bash fp.Read() 、fp.Seek()、fp.Write()、fp.WriteString()```等。
通过打开的句柄，还可以用ioutil.NewReader()和ioutil.ReadAll()利用句柄获取内容。

## 代码示例

- 使用ioutil.ReadFile方式读取文件
```golang
func readFile1(path string) string {
    f, err := ioutil.ReadFile(path)
    if err != nil {
        fmt.Printf("%s\n", err)
        panic(err)
    }
    return string(f)
}
```

- 使用os.Open打开文件，用原始的fp.Read方法逐行读取内容
```golang
func readFile2(path string) string {
    fp, err := os.Open(path)
    if err != nil {
        panic(err)
    }
    defer fp.Close()

    chunks := make([]byte, 1024, 1024)
    buf := make([]byte, 1024)
    for {
        n, err := fp.Read(buf)
        if err != nil && err != io.EOF {
            panic(err)
        }
        if 0 == n {
            break
        }
        chunks = append(chunks, buf[:n]...)
    }
    return string(chunks)
}
```
- 通过os.Open打开文件，用bufio.NewReader读取文件
```golang
func readFile3(path string) string {
    fp, err := os.Open(path)
    if err != nil {
        panic(err)
    }
    defer fp.Close()
    reader := bufio.NewReader(fp)

    chunks := make([]byte, 1024, 1024)

    buf := make([]byte, 1024)
    for {
        n, err := reader.Read(buf)
        if err != nil && err != io.EOF {
            panic(err)
        }
        if 0 == n {
            break
        }
        chunks = append(chunks, buf[:n]...)
    }
    return string(chunks)
}
```

- 通过os.Open()打开文件，用 ioutil.ReadAll()读取文件
```golang
func readFile4(path string) string {
    fp, err := os.Open(path)
    if err != nil {
        panic(err)
    }
    defer fp.Close()
    data, err := ioutil.ReadAll(fp)
    if err == nil {
        return string(data)
    }
    fmt.Println(err)
    return ""
}
```
## 总结
对于小文件操作，用ioutil.ReadFile非常方便， 对大文件操作，可以用os.Open()打开文件后，进行逐行读取。