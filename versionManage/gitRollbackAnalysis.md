## 从HEAD指针理解git和它的回退操作
### 从HEAD说起

![76b4be67-1cef-4b00-9f9c-604ce9b8489a](media/15108304297211/76b4be67-1cef-4b00-9f9c-604ce9b8489a.png)
#### 上图为是一张经典的git分支示例图，图中HEAD指向feature分支，说明当前所在分支为feature，简单来说就是：`你现在在哪儿，HEAD 就指向哪儿，所以 Git 才知道你在那儿`。但是HEAD指向的不一定是分支，也可能指向一个commit。这个应该怎么理解呢？
#### 参考[what-is-head-in-git](http://stackoverflow.com/questions/2304087/what-is-head-in-git )，我们可以查看.git/HEAD 文件，它保存了当前的分支
```
cat .git/HEAD
=>ref: refs/heads/master
```
#### 这里HEAD文件即是我们平时所说的HEAD指针，它存储了一个（refs）分支路径，这就是它指向分支的方式。继续查看文件的内容。

```
cat .git/refs/heads/master
=>f7d96e3977694657b233fffc45ef6be924a69a02
```
#### 可以看到分支存储了一个字符串的值，我们使用 cat-file 命令继续查看，内容如下。

```
git cat-file -p f7d96e3977694657b233fffc45ef6be924a69a02
=>tree 0287e77fcc9e3d377f0e6f0df4d39ecc2d8bf9b7
=>parent 58fedb2f921c76350fb13d990a8db8088da628d2
=>author 火禹 <hongyu.rhy@antfin.com> 1511788758 +0800
=>committer 火禹 <hongyu.rhy@antfin.com> 1511788758 +0800

=>for test
```
#### 这个字符串的值是什么呢 ，cat-file查看的内容是什么呢？我们有必要简单理解下git的原理

###git原理
#### 总所周知，git需要存储对每次提交的版本进行存储，为了节省存储的空间，git只会将同样的内容文件只保存了一份。这就引入了Sha-1算法。
#### 可以使用git命令计算文件的 sha-1 值。
```
echo 'hello world' | git hash-object --stdin
=>3b18e512dba79e4c8300dd08aeb37f8e728b8dad
```
#### 将文件的sha-1值作为文件的唯一 id，每个版本存储的只是物理文件的快照也就是通过sha-1计算出来的hash作为的索引值。
![quick](media/15108304297211/quick.png)

#### git存储的文件结构抽象为4种类型：blob、tree、commit、tag
1. blob：用来存放项目文件的内容，但是不包括文件的路径、名字、格式等其它描述信息。项目的任意文件的任意版本都是以blob的形式存放的。

2. tree：ree 用来表示目录。我们知道项目就是一个目录，目录中有文件、有子目录。因此 tree 中有 blob、子tree，且都是使用 sha-1值引用的。这是与目录对应的。从顶层的 tree 纵览整个树状的结构，叶子结点就是blob，表示文件的内容，非叶子结点表示项目的目录，顶层的 tree 对象就代表了当前项目的快照。

3. commit: 表示一次提交，有parent字段，用来引用父提交。指向了一个顶层 tree，表示了项目的快照，还有一些其它的信息，比如上一个提交，committer、author、message 等信息。

4. Tag： Tag非常像一个 commit 对象——包含一个标签，一组数据，一个消息和一个指针。最主要的区别就是 Tag 对象指向一个 commit 而不是一个 tree。它就像是一个分支引用，但是不会变化——永远指向同一个 commit，仅仅是提供一个更加友好的名字。
#### 更详细的信息点击这里[Git-对象](https://git-scm.com/book/zh/v1/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1)查看

#### 回过头来，我们可以看出cat-file得到的便是一个commit对象，除了标明提交信息以外，同时包含了tree对象的引用，和上一次的commit对象的引用。而.git/refs/heads/master中存储了commit对象的hash值。

```
git cat-file -p f7d96e3977694657b233fffc45ef6be924a69a02
//tree对象
=>tree 0287e77fcc9e3d377f0e6f0df4d39ecc2d8bf9b7
//上一次commit对象
=>parent 58fedb2f921c76350fb13d990a8db8088da628d2
=>author 火禹 <hongyu.rhy@antfin.com> 1511788758 +0800
=>committer 火禹 <hongyu.rhy@antfin.com> 1511788758 +0800

=>for test
```
查看tree对象。显示的是最近一次提交的项目文件结构的快照

```
git cat-file -p 0287e77fcc9e3d377f0e6f0df4d39ecc2d8bf9b7
=>100644 blob 5ab9270b2a72afa8ebb55ba5cee27ed45ac56ac4    README.md
=>040000 tree 55e957c008470f1ca265e228db91deea85cbf28b    browser
=>100644 blob 8ee97644746f6cd4176f61ddbf7dfc789416e67b    lala
=>040000 tree ca199d18337591bebefeda1d996c787ed6b18bc7    node
=>040000 tree 32c9de5eba93fed0ea9d69bd6a4761074f844c1f    versionManage

```
查看parent。显示的是上一次的commit对象，对于没有改动的文件，其hash值是一样的。

```
git cat-file -p 58fedb2f921c76350fb13d990a8db8088da628d2
=>tree c09ce8d920ed5ed2b2e38f760199a16ac07026d3
parent 4bfc2d805f8980bc7ebd861c094445ecbd5de561
=>author 火禹 <hongyu.rhy@antfin.com> 1511788736 +0800
=>committer 火禹 <hongyu.rhy@antfin.com> 1511788736 +0800

=> frist commit
```
看到这里有没有豁然开朗，那么同学们重点来了，我的理解是:HEAD指向的本就是commit，commit是体现当前提交的一个完整项目结构快照，同时包含一个指向上一次提交的指针，就像链表一样一环扣一环最终形成一条完成的commit线路，这就是分支。HEAD指向的既是commit亦是branch。

