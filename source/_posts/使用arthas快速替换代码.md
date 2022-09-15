# 命令 retransform

使用命令 `retransform`可以替换指定的`.class`文件，做到快速生效更改的代码

##### 操作步骤

- 本地编辑好代码后，构建编译代码

- 使用如下命令转换class文件为base64，保存到txt中

  ```shell
  base64 < Test.class > result.txt
  ```

- 到服务器上，新建并编辑`result.txt`，复制本地的内容，粘贴再保存

- 把服务器上的 `result.txt`还原为`.class`

   ```
   base64 -d < result.txt > Test.class
   ```
   
- 替换

   ```shell
   retransform /tmp/MathGame.class
   ```

   

##### retransform的限制

   - 不允许新增加field/method

   - 正在跑的函数，没有退出不能生效，比如下面新增加的`System.out.println`，只有`run()`函数里的会生效

##### 查看并清除retransform的影响

```shell
查看：retransform -l
删除指定retransform ：retransform -d 1
删除所有：retransform --deleteAll
```

