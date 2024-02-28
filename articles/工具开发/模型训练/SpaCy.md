> 在开头记录一下上一章的问题
>
> 单条语句比较清楚，如果数据过大，matcher肯定不止一个，这时候怎么处理matcher

## 使用spaCy进行大规模数据分析

+ 数据结构

  Vocab, Lexemes和StringStore

  spaCy把所有共享数据都存在一个词汇表里，也就是Vocab。

  - `Vocab`: 存储那些多个文档共享的数据
  - 为了节省内存使用，spaCy将所有字符串编码为**哈希值**。
  - 字符串只在`StringStore`中通过`nlp.vocab.strings`存储一次。
  - 字符串库：双向的**查询表**

  Lexeme（语素）是词汇表中和语境无关的元素。

  - 一个`Lexeme`实例是词汇表中的一个元素
  - 包含了一个词的和语境无关的信息
    - 词组的文本：`lexeme.text`和`lexeme.orth`（哈希值）
    - 词汇的属性如`lexeme.is_alpha`
    - **并不包含**和语境相关的词性标注、依存关系和实体标签

Doc、Span和Token

```python
# 导入Doc类
from spacy.tokens import Doc

# 用来创建doc的词汇和空格
words = ["Hello", "world", "!"]
spaces = [True, False, False]

# 手动创建一个doc
doc = Doc(nlp.vocab, words=words, spaces=spaces)
```

+ 词向量

  对比语义相似度

  ```
  spaCy可以对比两个实例来判断它们之间的相似度
  Doc.similarity()、Span.similarity()和Token.similarity()
  使用另一个实例作为参数返回一个相似度分数(在0和1之间)
  注意：我们需要一个含有词向量的流程，比如：
  ✅ en_core_web_md (中等)
  ✅ en_core_web_lg (大)
  🚫 而不是 en_core_web_sm (小)
  ```

  ```python
  # 读取一个有词向量的较大流程
  nlp = spacy.load("en_core_web_md")
  
  # 比较两个文档
  doc1 = nlp("I like fast food")
  doc2 = nlp("I like pizza")
  print(doc1.similarity(doc2))
  0.8627204117787385
  ```

## 处理流程

通过组件来进行实现，分词器会返回doc实例，然后按照流程运行每一个组件

+ 定制化流程组件

  定制化流程组件让我们可以在spaCy的流程中加入我们自己的函数，当我们在一段文本上调 用`nlp`时，这些函数就会被调用来完成比如修改doc为其增加更多数据的任务。

  ```python
  from spacy.language import Language
  
  @Language.component("custom_component")
  def custom_component_function(doc):
      # 对doc做一些处理
      return doc
  
  nlp.add_pipe("custom_component")
  ```

  ![image-20240201183329057](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240201183329057.png)

+ 扩展属性

  - 添加定制化元数据到文档document、词符token和跨度span中

  - 通过`._`属性来读取

  - 使用`set_extension`方法在全局的`Doc`、`Token`或`Span`上注册。

  - ```python
    doc._.title = "My document"
    token._.is_color = True
    span._.has_color = False
    
    # 导入全局类
    from spacy.tokens import Doc, Token, Span
    
    # 在Doc、Token和Span上设置扩展属性
    Doc.set_extension("title", default=None)
    Token.set_extension("is_color", default=False)
    Span.set_extension("has_color", default=False)
    ```

    

## 模型训练和更新

唯一真理来源config.cfg

- 定义了如何初始化`nlp`对象
- 包含了关于流程组件和模型实现的所有设定
- 配置了训练过程和超参数
- 使我们的训练过程可复现
