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

셔플링

```
$ shuf corpus.tsv > corpus.shuf.tsv
```

Train - 1200000 / Valid - 200000 / Test - 202409

```
$ head -n 1200000 corpus.shuf.tsv > corpus.shuf.train.tsv ; tail -n 402409 corpus.shuf.tsv | head -n 200000 > corpus.shuf.valid.tsv ; tail -n 202409 corpus.shuf.tsv > corpus.shuf.test.tsv
```

확인

```
$ wc -l corpus.shuf.*
$ head -n 3 corpus.shuf.*
```

다시 문제 발생,,, tsv인줄 알았는데 사실 csv였던거임...

Tokenization - 한국어와 영어를 나눠줘야 함

```
cut -f1 corpus.shuf.train.tsv > corpus.shuf.train.ko ; cut -f2 corpus.shuf.train.tsv > corpus.shuf.train.en
```
