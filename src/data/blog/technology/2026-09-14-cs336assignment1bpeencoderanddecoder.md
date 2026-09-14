---
title: cs336_assignment1_bpe_encoder_and_decoder
author: 迷途的汪汪
description: ""
pubDatetime: 2026-09-14T06:23:42.448Z
modDatetime: 2026-09-14T06:23:42.449Z
preview: ""
draft: false
featured: false
slug: "cs336_assignment1_bpe_encoder_and_decoder"
tags: ["cs336", "bpe"]
categories: ["cs336"]
---

# Class 2 BPE Tokenizer: Encoder \& Decoder

参考链接：

https://ichirinko\.top/2026/07/02/cs336%20assignment%201%20basic%20\(2\)/

第一节课，训练了一个输入文本，输出为token词表和BPE合并项列表的BPE Tokenizer。本节我们为它增添更多的功能，使其能够接收上面的词表与合并项列表，并使用它们实现文本和token ID的编解码。

### Encoder

对文本编码的过程其实和我们之前训练BPE分词器的过程是类似的。主要分为以下两个步骤：

1. 预分词

首先对文本序列进行预分词，并用UTF\-8编码表示分词序列。接下来，我们将各个分词的“token碎片”合并为词表中有的单词。

2. 应用合并

词表中的每个元素都有一个编号，这使得我们可以仅保存编号而不需要保存实际的文本。当我们对输入文本进行预分词后，需要去词表中找到对应的单词编号，从而实现文本的编号化。

### Decoder

解码的过程其实就是编码的逆过程。可以通过将编号对应的词表单词重新组合，并转换这些字节为Unicode字符来实现。

### 代码实现

1. 新建BPE_Tokenizer类

```Python
from typing import Iterable, Iterator

class BPE_Tokenizer:
    def __init__(self,vocab: dict[int, bytes],
                 merges: list[tuple[bytes, bytes]],
                 special_tokens: list[str] | None = None):
        """
        Construct a tokenizer from a given vocabulary, list of merges, and (optionally) a list of special tokens.
        Args:
            vocab: dict[int, bytes]
            merges: list[tuple[bytes, bytes]]
            special_tokens: list[str] | None = None
        """
        pass

    @classmethod
    def from_files(cls, vocab_filepath: str, merges_filepath: str, special_tokens=None):
        """
        Class  method that constructs and return a Tokenizer from a serialized vocabulary and list of merges (in the same format that your BPE training code output) and (optionally) a list of special tokens.
        Args:
            vocab_filepath: str
            merges_filepath: str
            special_tokens: list[str] | None = None
        """
        pass

    @staticmethod
    def encode(self, text: str) -> list[int]:
        """
        Encode an input text into a sequence of token IDs.
        """
        pass

    def encode_iterable(self, iterable: Iterable[str]) -> Iterator[int]:
        """
        Given an iterable of strings (e.g., a Python file handle), return a generator that lazily yields token IDs. This is required for memory-efficient tokenization of large files that we cannot directly load into memory.
        """
        pass

    def decode(self, ids: list[int]) -> str:
        """
        Decode a sequence of token IDs into text.
        """

```

2. 实现`from_files`方法

```Python
@classmethod
    def from_files(cls, vocab_filepath: str, merges_filepath: str, special_tokens: list[str] | None = None):
        """
        类方法：从序列化保存的词表、BPE合并规则文件，构建并返回一个Tokenizer分词器。
        文件格式必须和BPE训练代码输出的格式保持一致，可以额外传入特殊token列表。
        参数:
            vocab_filepath: str          # 词表pickle文件路径
            merges_filepath: str         # BPE合并规则pickle文件路径
            special_tokens: list[str] | None = None # 特殊token列表，例如["<|endoftext|>"]，可选
        """

        # 以二进制只读模式打开词表文件、BPE合并规则文件；with会自动关闭文件
        with open(vocab_filepath, 'rb') as vf, open(merges_filepath, 'rb') as mf:
            # 加载pickle文件: vocab字典, key是token id, value是token对应的字符串bytes
            vocab: dict[int, bytes] = pickle.load(vf)
            # 读取BPE合并规则: 列表，每个元素是一对(bytes, bytes)的元组，表示BPE合并的子词对，顺序很关键
            merges: list[tuple[bytes, bytes]] = pickle.load(mf)

            # 获取原始词表长度，新增特殊token的ID就从这个序号开始分配
            size = len(vocab)
            # 遍历所有特殊token，special_tokens为None时，special_tokens or []会返回空列表，避免报错
            for token in special_tokens or []:
                # 如果特殊token不在原始词表中，则将其加入词表，并分配新的ID
                if token.encode('utf-8') not in vocab.values():
                    vocab[size] = token.encode('utf-8')
                    # ID计数器 + 1，供下一个特殊token使用
                    size += 1
            # 传入处理好的词表、合并规则、原始特殊token列表，实例化分词器对象并返回
            return cls(vocab, merges, special_tokens)
```

3. 初始化`__init__`

先来考虑一下编码的全过程：

- 假设输入的文本是：`the cat ate``

- 当前的词表\(`vocab`\)是`{0: b' ', 1: b'a', 2: b'c', 3: b'e', 4: b'h', 5: b't', 6: b'th', 7: b' c', 8: b' a', 9: b'the', 10: b' at'}`

- 当前的合并规则merges（**顺序至关重要，按列表先后优先级合并**）： `[(b't', b'h'), (b' ', b'c'), (b' ', b'a'), (b'th', b'e'), (b' a', b't')]`

> merges 含义：优先合并排在前面的字节对；每一轮，**在当前片段内找 merges 列表里最先出现的一对进行合并**，循环直到没有可合并对。

- 预分词结果：`["the", " cat", " ate"]`

- ✅ 规则：**三个预分词片段独立合并，跨片段不能合并**

Encode过程:

片段 1：`"the"`

1. 把字符串转 UTF\-8 最小字节单元：`"the"` → `[b't', b'h', b'e']`

2. 进入循环，扫描 merges 顺序，查找当前片段里存在的字节对
   - merges 第 1 组：`(b't', b'h')`。当前序列`[b't', b'h', b'e]`存在相邻对`b't`\+`b'h'`

   - 执行合并：`b't' + b'h' → b'th`

   - 序列更新：`[b'th', b'e']`

3. 再次从头遍历 merges 列表，寻找当前序列`[b'th', b'e']`中可合并相邻对
   - 前 3 组`(t,h) / ( ,c) / ( ,a)` 在`[b'th', b'e']`里不存在

   - 第 4 组：`(b'th', b'e')`，正好匹配相邻两个 token

   - 执行合并：`b'th' + b'e' → b'the'`

   - 序列更新：`[b'the']`

4. 再次扫描 merges：序列只有单个元素，不存在相邻字节对，**本片段合并结束**

> 片段 1 合并结果：`[b'the']`

片段 2: " cat"\(开头带空格！预分词输出，字符串是`" cat"`\)

1. 拆成最小字节单元：`" cat"` → `[b' ', b'c', b'a', b't']`

2. 再次扫描 merges：序列只有单个元素，不存在相邻字节对，**本片段合并结束**
   - 片段 1 合并结果：`[b'the']`

   - 从头扫描 merges：
     - 第 1 组`(t,h)`：无；

     - 第 2 组`(b' ', b'c')`，当前序列开头正好是`b' '`后面跟着`b'c'`，匹配！

     - 合并 `b' ' + b'c' → b' c'`

     - 序列更新：`[b' c', b'a', b't']`

3. 再次从头扫描 merges:
   - `(t,h)`无；`( ,c)`：现在序列第一个元素是`b' c'`，不再有独立`b' ' + b'c`；

   - `( ,a)`：当前相邻对是`b'c'`与`b'a'`，不是`b' '`\+`b'a'`，不匹配；

   - `(th,e)`、`( a,t)`也不匹配。

- 没有可合并的字节对，循环终止

片段 2 合并结果：`[b' c', b'a', b't']`

片段3: `" ate"`（开头带空格，预分词结果）

1. 拆最小字节单元：`" ate"` → `[b' ', b'a', b't', b'e']`

2. 从头遍历 merges：
   - 第 1 组`(t,h)`无；第 2 组`( ,c)`无；

   - 第 3 组`(b' ', b'a')`匹配！相邻`b' '` \+ `b'a'`

   - 合并得到`b' a'`，序列更新：`[b' a', b't', b'e']`

3. 再次从头扫描 merges：
   - 前面几组不匹配；来到第 5 组 `(b' a', b't')`，正好匹配相邻两项 `b' a'` \+ `b't'`

   - 合并得到`b' at'`，序列更新：`[b' at', b'e']`

4. 再次扫描全部 merges，`[b' at', b'e']`里面没有任何 merges 里定义的相邻字节对。停止合并

> 片段 3 合并结果：`[b' at', b'e']`

# 合并全部片段的字节序列

拼接三个片段结果： `[b'the'] + [b' c', b'a', b't'] + [b' at', b'e']`， 最终合并后的子词 bytes 列表： `[b'the', b' c', b'a', b't', b' at', b'e']`

# 映射成 token id（查表 vocab）

- `b'the'` → 9

- `b' c'` →7

- `b'a'` →1

- `b't'` →5

- `b' at'` →10

- `b'e'` →3

最终编码输出 id 序列：`[9,7,1,5,10,3]`

```Python
def __init__(self,vocab: dict[int, bytes],
                 merges: list[tuple[bytes, bytes]],
                 special_tokens: list[str] | None = None):
        """
        根据给定词表、BPE合并对列表，以及可选的特殊token列表，构造分词器。
        1. 按merges里的顺序查找合并对，优先匹配merges列表靠前的合并对。
        2. special_token（特殊标记）和普通文本要分开处理。
        3. 匹配special_token时，优先匹配更长的special_token（你注释写反了！！重点提醒）。
        Args:
            vocab: dict[int, bytes]        # 主词表 key:token_id，value:对应的bytes子词
            merges: list[tuple[bytes, bytes]] # BPE合并规则列表，顺序代表合并优先级，越靠前优先级越高
            special_tokens: list[str] | None = None # 特殊token列表，如["<|endoftext|>"]，可选
        """

        # 保存词表、合并规则到实例属性
        self.vocab = vocab
        self.merges = merges
        # 如果special_tokens是None，转为空列表，避免后续遍历报错
        self.special_tokens = special_tokens or []

        # 对special_tokens进行排序，确保较长的special_token优先匹配
        # 例：如果同时有 `<abc>` 和 `<ab>`，不能先匹配短的`<ab>`，否则`<abc>`被拆错。
        # key=lambda x: len(x) → 按长度从小到大排序；reverse=True → 从长到短排序
        self.special_tokens = sorted(self.special_tokens, key=lambda x: len(x), reverse=True)  # Sort special tokens by length and lexicographically

        # 构建词表倒排索引：bytes子词 → token id
        # 原vocab是 id → bytes；这个字典用来快速根据bytes子词查找对应的编号
        self.bytes_to_id = {v: k for k, v in vocab.items()}

        # 构建merge的pairs合并优先级倒排索引，使查找效率达到O(1)
        # 构建合并对优先级映射字典 pair:rank
        # rank = 在merges列表中的下标，数字越小优先级越高
        # 作用：后续编码BPE循环时，拿到一对bytes，可以O(1)快速查它的合并优先级
        self.merge_ranks = {pair: rank for rank, pair in enumerate(merges)}
```

5. **`encode_iterable()`**

为了防止内存爆炸，这个函数实现了流式编码的效果，可以在还没有实现`encode()`的情况下先实现这个函数

```Python
def encode_iterable(self, iterable: Iterable[str]) -> Iterator[int]:
        """
        接收一个字符串可迭代对象（例如Python文件句柄），返回生成器，惰性逐个产出token id。
        这个函数用于处理超大文件的分词，不需要一次性把整个文件加载进内存，节省内存开销。
        """
        # 遍历可迭代对象里的每一段字符串（chunk，比如文件按行读取的一行文本）
        for chunk in iterable:
            # 调用当前分词器的encode函数，把这一段文本转为token id序列
            # yield from：逐个产出encode返回的所有token id，交给外层调用方
            for token_id in self.encode(chunk):
                yield token_id
```

6. `encode`

此处的算法逻辑:

1. 优先处理**特殊token分离**: `split_by_special`把文本切开，例如:`hello<eot>world`\-\>`["hello", "<eot>", "world"]`,特殊token不会被普通BPE拆开。

2. 特殊token：直接转字节查表拿id。

3. 普通文本块：
   1. 正则粗切分（BPE 标准的预切分，处理空格、标点、字母数字

   2. 转为 utf\-8 单字节列表

   3. `get_best_merge`：BPE 合并规则，从单字节不断两两合并，得到子词字节单元

   4. 每个合并后的子词字节查表映射成整数 token id

```Python
def encode(self, text: str) -> list[int]:
        """
        将输入文本编码为token ID序列
        整体逻辑：先把文本按特殊token切分，特殊token单独处理；普通文本走BPE字节合并分词，映射为token id
        :param text: 原始输入字符串
        :return: 编码后的token ID整数列表
        """
        # 如果输入文本为空字符串，直接返回空列表
        if not text:
            return []

        # 使用正则表达式模式匹配文本中的token, 用于把普通文本切分为基础utf-8字节单元(BPE常用的分词正则PAT)
        pattern = re.compile(PAT)

        token_ids = []

        # 按特殊token对原文做分块切割，保留特殊token不丢弃(drop_special=False)
        # blocks是切分后的文本块列表，块要么是普通文本，要么是单个特殊token
        blocks = self.split_by_special(text, self.special_tokens, drop_special=False)
        # 遍历每个被切分出来的文本块
        for block in blocks:
            # 判断: 当前块是预定义的特殊token(如<|endoftext|>)，则直接查表映射为token id
            if self.special_tokens and block in self.special_tokens:
                block_bytes = bytes(block.encode('utf-8'))
                token_ids.append(self.bytes_to_id[block_bytes])
            else:
                # 对于普通文本块，进行BPE分词, 先用正则做基础分词，得到候选字符串单元。
                tokens: list[str] = []
                # 使用正则表达式查找所有匹配的token
                for match in re.finditer(pattern, block):
                    tokens.append(match.group(0))
                # 遍历正则切分出的每一个基础单元
                for token in tokens:
                    # 把字符串编码成utf-8,拆成单个字节，每个字节包装成bytes对象，形成一个字节列表
                    token_bytes = [bytes([b]) for b in list(token.encode('utf-8'))]
                    # BPE核心: 递归/迭代执行最优合并，将单字节列表合并成训练好的BPE子词单元
                    bytes_list = self.get_best_merge(token_bytes)
                    # 遍历合并完成后的字节子词，查表转token_id,追加到输出序列
                    for b in bytes_list:
                        # 查表获取对应的token id，并追加到token_ids列表
                        token_ids.append(self.bytes_to_id[b])
        return token_ids
```

7. `split_by_special`

```Python
def split_by_special(self, text, special_tokens, drop_special = True):
        """
        将文本按特殊token分割为若干块，返回分割后的块列表。
        原理：用正则一次性匹配所有特殊token，对原文切割；
        drop_special=True：切割后丢掉特殊token本身，只保留普通文本；
        drop_special=False：切割结果里**保留特殊token**，文本和特殊token交替出现在列表中。

        :param text: 待分割原始字符串
        :param special_tokens: 特殊token列表，例如 ["<|endoftext|>", "<|im_start|>"]
        :param drop_special: 是否丢弃特殊token。True=丢掉，False=保留特殊token作为独立块
        :return: 分割后的文本块列表，自动过滤空字符串
        """
        # 如果没有定义任何特殊token，直接返回原文包裹在列表中，无需分割
        if not special_tokens:
            return [text]

        # 1. 对每个特殊token做正则转义(防止token里面含有. * + ? 这类正则元字符)
        # 用|拼接，表示正则“或”，匹配任意一个特殊token。
        pattern = '|'.join(re.escape(token) for token in special_tokens)

        # 2. 如果不丢弃特殊token，给整个表达式加括号 ()
        # re.split 在捕获组模式下，会把匹配到的内容也放进返回的chunks列表
        # 不加括号时，split只会返回分割后的文本，丢掉被匹配到的内容
        if not drop_special: pattern = f'({pattern})'

        # 预编译正则
        pattern = re.compile(pattern)
        # 使用正则分割文本
        chunks = pattern.split(text)

        # 列表推导：过滤掉分割产生的空字符串（例如文本开头/结尾匹配特殊token会产出空chunk）
        return [chunk for chunk in chunks if chunk]  # Remove empty strings
```

8. `get_best_merge`

假设：`self.merge_ranks` 字典存 BPE 合并优先级，key 是`(b'h',b'e')`、`(b'l',b'l')`这种 pair，value=rank，**rank 越小越优先合并**

```Python
merge_ranks = {
    (b'h',b'e'): 1,
    (b'l',b'l'): 2
}
```

输入： `token_bytes = [b'h', b'e', b'l', b'l', b'o']`

**第一轮 while**

遍历相邻对:

- i=0: pair=\(b'h',b'e'\), rank=1

- i=1: pair=\(b'e',b'l'\), rank=None（不存在合并规则）

- i=2: pair=\(b'l',b'l'\), rank=2

- i=3: pair=\(b'l',b'o'\), rank=None

最优是 i=0，pair`(b'h',b'e')`，rank=1 合并：`b'h'+b'e' = b'he'` 新列表：`[b'he', b'l', b'l', b'o']`

**第二轮 while**

现在列表：`[b'he', b'l', b'l', b'o']`

相邻对：

i=0: \(b'he',b'l'\) → 无 rank

i=1: \(b'l',b'l'\) → rank=2

i=2: \(b'l',b'o'\) → 无 rank

最优 i=1，合并`b'l'+b'l'=b'll'` 新列表：`[b'he', b'll', b'o']`

### 第三轮 while

列表：`[b'he', b'll', b'o']`

相邻对： i=0: \(b'he',b'll'\) → 无 rank i=1: \(b'll',b'o'\) → 无 rank

全部 pair 查不到 rank → best_pair_id=\-1 → break 循环 返回结果：

`[b'he', b'll', b'o']`

```Python
def get_best_merge(self, token_bytes: list[bytes]) -> list[bytes]:
        """
        BPE 合并主逻辑：不断查找当前列表里**优先级最高（rank最小）**的可合并字节对，循环合并直到不能再合并
        BPE训练时会给每一对相邻子词分配一个rank，rank越小代表合并优先级越高
            :param token_bytes: 待合并的字节列表，例如 [b'h', b'e', b'l', b'l', b'o']
            :return: 完成BPE合并后的字节子词列表，例如 [b'he', b'll', b'o']
        """
        # 只要列表里还有>=2个单元，就可以尝试继续两两合并；只剩1个就停止
        while len(token_bytes) >= 2:
            # 记录当前找到最优合并对的rank，初始化为无穷大，表示还没找到
            best_rank = float('inf')
            # 记录最优合并对的起始下标；-1代表本轮找不到可合并对
            best_pair_id = -1
            # 遍历所有相邻字节对，查找本轮优先级最高的可合并pair
            for i in range(len(token_bytes) - 1):
                # 取出第i个和i+1个，组成相邻pair
                pair = (token_bytes[i], token_bytes[i + 1])
                # 去merge_ranks字典查询这一对是否在与训练好的合并规则中
                rank = self.merge_ranks.get(pair)
                # 如果这个pair存在合并规则，并且它的rank比当前最优更小(优先级更高)
                if rank is not None and rank < best_rank:
                    best_rank = rank
                    best_pair_id = i # 更新: 记录最优合并对的起始下标
            # 如果这次循环没有找到可以合并的，说明已经无法再合并下去
            if best_pair_id == -1:
                break
            # 合并best_pair_id对应的两个token
            best_merge = token_bytes[best_pair_id] + token_bytes[best_pair_id + 1]

            # 别忘了更新token_bytes
            # 前半部分: best_pair_id之前所有元素
            # 中间部分: 替换成刚合并好的单个子词
            # 后半部分: 跳过被合并的两个元素，取后面剩下的
            token_bytes = (
                token_bytes[:best_pair_id] +
                [best_merge] +
                token_bytes[best_pair_id + 2:]
            )

        # 无法继续合并，返回最终的子词字节列表
        return token_bytes
```

9. `decode`

```Python
def decode(self, ids: list[int]) -> str:
    """
    Decode a sequence of token IDs into text.
    将token ID序列还原为原始字符串（BPE分词器的解码函数）
    :param ids: 整数构成的token id列表，例如 [2001, 2002, 111, 999]
    :return: 拼接还原后的utf-8文本字符串
    """
    # 1. 遍历每一个token id t，从vocab字典查表：id → 对应的bytes子词
    # 2. b''.join(...)：把所有子词字节串拼接成一整个连续的大bytes
    # 3. .decode('utf-8', errors='replace')：把完整字节转为utf-8字符串
    #    errors='replace'：遇到无法解析的非法字节，用 � 替代，不会直接抛异常崩溃
    return b''.join([self.vocab[t] for t in ids]).decode('utf-8',errors='replace')
```

举例：

```Python
# self.vocab 是 vocab 字典：key=token_id，value=对应的bytes
self.vocab = {
    2001: b'he',
    2002: b'll',
    111: b'o',
    999: b'<|endoftext|>'
}
ids = [2001, 2002, 111, 999]
```

执行拆解：

1. 列表推导 `[self.vocab[t] for t in ids]` 得到：`[b'he', b'll', b'o', b'<|endoftext|>']`

2. `b''.join(...)` 拼接所有字节 → `b'hello<|endoftext|>'`

3. `.decode('utf-8', errors='replace')` → 转字符串：`"hello<|endoftext|>"`
