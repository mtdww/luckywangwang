---
title: cs336 assignment1 BPE Tokenizer
author: 迷途的汪汪
description: "This is a simple BPE tokenizer implementation for the cs336 assignment."
pubDatetime: 2026-09-10T02:19:32.107Z
modDatetime: 2026-09-10T02:19:32.107Z
preview: "This is a simple BPE tokenizer implementation for the cs336 assignment."
draft: false
featured: false
slug: "cs336-assignment1-bpe-tokenizer"
tags: ["BPE", "Tokenizer", "cs336"]
categories: ["cs336"]
---

## 参考链接

- [CS336 第一章② BPE Tokenizer的编码与解码](<https://ichirinko.top/2026/07/02/cs336%20assignment%201%20basic%20(2)/>)
- [斯坦福 CS336 作业1学习笔记（上）：大模型 Tokenizer 与 BPE 算法核心](https://zhuanlan.zhihu.com/p/1999603057755971636)
- [BPE Trainer (1) - BPE 算法实现](https://fancyerii.github.io/2025/09/07/bpe-trainer-1/)
- [CS336 第一章① 来做一个自己的大模型!](<https://ichirinko.top/2026/06/30/cs336%20assignment%201%20basic%20(1)/>)
- [BPE 算法原理及使用指南【深入浅出】](https://zhuanlan.zhihu.com/p/448147465)

## 主要完成BPE章节

> 这一节主要完成的是子词(Subword)分词

### SubWord概念介绍

把一个词切成更小的一块一块的子词。如果我们能使用将一个 token 分成多个 subtokens。

目前主流的Subword算法，它们分别是: Byte Pair Encoding(BPE)、WordPiece和Unigram Language Model.

### BPE(Byte Pair Encoding)算法介绍

字节对编码（BPE, Byte Pair Encoder），又称 digram coding 双字母组合编码，是一种数据压缩 算法，用来在固定大小的词表中实现可变⻓度的子词。该算法简单有效，因而目前它是最流行的方法。

#### 算法过程:

1. 准备语料库，确定期望的 subword 词表大小等参数
2. 通常在每个单词末尾添加后缀 </w>，统计每个单词出现的频率，例如，low 的频率为 5，那么我们将其改写为 "l o w </ w>”：5
   > 停止符 </w> 的意义在于标明 subword 是词后缀。举例来说：st 不加 </w> 可以出现在词首，如 st ar；加了 </w> 表明该子词位于词尾，如 we st</w>，二者意义截然不同
3. 将语料库中所有单词拆分为单个字符，用所有单个字符建立最初的词典，并统计每个字符的频率，本阶段的 subword 的粒度是字符
4. 挑出频次最高的符号对 ，比如说 t 和 h 组成的 th，将新字符加入词表，然后将语料中所有该字符对融合（merge），即所有 t 和 h 都变为 th。
5. 重复遍历 2 和 3 操作，直到词表中单词数达到设定量 或下一个最高频数为 1 ，如果已经打到设定量，其余的词汇直接丢弃

### 0. 建立BPE分词的训练类

```python
import os
import regex as re
import collections

CHUNK_SIZE = 1024 * 50  # 1MB
BYTES_NUM = 256  # 字节数
class BPE_Trainer:
    def __init__(self, input_path: str | os.PathLike, vocab_size: int, special_tokens: list[str] = ["<|endoftext|>"]) -> None:
        """
        Initializes the BPE_Trainer with the specified input path, vocabulary size, and special tokens.

        Args:
            input_path: The path to the input text file.
            vocab_size: The desired size of the vocabulary.
            special_tokens: A list of special tokens to include in the vocabulary.

        """
        self.input_path = input_path
        self.vocab_size = vocab_size
        self.special_tokens = special_tokens
```

### 1. 训练BPE分词器

#### 初始化词表

分词器词表是一个从字节化 token 到常数 ID 的一对一映射表。我们要训练的是一个字节级词表。因此我们初始化的词表就是一个简单的全ASCII字节集合，众所周知ASCII字符有256个（我们这里用的是扩展ascii字符表），我们可以初始化词表大小为256。

- 基础单元:0 ~ 255(一共256个字节)
- 词表大小:256
- 映射方向:
  - 词表vocab:token_id -> bytes
  - 反过来: bytes -> token_id(一对一反向映射)
- 举例

```python
# -------- 第一步: 初始化词表: 基础单元是全部256个字节(0-255)，再加上特殊token
# {i: bytes([i]) for i in range(BYTES_NUM)}
# BYTES_NUM一般 = 256，对应所有ASCII/utf8基础单字节
# 例子: {0: b'\x00', 1: b'\x01', 2: b'\x02', ..., 255: b'\xff'}
vocabulary = {i: bytes([i]) for i in range(BYTES_NUM)}
# 把特殊token加入词表，特殊token的id从BYTES_NUM开始
for i, sp_token in enumerate(self.special_tokens):
    vocabulary[BYTES_NUM + i] = sp_token.encode("utf8")

# 当前词表总大小 = 256 + len(special_tokens)
size = BYTES_NUM + len(self.special_tokens)
```

### 2.预分词

一旦我们已经拥有一个词汇表，从原则上讲，就可以统计文本中各个字节相邻出现的频率，并从最频繁的一对字节开始进行合并。

#### 代码实现

##### 1. 实现`pretokenize_and_count`方法

```python
def _pretokenize_and_count(self, input_path: str | os.PathLike, special_tokens: list[str] = ["<|endoftext|>"]) -> dict:
        """
        预分词并统计词频（GPT2风格预分词）
        作用：把原始文本先做**预切分(pretokenize)**，得到预分词单元，统计每个预分词单元出现次数
        BPE合并只在预分词单元内部做，**不会跨预分词单元合并**，这是GPT2 ByteBPE的核心规则。
        :param input_path: Path to the input file
        :param special_tokens: List of special tokens(e.g., ["<|endoftext|>"])
        :return: A list of bytes, each element representing a base symbol
        """
        # GPT2原版预分词正则表达式，用于把文本切分为预分词单元
        # 规则: 缩写(如 's, 't, 're, 've, 'm, 'll), 词语(Unicode字母), 数字(Unicode数字), 非空白符号(标点符号等), 空白符号
        pattern = re.compile(r"""'(?:[sdmt]|ll|ve|re)| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+""")

        # 构造正则表达式，用于匹配特殊token，拼接成 token1|token2|token3 的形式，方便后续分割文本块
        special_pattern = '|'.join(re.escape(token) for token in special_tokens)
        # 字典: key是预分词文本，value是该预分词单元出现的总次数
        word_counts = collections.defaultdict(int)

        # 流式读取文件，分块加载(防止一次读超大文件，撑爆内存)
        for chunk in BPE_Trainer._chunk_documents_streaming(input_path):
            # 先用特殊token分割文本块，再对每个块进行正则匹配
            # 保证每个块都是完整的文档，避免跨文档的预分词
            blocks = re.split(special_pattern, chunk)
            # 遍历切分后的每一块,不含特殊token
            for block in blocks:
                # 使用正则匹配基础符号，并统计每个符号的频率
                for match in re.finditer(pattern, block):
                    text = match.group(0)
                    word_counts[text] += 1

        return word_counts
```

- 这里需要重点理解的代码
```python
# 构造正则表达式，用于匹配特殊token，拼接成 token1|token2|token3 的形式，方便后续分割文本块
special_pattern = '|'.join(re.escape(token) for token in special_tokens)
```
_这块代码主要是我不理解，所以需要解释一下._
1. `re.escape()` 的作用：**把字符串里所有正则特殊字符转义**,`|`在正则中表示`或`的意思, `re.escape("<|endoftext|>")` → `\<\|endoftext\|\>`,防止特殊 token 里面本身含有正则符号，把正则表达式搞崩。
2. `'|'.join(...)` 这块代码是`|`在正则中表示OR，用来把多个转义后的token用`|`拼接，构成多选一的规则。
3. 举个例子，`"Today is good<|endoftext|>Hello world<|bos|>Test"`, `special_tokens = special_tokens = ["<|endoftext|>", "<|bos|>"]`, 我们可以直接切开为`['Today is good', 'Hello world', 'Test']`

- 第二个代码我需要理解的部分
```python
for block in blocks:
    # 使用正则匹配基础符号，并统计每个符号的频率
    for match in re.finditer(pattern, block):
        text = match.group(0)
        word_counts[text] += 1
```
1. 官方推荐使用`re.finditer`: **迭代器，找出当前 block 里所有匹配 pattern 的子串**，不会一次性生成全部列表，内存更友好。
2. 和`re.findall`区别：findall 直接返回字符串列表；finditer 返回匹配对象迭代器。

##### 2. 实现`_chunk_documents_streaming`方法
```python
@staticmethod
    def _chunk_documents_streaming(input_path: str | os.PathLike, chunk_size: int = CHUNK_SIZE, special_tokens: list[str] = ["<|endoftext|>"]):
        """
        流式读取文件，按文档边界切分，生成文本块
        每次yield的文本块以<|endoftext|>结尾，方便后续处理
        :param input_path: 输入文件的路径
        :param chunk_size: 每个文本块的大小，默认1MB
        :param special_tokens: 特殊token列表
        :return: 生成器，yield每个文本块
        """
        # 保存上一轮读取中，还没凑成完整文档的残留文本
        leftover = ""
        # 分隔符token的长度，后面切片用
        token_len = len(special_tokens[0])

        with open(input_path, 'r', encoding='utf-8') as f:
            while True:
                # 从文件中读取chunk_size大小的一段文本
                block = f.read(chunk_size)
                if not block:
                    # 如果已经读到文件末尾，且残留文本不为空，则yield残留文本
                    break
                # 将残留文本和新读取的文本块拼接
                block = leftover + block

                # 拼接完成, 清空残留缓存（如果后面还有残留会重新赋值）
                leftover = ""

                last_special_idx = block.rfind(special_tokens[0])  # 找到最后一个<|endoftext|>的位置
                if last_special_idx == -1:
                    # 如果当前块中没有找到<|endoftext|>，说明当前块还不完整，将整个块作为残留文本
                    leftover = block
                else:
                    # 从开头一直截取到【最后一个分隔符的末尾】，这部分是一组完整文档
                    # +token_len：把<|endoftext|>本身也包含进去
                    yield block[:last_special_idx + token_len]  # yield到最后一个<|endoftext|>的文本块
                    # 分隔符后面剩下的文本，作为残留，留给下一轮读取拼接
                    leftover = block[last_special_idx + token_len:]  # 将剩余的文本

        if leftover != "":
            # 如果文件读取完毕后，残留文本不为空，则yield残留文本
            yield leftover
```

### 3. BPE合并
举例:
- `word = "hello"`，`count = 120`（这个预分词片段在语料出现 120 次）
- `word_encodings["hello"] = [104,101,108,108,111]`
遍历相邻对：
`(104,101)`, `(101,108)`, `(108,108)`, `(108,111)`
每一对都 `pair_counts[pair] += 120`
##### 1. 实现`_count_pairs`方法
- 举例`vocabulary, word_counts, word_encodings, pair_strings`，需要了解它们都存的什么
Byte BPE，基础字节 id 0~256，这里简化，只用少量 id 演示
" hello" 出现 5次
"hi" 出现 2次
```python
# 1. vocabulary
`vocabulary`: token_id(int) → token_bytes(bytes)，
词表，key 是 token 整数 id，value 是对应的字节串
vocabulary = {
    104: b'h',
    101: b'e',
    108: b'l',
    111: b'o',
    32: b' ',
    105: b'i'
}

# 2. word_counts: 预分词字符串->出现次数
word_counts = {
    " hello": 5,
    "hi": 2
}

# 3. word_encodings：预分词字符串 → token id 列表
word_encodings = {
    " hello": [32, 104, 101, 108, 108, 111], # 空格 h e l l o
    "hi": [104, 105]                       # h i
}

# 4. ## pair_strings：缓存字典，`(id1,id2) → (bytes1, bytes2)`
# 作用：第一次碰到 pair 的时候，查表 vocabulary，把两个字节存下来做缓存，避免反复查询。
# key 是 pair 元组`(token_id1, token_id2)`，value 是一对字节。

```
###### 代码实现
```python
import collections

def _count_pairs(self, vocabulary, word_counts, word_encodings, pair_strings):
    """
    统计候选相邻token对(pair)的总出现频次
    Args:
        vocabulary: 当前BPE词表, token_id(int) -> token_bytes(bytes)
        word_counts: key=预分词片段字符串，value=该片段在语料中出现次数
        word_encodings: key=预分词片段字符串，value=片段对应的字节id整数列表
        pair_strings: 候选pair集合（缓存字典），只统计在这个集合里的pair，缓存pair对应的原始字节
    Returns:
        defaultdict: key是pair元组(token1, token2), value是这个pair的总出现次数
    """
    # 字典：记录每一对相邻token的总频次，key=(id1,id2)，value为累计出现次数，默认初始值0
    pair_counts = collections.defaultdict(int)

    # word: 预分词之后的片段字符串，count: 该片段在语料中出现的总次数
    for word, count in word_counts.items():
        # 获取当前预分词片段对应的编码：由token id组成的列表
        # 例：word=" hello", encoding=[104, 101, 108, 108, 111]
        encoding = word_encodings[word]

        # 遍历序列，取出所有相邻的一对一对token
        # range(len(encoding)-1)：i取 0,1,2...len(encoding)-2；保证 i+1 不越界
        for i in range(len(encoding) - 1):
            # 取出相邻两个token id，封装成元组作为pair键 (a,b)
            # pair存的是两个id_a, id_b
            pair = encoding[i], encoding[i + 1]

            # 累加频次：这个预分词片段一共出现count次，所以这一对pair一次性+count
            # 【重点优化】不是每一次出现+1，而是直接乘该预分词单元的总频次，大幅提速
            pair_counts[pair] += count

            # 如果这个pair是第一次遇到，不在pair_strings缓存字典里
            if pair not in pair_strings:
                # vocabulary[pair[0]]：第一个id对应的bytes
                # vocabulary[pair[1]]：第二个id对应的bytes
                # 把两个字节存入pair_strings缓存：pair元组 → (字节串1,字节串2)
                # 后续合并pair的时候直接读取，不用反复查表，减少重复计算
                # bytes_a = vocabulary[pair[0]], bytes_b = vocabulary[pair[1]] 
                # 所以这里就相当于存的是 原始`bytes`字节串
                # `pair_strings[(108,108)] = (b'l', b'l')`
                pair_strings[pair] = (vocabulary[pair[0]], vocabulary[pair[1]])

    return pair_counts

```

##### 2. train代码
> 从第三步开始看
1. 第一个问题: 为什么要`size < self.vocab_size`呢

`size` 代表**当前词表里一共有多少个 token**。每一轮 while 循环，我们**新增 1 个 token**，`size = size + 1`。
循环条件：`while size < self.vocab_size`

- 初始: 词表大小为 (初始 257 个 token：0~255（256 字节），id=256 是特殊 token。)
- 假设我们设置目标 `self.vocab_size = 260`。
- 目标：词表最终一共要有 260 个 token。
- 现在 size=257，小于 260 → 进入循环。
- 第一轮循环: 统计pair，选出最高频pair, 合并生成1个全新token->`vocabulary[size] = merge_bytes` → vocabulary [257] = 新子词 ->new_token_id = size -> `size +=1` → size 变成 258
2. `merge_pair, max_count = max(pair_counts.items(), key=lambda x: (x[1], pairs_strings[x[0]]))`这一行代码的原理

- 首先，`pair_counts`存储的是`{(id1,id2): 出现频次}`, `pair_counts.items()` → 迭代器，每一项是 `(pair元组, count)`，形如 `((108,108), 120)`
- lambda x 是什么? `x`代表`pair_counts.items()`里的每一条元素：`x = ((id_a, id_b), count)`
  - x[0]：pair，也就是`(id_a, id_b)`
  - x[1]: 这个 pair 的频次 count
  - `key=lambda x: (x[1], pairs_strings[x[0]])`,返回一个元组`(频次, pair对应的字节对)`
- 比较规则:
  - 元组比较规则（Python）
  - (A, B)比较的时候：
    - 先比较第一个元素A；A大，整体就大
    - 如果A想等，再比较第二个元素B，用来做稳定排序、打破平局。
3. 我不知道为什么预分词片段的编码序列要更新？
更新word_encodings，是为了让下一轮循环能够在【新的 token 序列】里继续找更长的子词。
举例子:
```python
word_encodings["hello"] = [104, 101, 108, 108, 111]
# 104:h,101:e,108:l,111:o
```
本轮选出的最高频的merge_pair=(108,108)，新的id=257，对应字节`b'll'`
不更新的话，`word_encodings["hello"]` 仍然是 `[104,101,108,108,111]`,下一轮调用`_count_pairs`统计相邻 pair，依然识别成：, `(104,101), (101,108), (108,108), (108,111)`,永远看不到 `(101,257)` 这个新 pair（`e + ll`）。

更新的操作是, 扫描`[104, 101, 108, 108, 111]`,发现`(108,108)`，替换成新 id `257`, 现在的序列变成了`h e ll o`, 下一轮的统计pair，会产生新的pair: (104, 101), (101, 257), (257, 111), 现在就有机会合并(101, 257)，也就是 `e + ll = ell`
```python
def train(self):
        
        # -------- 第一步: 初始化词表: 基础单元是全部256个字节(0-255)，再加上特殊token
        # {i: bytes([i]) for i in range(BYTES_NUM)}
        # BYTES_NUM一般 = 256，对应所有ASCII/utf8基础单字节
        # 例子: {0: b'\x00', 1: b'\x01', 2: b'\x02', ..., 255: b'\xff'}
        vocabulary = {i: bytes([i]) for i in range(BYTES_NUM)}
        # 把特殊token加入词表，特殊token的id从BYTES_NUM开始
        for i, sp_token in enumerate(self.special_tokens):
            vocabulary[BYTES_NUM + i] = sp_token.encode("utf8")
        
        # 当前词表总大小 = 256 + len(special_tokens)
        size = BYTES_NUM + len(self.special_tokens)

        # -------- 第二步: 预分词
        # 获取单词频率
        word_counts = self._pretokenize_and_count(self.input_path, self.special_tokens)

        # 将词转为utf-8编码
        word_encodings = {}
        for word in word_counts:
            word_encodings[word] = list(word.encode("utf-8"))

        # -------- 第三步: BPE合并
        merges = [] # 存储合并规则，每一条记录哪两个字节/子词合并在一起
        pairs_strings = {} # pair缓存: (id1, id2) -> (bytes1, bytes2)，用于统计pair频次时，避免重复拼接bytes
        # 循环：不断合并，直到词表size达到我们设定的目标vocab_size
        while size < self.vocab_size:
            # 统计当前所有相邻token对pair的全局频次，同时填充pairs_strings缓存
            pair_counts = self._count_pairs(vocabulary, word_counts, word_encodings, pairs_strings)
            # 选择频数最大的合并对, 将其合并
            # key=lambda x: (x[1], pairs_strings[x[0]])
            # # 优先按频次x[1]排序；频次相同的情况下，按字节串做次排序（稳定选择）
            merge_pair, max_count = max(pair_counts.items(), key=lambda x: (x[1], pairs_strings[x[0]]))
            # 拿到两个pair对应的字节，拼接得到新token的字节内容
            merge_bytes = vocabulary[merge_pair[0]] + vocabulary[merge_pair[1]]
            # 将新合并得到的token加入词表中，并分配当前size作为新token id
            vocabulary[size] = merge_bytes
            new_token_id = size
            size += 1 # 词表容量 + 1

            # ========== 更新所有预分词片段的编码序列（核心！） ==========
            # 遍历每一个预分词片段的token id序列，把所有【相邻=merge_pair】替换成新token id
            for word, word_tokens in word_encodings.items():
                i = 0
                new_tokens = []
                has_new_id = False # 标记这个序列是否发生了替换(没替换就不用回写)
                while i < len(word_tokens):
                    # 判断: 当前位置和下一个位置，刚好等于本次待合并的token_id
                    if i < len(word_tokens) - 1 and (word_tokens[i], word_tokens[i + 1]) == merge_pair:
                        new_tokens.append(new_token_id) # 替换成新合并后的id
                        i += 2 # 一次跳过两个位置，继续往后找
                        has_new_id = True # 标记这个序列发生了替换
                    else:
                        new_tokens.append(word_tokens[i]) # 没匹配到，保留原来的token id
                        i += 1 # 继续往后找
                # 如果这个片段发生了合并替换，更新word_encodings字典
                if has_new_id:
                    word_encodings[word] = new_tokens
            # 保存本次合并规则：记录是哪两个字节合并（推理阶段BPE需要merges列表）
            # merges存出的是(b'l', b'l'),bytes
            merges.append((vocabulary[merge_pair[0]], vocabulary[merge_pair[1]]))

        return vocabulary, merges
```
