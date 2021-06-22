# NLP_Generation

## Raw Data 획득

- 본 연구의 raw data는 AIHub의 "한국어-영어 번역(병렬) 말뭉치"를 이용하였다.
- 킹작권 & 용량의 문제로 레파지토리에 업로드는 X
- [AIHub](https://aihub.or.kr/aidata/87)

## Preprocessing

txt 파일을 하나의 tsv파일로 합치기

```
$ cat *.csv > corpus.tsv
```

여기서 고생을 많이 했다.. 왜인지.. terminal에선 txt파일의 한글 인코딩이 터미널에서 깨지는 현상이 발생하여, CSV utf-8 파일로 변경하여 작업을 진행하였다.
이렇게 하면 tsv파일에서 한글이 깨지지 않는다...
~~utf-8 만세! CSV 만세!~~

잘 나왔나 확인, 총 1602409문장

```
$ wc -l corpus.tsv
```

```
$ head -n 5 corpus.tsv
```

다시 문제 발생,,, tsv인줄 알았는데 사실 csv였던거임...

그래서 그냥... xlsx파일의 한국어와 영어 칼럼을 복사하여 VSCode로 연 tsv파일에 복사 - 붙여넣기 작업을 노가다로 해줬다...
엑셀의 두 칼럼을 txt(tsv)파일에 붙여넣으면 자동으로 tab으로 분리된다..
이렇게 해서 만든 corpus.tsv파일은 깨지지도 않고, tab으로 잘 나뉘어 있다...

셔플링

```
$ shuf corpus.tsv > corpus.shuf.tsv
```

### Data Set 분리

- Train - 1200000 / Valid - 200000 / Test - 202409

  ```
  $ head -n 1200000 corpus.shuf.tsv > corpus.shuf.train.tsv ; tail -n 402409 corpus.shuf.tsv | head -n 200000 > corpus.shuf.valid.tsv ; tail -n 202409 corpus.shuf.tsv > corpus.shuf.test.tsv
  ```

- 확인

  ```
  $ wc -l corpus.shuf.*
  $ head -n 3 corpus.shuf.*
  ```

- 한국어와 영어를 나눠줘야 함

  ```
  $ cut -f1 corpus.shuf.train.tsv > corpus.shuf.train.ko ; cut -f2 corpus.shuf.train.tsv > corpus.shuf.train.en

  $ cut -f1 corpus.shuf.valid.tsv > corpus.shuf.valid.ko ; cut -f2 corpus.shuf.valid.tsv > corpus.shuf.valid.en

  $ cut -f1 corpus.shuf.test.tsv > corpus.shuf.test.ko ; cut -f2 corpus.shuf.test.tsv > corpus.shuf.test.en
  ```

- cleaning은 본 연구에선 생략

### Tokenization

- Mecab을 활용하여 Pos Tagging
- tokenization 진행 후 다시 generation등의 과정을 거쳐 문장을 출력할 때, tokenization이 된 상태라면 사람이 읽기 굉장히 불편 할 수 있음. 따라서 post_tokenize과정을 거쳐줘야 함
- post*tokenize는 원래 띄워쓰기가 있던 곳에 *(underscore랑은 다른 특수문자)를 붙여서 토큰화 되었다는 것을 표시하는 것
- 이를 통해, detokenization가능

- 영어의 경우 'mosestokenizer'를 활용함

  ```
  $ cat data/corpus.shuf.train.ko | mecab -O wakati -b 99999 | python post_tokenize.py data/corpus.shuf.train.ko > data/corpus.shuf.train.tok.ko

  $ cat data/corpus.shuf.train.en | python tokenizer.py | python post_tokenize.py data/corpus.shuf.train.en > data/corpus.shuf.train.tok.en

  $ cat data/corpus.shuf.valid.ko | mecab -O wakati -b 99999 | python post_tokenize.py data/corpus.shuf.valid.ko > data/corpus.shuf.valid.tok.ko

  $ cat data/corpus.shuf.valid.en | python tokenizer.py | python post_tokenize.py data/corpus.shuf.valid.en > data/corpus.shuf.valid.tok.en

  $ cat data/corpus.shuf.test.ko | mecab -O wakati -b 99999 | python post_tokenize.py data/corpus.shuf.test.ko > data/corpus.shuf.test.tok.ko

  $ cat data/corpus.shuf.test.en | python tokenizer.py | python post_tokenize.py data/corpus.shuf.test.en > data/corpus.shuf.test.tok.en
  ```

- 확인
  ```
  $ head -n 2 data/corpus.shuf.*.tok.*
  ```

### Subword Segmentation

- BPE 압축 알고리즘을 통해 통계적으로 더 작은 의미 단위(subword)로 분절 수행
- BPE를 통해 OoV를 없앨 수 있으며, 이는 성능상 매우 큰 이점으로 작용한다.
- 한국어의 경우

  1. 띄워쓰기가 엉망, 따라서 normalization업이 Subword Segmentation을 적용하는 것은 위험할 수 있다.
  2. 따라서 형태소 분석기(Mecab)을 통해서 Tokenization을 진행 한 후 Subword Segmentation을 적용하는 것이 권장 됨

- [Wikibooks_BPE에 대한 대략적인 설명](https://wikidocs.net/22592)

- learning, vocabulary 생성, train set으로 생성

  - bpe 적용, 영어의 경우 50000번의 merge operation 수행
    ```
    $ python subword-nmt/learn_bpe.py --input data/corpus.shuf.train.tok.en --output bpe.en.model --symbols 50000 --verbose
    ```
  - 한국어의 경우 30000번의 merge operation 수행
    ```
    $ python subword-nmt/learn_bpe.py --input data/corpus.shuf.train.tok.ko --output bpe.ko.model --symbols 30000 --verbose
    ```

- apply (& command는 백그라운드 실행)

  ```
  $ cat data/corpus.shuf.train.tok.ko | python subword-nmt/apply_bpe.py -c bpe.ko.model > data/corpus.shuf.train.tok.bpe.ko &

  $ cat data/corpus.shuf.valid.tok.ko | python subword-nmt/apply_bpe.py -c bpe.ko.model > data/corpus.shuf.valid.tok.bpe.ko &

  $ cat data/corpus.shuf.test.tok.ko | python subword-nmt/apply_bpe.py -c bpe.ko.model > data/corpus.shuf.test.tok.bpe.ko &

  $ cat data/corpus.shuf.train.tok.en | python subword-nmt/apply_bpe.py -c bpe.en.model > data/corpus.shuf.train.tok.bpe.en &

  $ cat data/corpus.shuf.valid.tok.en | python subword-nmt/apply_bpe.py -c bpe.en.model > data/corpus.shuf.valid.tok.bpe.en &

  $ cat data/corpus.shuf.test.tok.en | python subword-nmt/apply_bpe.py -c bpe.en.model > data/corpus.shuf.test.tok.bpe.en &
  ```
