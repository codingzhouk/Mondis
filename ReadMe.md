# Mondis
Mondis is a key-value database powered by redis and add some new feature。
# 什么是Mondis
Mondis是一个key-value数据库，它很像redis，但是支持许多redis不支持的新特性。实际上，它的名字mondis就是取自mongodb与redis。后面将会说明这样做的原因。
下面，让我们学习如何来使用它。
# 选择mondis的理由
## mondis嵌套
mondis支持数据结构的任意嵌套，mondis键的取值只能是字符串，但是值的取值可以是string,list,set,zset,hash的任意一种。
mondis嵌套意味着list,zset的元素与hash类型键值对的值可以是另一个list,set,zset或者hash，就像json
那样。注意，set类型的元素只能是string，这是因为set底层采用hash表进行实现，而list,zset与hash没有默认的哈希函数。
## mondis多态命令
在redis里面，我们在操作的时候必须指定底层数据的类型，比如list命令全部以l开头，zset命令全部以z开头，hash命令全部以hash开头，
但是在mondis里面，这些统统不需要。mondis命令具有多态性，相同的命令在不同数据结构上的效果是不同的，只需要执行命令而无需关心底层数据结构是
什么。如果执行了不合适的命令，mondis会处理这种情况，不用担心崩溃。
## mondis定位命令
由于mondis支持任意嵌套，有时候我们要操作一个嵌套层数很深的数据对象，此时就需要用到定位命令。
定位命令是locate，它可以具有不定数量的参数。它的作用就是定位到当前要操作的数据对象上，然后执行操作命令。
多个locate命令之间需要以|分隔，看上去就像linux的管道命令。
## mondis查询
mondis查询类似于redis查询，但是不同的是mondis查询返回的格式是json。这使得mondis更方便使用。
## mondis持久化
mondis支持两种持久化方式，json与aof。json类似于rdb，但是持久化的格式是完全兼容的json，
这使得持久化文件方便迁移并解析。aof持久化则与redis的aof完全相同，除了命令格式。但是目前mondis还不支持aof重写。
## mondis底层实现相对于redis的改进
### list
在mondis里面，list还是用链表实现。不同的是mondis保存了对象指针与索引的双向映射，这样虽然多占用了一些空间，不过可以方便的定位到所需元素。
### set
mondis的set采用的是value为空的hash表进行实现，听起来似乎与redis没什么不同，但是mondis的hashmap采用avl树解决哈希冲突，减小了哈希表操作的常数因子。
### zset
redis的zset采用跳表+哈希表实现，编码异常复杂，空间占用也没很大优势。
mondis的zset采用伸展树实现，在时间与空间复杂度上进行了很好的权衡，而且伸展树方便的支持区间操作。
### hash
redis的hash在键值对数量不多时采用ziplist，虽然空间
占用小，但是查询时必须线性扫描，而且插入时有可能连锁更新影响性能。
在键值对数量较多时采用哈希表，浪费大量空间。
mondis的hash采用平衡树实现，在时间与空间复杂度上进行了很好的权衡。
# 开始使用Mondis
使用Mondis非常简单，首先需要取得Mondis的可执行文件，然后在终端输入./mondis xxx.conf。这时候可以
看到终端上打印出启动信息，表示Mondis启动成功。xxx.conf即配置文件路径，如果不指定配置文件，Mondis将
以默认配置启动。
Mondis启动后并不能立即使用。实际上，你需要登录才能执行命令。登录的命令是login username password。
username与password可以在配置文件中指定，如果没有配置文件，默认的username为root，password为admin。
其他命令将在下面介绍。
# mondis的数据结构
mondis支持redis所支持的全部数据结构，除此之外，mondis还支持二进制数据。也就是说Mondis支持string,list,set,zset,hash,binary
六种数据结构。下面一一进行介绍。
## string
顾名思义，string就是字符串，它跟redis的string，以及其他编程语言中的字符串并没有任何不同。但是string有两种编码方式,RAW_STRING和RAW_INT.
RAW_STRING就是字符串编码，如果string可以被转化为一个int的话，那么它将采用RAW_INT编码，这意味着在内存中它将是一个int。序列化结果是一个字符串。
## binary
即二进制数据，表现为内存中一块连续的char数组，长度不可变。支持随机访问和读写操作。序列化结果是一个字符串。
## list
即列表，相当于java里面的LinkedList，但是支持随机访问，时间复杂度为常数。除此之外还支持双端队列操作，即push_front,
push_bakc,pop_front和pop_back。list里面的元素可以是任意类型。序列化结果是一个json数组，内部元素按照list内部顺序排列。
但是第一个元素是一个"LIST"字符串，表明这是一个LIST序列化的结果，用于反序列化时参考。
## set
即集合，集合中不包含重复的元素，而且集合中的元素是无序的。集合支持add,remove,size和exists操作，时间复杂度均为常数。
与list不同，集合中的元素只能是string或者binary。序列化结果是一个json数组，内部元素顺序未定义。但是第一个元素是一个
"SET"字符串，表明这是一个SET序列化的结果，用于反序列化时参考。
## zset
即有序集合，在存取元素时需要指定一个int的score，内部元素根据score从小到大有序。但是score不能指定int类型的极值。
zset里面的元素可以是任意类型，支持set的所有操作，同时支持根据rank的区间访问，区间删除和根据score的区间访问，区间删除，
还有更改一个元素的score。序列化结果是一个json数组，元素是内部元素，顺序按score从大到小排列。但是第一个元素是一个"ZSET"字符串，表明这是一个ZSET序列化的
结果，用于反序列化时参考。zset的序列化分为带score与不带score的两种，带score的序列化为把四个字节的score作为一个元素以string形式添加到对应元素
序列化结果的前面，不带score的序列化不包括score。默认序列化方式为不带score，当执行get命令获取zset时采用的就是这种。
但是在json持久化时必须采用带score的版本。
## hash
即键值对的集合，key只能是RAW_STRING编码的string，值可以是任意类型。支持add,remove,size和exists操作。
序列化结果是一个json对象，键值对即hash的键值对。
# Mondis命令
## 概述
mondis有很多命令，分为键空间命令，locate命令与数据对象命令。键空间命令即直接在数据库空间执行的命令，
locate命令即定位命令，用来定位要操作的数据对象。数据对象命令即操作对应数据对象的命令。mondis命令有若干参数，
参数分为普通参数与string参数，普通参数就是一段连续的token，中间不能有空格，两端没有分号，如abc,1235等。
字符串参数就是一对引号括起来的参数，中间可以有任意字符。Mondis命令具有多态性，也就是说在操作数据对象时并不需要
关心这个数据对象的类型，只要它可以执行这条命令，命令就会被正确执行。否则你将会得到一个错误。mondis命令不区分大小写，
这意味着LOGIN，login与LoGIn均能被正确识别。下文以&lt;&gt;指定普通参数，以[]指string参数。注意，string参数
在被读取时是忽略两端的引号的，这意味着字符串参数"abc"将会被读取为abc。
## 键空间命令
### set &lt;key&gt; [content]
set命令即添加一条键值对。第一个参数是键，第二个是值。如果键值对已存在，则值将会被覆盖。第二个参数应当是标准的
json字符串，它将会被自动解析成合适的数据结构。例如，""this is a test""将会被解析成一个RAW_STRING，内容是
this is a test。注意这里出现了两对引号，这样做的原因是外面的一对引号表明这是一个string参数，内部的一对引号是
json字符串表示法的引号。""12345""将会被解析成RAW_INT编码的12345。如果想要使用RAW_BIN编码，可以在
二进制字符串的最开始加上LatentDragon这十二个字符。例如，""LatentDragonxxx""将会被解析成RAW_BIN编码的
xxxx,开头的LatentDragon将被忽略。注意所有以LatentDragon开头的字符串都将被解析成RAW_BIN，也就是说我们无法
保存普通的以LatentDragon开头的RAW_STRING。这个缺陷无伤大雅，而且会在后续版本中修复。"{}"将会被解析成没有键值对的hash。
"[]"将会被解析成没有元素的list,"["LIST"]"同样是list,"["SET"]"则是空的set,"["ZSET"]"则是空的zset。
"[{}]"则会被解析成有一个hash元素的list。
### del &lt;key&gt;
删除对应的key。
### get &lt;key&gt;
返回对应key的json表示
### exists &lt;key&gt;
检查是否存在对应的key。
### rename &lt;oldkey&gt; &lt;newkey&gt;
修改oldkey为newkey。
### type &lt;key&gt;
返回key对应的value底层编码类型。类型有RAW_STRING,RAW_INT,RAW_BIN,LIST,SET,ZSET,HASH
### move &lt;new&gt;
将一对kv迁移到编号为new的键空间中。
### select &lt;db&gt;
修改当前键空间为编号为db的键空间。
### save &lt;filepath&gt;
将整个当前键空间以json持久化到file里面。save命令的默认实现是开启子进程持久化，因此不需要担心性能问题。
mondis不提供bgsave。
### login &lt;username&gt; &lt;password&gt;
以username和password登录
### exit
退出登录并退出Mondis server。
### size
返回键空间内键的数量

## string命令
string命令分为两大类，有些只能在RAW_INT上执行，有些可以在RAW_STRING上执行。下面分别介绍
##RAW_STRING命令
### get &lt;position&gt; 
返回position处的字符。
### set &lt;position&gt; [char]
设置position处的字符为char。char的长度必须为1。
### get_range &lt;start&gt; &lt;end&gt;
获得从start到end的子串，包括start处的字符不包括end。
###set_range &lt;start&gt; [new]
将从start开始长度等于new的子串设为new。从new的第一个字符开始设置,如果new的长度大于从start开始到
结尾的子串，new多余的部分会被截断。
### set_range &lt;start&gt; &lt;end&gt; [new]
将从start到end的子串设为new。从new的第一个字符开始设置，多余的部分会被截断。
### strlen
返回该字符串的长度
### append [new]
将new添加到原来字符串的末尾。
##RAW_INT命令
### incr
值加一。
### decr
值减一。
### incr_by &lt;increment&gt;
值加上增量。
### decr_by &lt;decrement&gt;
值减去增量。
### to_string
将底层编码变成RAW_STRING

## binary命令
## get &lt;position&gt; 
返回position处的字符。
### set &lt;position&gt; [char]
设置position处的字符为char。char的长度必须为1。
### get_range &lt;start&gt; &lt;end&gt;
获得从start到end的子数组的字符串表示，包括start处的字符不包括end。
###set_range &lt;start&gt; [new]
将从start开始长度等于new的子数组设为new。从new的第一个字符开始设置,如果new的长度大于从start开始到
结尾的子串，new多余的部分会被截断。
### set_range &lt;start&gt; &lt;end&gt; [new]
将从start到end的子数组设为new。从new的第一个字符开始设置，多余的部分会被截断。
### set_position &lt;pos&gt;
将当前读写指针设为pos。
### forward &lt;increment&gt;
读写指针前进increment字节。
### backward &lt;decrement&gt;
读写指针后退decrement
###read系列命令
read系列命令在剩余长度小于想要读取的长度时，会读到末尾，直接返回。
### read_char
从当前读写指针读一个字节
### read_short
从当前读写指针读两个字节
### read_int
从当前读写指针读四个字节
### read_long
从当前读写指针读八个字节
### read_long_long
从当前读写指针读十六个字节
### read &lt;length&gt;
从当前读写指针读length个字节
### write &lt;length&gt; [data]
从当前读写指针开始从data里面读取length个字节长度的数据到binary里面。data的长度必须大于等于length，
如果剩余可读区间小于length，则length多余的部分会被截断。
### check_pos
返回当前读写指针的位置。

## list命令
### set &lt;index&gt; [data]
将list的第index个元素设为data。
### get &lt;index&gt;
获取list的第index个元素
### size
返回list中元素数量
### push_front [data]
将data添加到list开头
### push_back [data]
将data添加到list末尾
### pop_front
从list开头弹出并返回这个元素。如果list为空，则返回一个错误。
### pop_back 
从list末尾弹出并返回这个元素。如果list为空，则返回一个错误。
### get_range &lt;from&gt; &lt;to&gt;
获取并返回list下标从from到to之间的所有元素。包括from处的元素，不包括to。返回的格式是json数组。

## set命令
### add [data]
将data添加到list里面。data的反序列化结果的编码只能是RAW_INT或者RAW_STRING，否则会出错。下同。
### remove [data]
从set里面删除data。
### size
返回set中元素数量
### exists [data]
检查set中是否存在data

## zset命令
### get_by_score &lt;score&gt;
获得zset里面分数等于score的元素
### get_by_rank &lt;rank&gt;
获得zset里面排名等于rank的元素
### remove_by_score &lt;score&gt;
删去zset里面分数为score的元素
### remove_by_rank &lt;rank&gt;
删去排名为rank的元素。
### get_range_by_score &lt;from&gt; &lt;to&gt;
获得分数在from与to之间的所有元素。包括from，不包括to
### get_range_by_rank &lt;from&gt; &lt;to&gt;
获得排名在from与to之间的所有元素。包括from，不包括to
### remove_range_by_score &lt;from&gt; &lt;to&gt;
删除分数在from与to之间的所有元素。包括from，不包括to
### remove_range_by_rank &lt;from&gt; &lt;to&gt;
删除排名在from与to之间的所有元素。包括from，不包括to
### set &lt;score&gt; [data]
添加一个分数为score的元素进zset。如果该score已存在，则覆盖。
### exists &lt;score&gt;
检查是否存在分数为score的元素。
### size
返回zset中元素数量
### change &lt;old&gt; &lt;new&gt;
将分数为old的元素的分数变为new。如果分数为old的元素不存在，或者分数为new的元素已存在，则出错。

## hash命令
### set &lt;key&gt; [value]
添加一个键值对。如果key已存在，则覆盖。
### remove &lt;key&gt;
删除一个键值对。
### get &lt;key&gt;
返回key所对应的value。
### exists &lt;key&gt;
检查是否存在key。
### size
返回键值对的数量。

## locate命令
locate命令用于定位要操作的数据对象。下面是一个简单的示例。
执行set a "{"test"}"以后，a就是一个有一个元素的list，这个元素是"test"。那么我们要操作test，就要定位到
这个元素。首先在键空间中定位a，然后在a中定位"test"。假如我们要获得test的第一个字符，那么应该执行如下命令：
locate a|locate 1|get 1。locate a在键空间中定位到a，locate 1定位到a的第一个元素，然后获得test的第一个字符。
竖线的作用是分隔locate命令。相邻的locate命令之间必须有竖线。下面详细介绍各个数据结构的locate。
### string,binary，set
string，binary与set无法locate，如果尝试执行，将会出现一个错误。
### list
list的locate命令格式是locate &lt;index&gt;。
### zset
zset的locate命令格式是locate "RANK" &lt;rank&gt;或者locate "SCORE" &lt;score&gt;
### hash
hash的locate命令格式是locate &lt;key&gt;。
### 键空间
与hash相同。

