# Apache Lucene
> Lucene是一个高性能、可伸缩的信息搜索(IR)库。Information Retrieval(IR) library.它使你可以为你的应用程序添加索引和搜索能力。但Lucene也仅提供搜索能力，需要根据实际使用自行完成搜索程序的其他模块（例如网页抓取、文档处理、服务器运行、用户界面和管理等）。

 人们初次解除到Lucene时，很容易将它和一些即用型程序搞混淆，比如文件搜索程序、网页搜索器以及网站搜索引擎等。其实这不是Lucene的真面目：Lucene只是一个软件类库，或者一个工具箱，而并不是一个完整的搜索程序。

 ## Lucene核心
 ### 索引过程的核心类
 ####  IndexWriter
 提供针对索引文件的写入操作，但不能用于读取或搜索索引，IndexWriter需要开率一定空间来存储索引，该功能可以由Directory完成。
 #### Directory
描述了Lucene索引的存放位置，它是一个抽象类，它的子类负责具体制定索引的存储路径。
 #### Analyzer
 文本文件在被索引之前，需要经过Analyzer（分词器）处理，它负责从被索引文本文件中提取语汇单元，并提出剩下的无用信息，如果被索引内容不是纯文本文件，那就需要先将其传唤为文本文档。
#### Document
代表一些域（Field）的集合，文档的域代表文档或者和文档相关的一些元数据。Document对象结构为一个包含多个Field对象的容器，Field是指包含能被索引的文本内容的类。
#### Field
索引中的每个文档都包含一个或多个不同命名的域，这些域包含在Field类中。

### 搜索过程的核心类
#### IndexSearcher
用于搜索由IndexWriter类创建的索引，可将其看所一个以只读方式打开索引的类，需要利用Directory实例来获取创建的索引。
#### Term
搜索功能的基本单元，与Field对象类似，Term对象包含一堆字符串元素：域名和单词，由于Term对象是由Lucene内部创建的，我们并不需要在索引阶段详细了解它们。
```java
// 搜索contents域中包含单词lucene的前10个文档，并按降序排列
Query q = new TermQuery(new Term("contents", "lucene"));
TopDocs hits = searcher.search(q, 10);
```
#### Query
Lucene含有许多具体的Query子类，如TermQuery、BooleanQuery、PhraseQuery等。
#### TermQuery
最基本的查询类型，用来指定域中包含特定项的文档。
#### TopDocs
简单的指针容器，一般指向前N个排名的搜索结果。

