TinySegmenterMaker
=====

[TinySegmenter](http://chasen.org/~taku/software/TinySegmenter/)用の学習モデルを自作するためのツール．


## automakeのインストール

```
$ sudo apt -y install automake
```

## Boostのインストール

```
sudo apt -y install libboost-all-dev
```

## Mecabのインストール

```
$ sudo apt install -y mecab
```

## UniDicのインストール

```
$ sudo apt install -y unidic-mecab
```


## ko-dicのインストール

```
$ curl -L -o /tmp/mecab-ko-dic-2.1.1-20180720.tar.gz https://bitbucket.org/eunjeon/mecab-ko-dic/downloads/mecab-ko-dic-2.1.1-20180720.tar.gz
$ cd /tmp
$ tar zxvf mecab-ko-dic-2.1.1-20180720.tar.gz
$ cd mecab-ko-dic-2.1.1-20180720
$ ./autogen.sh
$ ./configure --with-dicdir=/var/lib/mecab/dic/ko-dic
$ make
$ sudo make install
```

## CC-CEDICTのインストール

```
$ curl -L -o /tmp/CC-CEDICT-MeCab.zip https://github.com/ueda-keisuke/CC-CEDICT-MeCab/archive/refs/heads/master.zip
$ unzip /tmp/CC-CEDICT-MeCab.zip -d /tmp
$ cd /tmp/CC-CEDICT-MeCab-master
$ /usr/lib/mecab/mecab-dict-index -f utf-8 -t utf-8
$ sudo cp -pr /tmp/CC-CEDICT-MeCab-master /var/lib/mecab/dic/cc-cedict
```

## pyenvのインストール

```
$ curl https://pyenv.run | bash
```

## Pythonのビルドに必要なパッケージのインストール

```
$ sudo apt install build-essential libbz2-dev libdb-dev \
  libreadline-dev libffi-dev libgdbm-dev liblzma-dev \
  libncursesw5-dev libsqlite3-dev libssl-dev \
  zlib1g-dev uuid-dev tk-dev
```

## Pythonのインストール

```
$ pyenv install 3.7.17
```

## Pythonのバージョンを指定して仮想環境を作成

```
$ pyenv local 3.7
$ python -m venv .venv
$ source .venv/bin/activate
```

## データソースのダウンロード

```
$ curl -o /mnt/e/jawiki-20230701-pages-articles-multistream.xml.bz2 https://dumps.wikimedia.org/jawiki/20230701/jawiki-20230701-pages-articles-multistream.xml.bz2

$ curl -o /mnt/e/kowiki-20230701-pages-articles-multistream.xml.bz2 https://dumps.wikimedia.org/kowiki/20230701/kowiki-20230701-pages-articles-multistream.xml.bz2

$ curl -o /mnt/e/zhwiki-20230701-pages-articles-multistream.xml.bz2 https://dumps.wikimedia.org/zhwiki/20230701/zhwiki-20230701-pages-articles-multistream.xml.bz2
```

## Wikiextractorのインストール

```
$ pip install wikiextractor
```

## Wikiextractorの実行
```
$ wikiextractor --json -o /mnt/e/jawiki /mnt/e/jawiki-20230701-pages-articles-multistream.xml.bz2

$ wikiextractor --json -o /mnt/e/kowiki /mnt/e/kowiki-20230701-pages-articles-multistream.xml.bz2

$ wikiextractor --json -o /mnt/e/zhwiki /mnt/e/zhwiki-20230701-pages-articles-multistream.xml.bz2
```

## コーパスの準備

```
files=$(find /mnt/e/jawiki -type f | sort)
for file in $files; do
  echo "$file"
  cat "$file" | jq -r .text | sed -e 's/&quot;/"/g' -e 's/&apos;/'\''/g' -e 's/&lt;/</g' -e 's/&gt;/>/g' -e 's/&amp;/\&/g' | mecab -d /var/lib/mecab/dic/unidic -b 32768 -O wakati | sed -E ':start; s/([0-9]) +([0-9])/\1\2/; t start' >> /mnt/e/jawiki_corpus.txt
done

files=$(find /mnt/e/kowiki -type f | sort)
for file in $files; do
  echo "$file"
  cat "$file" | jq -r .text | sed -e 's/&quot;/"/g' -e 's/&apos;/'\''/g' -e 's/&lt;/</g' -e 's/&gt;/>/g' -e 's/&amp;/\&/g' | mecab -d /var/lib/mecab/dic/ko-dic -b 32768 -O wakati | sed -E ':start; s/([0-9]) +([0-9])/\1\2/; t start' >> /mnt/e/kowiki_corpus.txt
done

files=$(find /mnt/e/zhwiki -type f | sort)
for file in $files; do
  echo "$file"
  cat "$file" | jq -r .text | sed -e 's/&quot;/"/g' -e 's/&apos;/'\''/g' -e 's/&lt;/</g' -e 's/&gt;/>/g' -e 's/&amp;/\&/g' | mecab -d /var/lib/mecab/dic/cc-cedict -b 40960 -O wakati | sed -E ':start; s/([0-9]) +([0-9])/\1\2/; t start' >> /mnt/e/zhwiki_corpus.txt
done
```

## 素性の抽出

スペースで分かち書きしたコーパスをあらかじめ準備しておきます．
コーパスから分かち書きの情報と素性を取り出します．

``` bash
$ cat *_corpus.txt > corpus.txt
$ time ./extract < /mnt/e/jawiki_corpus.txt > /mnt/e/jawiki_features.txt
$ time ./extract < /mnt/e/kowiki_corpus.txt > /mnt/e/kowiki_features.txt
$ time ./extract < /mnt/e/zhwiki_corpus.txt > /mnt/e/zhwiki_features.txt
```

## モデルの学習

AdaBoostを用いて学習します．
新しい弱分類器の分類精度が0.001以下，繰り返し回数が10000回以上となったら学習を終了します．

``` bash
$ g++ -O3 -o train train.cpp # コンパイル
$ ./train -t 0.001 -n 10000 /mnt/e/features.txt model # 学習
```

きちんと分割できるか実際に試してみます．

``` bash
$ ./segment model
私の名前は中野です
私 の 名前 は 中野 です
```

## マルチスレッドで学習

学習プログラム train はマルチスレッドに対応しています．
コンパイルにはboostが必要です．

``` bash
$ g++ -DMULTITHREAD -lboost_thread-mt -O3 -o train train.cpp
```


## 学習済みモデル

JEITA\_Genpaku\_ChaSen\_IPAdic.model は
電子情報技術産業協会(JEITA)が公開している[形態素解析済みコーパス](http://anlp.jp/NLP_Portal/jeita_corpus/index.html)
を学習したものです．
[プロジェクト杉田玄白](http://www.genpaku.org/)を茶筌+IPAdicで解析したものを使用しました．

RWCP.model は オリジナルの [TinySegmenter](http://chasen.org/~taku/software/TinySegmenter/) から
モデルの部分のみを取り出したものです．


## 再学習

学習済みのモデルとコーパスから学習を再開し，性能を向上させることができます．
コーパスは学習に使用したものを想定していますが，別のコーパスを使っても動作はするはずです．

``` bash
$ ./train -t 0.0001 -n 10000 -M model features.txt model_new
```

## ライブラリの作成

makerコマンドで各種言語用のライブラリを作れます．
allを指定することで，対応しているすべての言語向けのライブラリを出力します．

``` bash
$ ./maker javascript < model
$ ./maker perl < model
$ ./maker ruby < medel
$ ./maker python < model
$ ./maker cpp < model
$ ./maker tex < model
$ ./maker vim < model
$ ./maker go < model
$ ./maker jsx < model
$ ./maker csharp < model
$ ./maker all < model # 上のライブラリをすべて作成します
```


### JavaScript

``` javascript
<script src="tinysegmenter.js"></script>
<script>

var segmenter = new TinySegmenter();                 // インスタンス生成

var segs = segmenter.segment("私の名前は中野です");  // 単語の配列が返る

alert(segs.join(" | "));  // 表示

</script>
```

もちろん node.js でも使えます．

``` javascript
var TinySegmenter = require('./tinysegmenter.js').TinySegmenter;

var segmenter = new TinySegmenter();                 // インスタンス生成

var segs = segmenter.segment("私の名前は中野です");  // 単語の配列が返る

console.log(segs);
```

### Perl

``` perl
use utf8;
use tinysegmenter;

my $str = '私の名前は中野です';
my @words = tinysegmenter->segment($str);
```

### Ruby

``` ruby
require './tinysegmenter'
ts = TinySegmenter.new
puts ts.segment("私の名前は中野です");
```

### Python

``` python
from tinysegmenter import TinySegmenter
segmenter = TinySegmenter()
print segmenter.segment(u'私の名前は中野です')
```

### C++

``` c++
#include <iostream>
#include <string>
#include <vector>
#include "tinysegmenter.hpp"

using namespace std;

int main() {
    TinySegmenter segmenter;
    string s = "私の名前は中野です";
    vector<string> v = segmenter.segment(s);
    for(int i = 0; i < v.size(); ++i) {
        cout << v[i] << endl;
    }
}
```

### TeX

```tex
\documentclass{jsarticle}
\usepackage{tinysegmenter}
\begin{document}
\TinySegmenter{-}{私の名前は中野です}
\end{document}
```

### Vim script

```vim
:echo tinysegmenter#segment('私の名前は中野です')
```

### Go

```go
package main

import (
	"fmt"
	"tinysegmenter"
)

func main() {
	s := tinysegmenter.NewSegmenter()
	segs := s.Segment("私の名前は中野です")
	for _, seg := range segs {
		fmt.Printf("%s\n", seg)
	}
}
```

### JSX

```javascript
import "tinysegmenter.jsx";

class _Main
{
    static function main(args : string[]) : void
    {
        log TinySegmenter.segment("私の名前は中野です");
    }
}
```

### Csharp

``` csharp
static void Main(string[] args)
{
    tinysegmenter segmenter = new tinysegmenter();
    List<string> segments = segmenter.Segment("私の名前は中野です");

    for(int i = 0; i < segments.Count; i++)
    {
        Console.WriteLine(segments[i]);
    }
}
```

### Julia

```jl
using TinySegmenter

tokenize("私の名前は中野です")
```
