# Typora学习

**无序的列表**
- tfboys
- 杨洋
- 我爱你
  - 111
  - 222

:cry:😅😂😉🐻🐽🐧🐝

~~删除线~~
~~aaa~~

<u>下划线</u>

**粗体**

---



*斜体*

```
1. 步骤1
2. 步骤2  
```

```
- 步骤3
- ssss
- 步骤4
```


```
- [ ] 洗脸
- [ ] shua ya
- [x] 吃饭
```

---

## 链接 

1. 普通链接
```
用`[显示内容](链接地址 "Title")` 就OK了，`cmd` + 左键可以
```
   	[百度](www.baidu.com "度娘") 
2. 引用式链接
    [163](#mailurl)

```
[163]_[]
[163]:http://mail.163.com
```
​	####<span id="mailurl">http://mail.163.com</span>

​	[163邮箱](mail.163.com)

​	http://www.baidu.com

# 表格

普通表格

```
| ID  | name  | age |
| --- | ---   | --- |
| 99  | Frank | 28  |
```

| ID   | name  | age  |
| :--- | :---- | :--: |
| 99   | Frank |  28  |

| #    | 姓名   | 年龄   |
| ---- | :--- | ---- |
| 1    | 但丁   | 32   |
| 2    | 蛇🐍  | 31   |

# 图片

图片比链接多了 `!`在前面[^说明1]
```html
![图片的 alt 内容](/user/dante/img/a.png "图片不存在时的title")
```

![这是一个新世界](a.png "新世界丢失了")



# YAML

在第一行使用 `-` 回车就可以写了

```yaml
spring: 
  database: 
    url: localhost
    username: dante
    password: 123456
```



## 张婉清

```sql
select now() from dual 
select GETDATE()
```

###  高亮语法

`==高亮特性==` 

==高亮特性==

## 流程图

-  序列图

```sequence
Alice->Bob: Hello Bob, how are you?

Note right of Bob: Bob thinks

Bob-->Alice: I am good thanks!
```

- 流程图

  ```flow
  st=>start: 开始
  op=>operation: 判断条件
  cond=>condition: 是否？
  e=>end: 结束

  st->op->cond
  cond(yes)->e
  cond(no)->op
  ```





- Mermaid 图

  ```mermaid
  %% Example of sequence diagram
    sequenceDiagram
      Alice->>Bob: Hello Bob, how are you?
      alt is sick
      Bob->>Alice: Not so good :(
      else is well
      Bob->>Alice: Feeling fresh like a daisy
      end
      opt Extra response
      Bob->>Alice: Thanks for asking
      end
  ```

  ​