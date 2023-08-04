+++
title = "C標準でasprintfを実装する"
date = 2023-08-04
[taxonomies]
tags = ["C"]
+++

## `asprintf` って何ぞや

man asprintfより一部を引用

>  `int asprintf(char **strp, const char *fmt, ...);`
>
> The functions `asprintf()` and `vasprintf()` are analogs of `sprintf(3)` and `vsprintf(3)`, except that they allocate a string large enough to hold the output including the terminating null byte ('\0'), and return a pointer to it via the first argument. This pointer should be passed to `free(3)` to release the allocated storage when it is no longer needed.

まあつまり、`sprintf` やら `snprintf` やらだとこちら側でよしなにバッファを確保してやる必要があるところを、関数側が勝手に確保してくれるといった感じだ。これによって `snprintf` で `n` が十分に大きくなく、出力内容が切り捨てられたり、あるいは `sprintf` でバッファオーバーランが発生、なんて事だ、もう助からないゾ♡みたいなことにはならないというわけだ。

しかし、こいつには一つ問題がある。それはこいつがGNU拡張であるということだ。C標準やPOSIXにはこいつはいない。そこで、こいつをC標準(C99)の範囲で実装していこうのコーナーです。

## 実装する
さて、 `asprintf` の肝は出力文字列を格納するメモリを勝手に確保してくれるところである。逆に、その他は単に `vsnprintf` や `malloc` に投げつければ終わる仕事だ。

しかし、出力文字列を書き込むメモリを確保するためにはフォーマット結果の文字数を知る必要がある。これには `snprintf` (実際に使うのは `vsnprintf` であるが、引数の渡し方の違いでしかないので真にそこは引っかからなくていい)を使う。実は `snprintf` は `n` を `0` にして呼ぶと `s` に何も書き込まないうえに、`s` はヌルポインタで良い。そして、 `snprintf` は `n` が十分大きかった場合に書き込まれるはずだった文字数、つまり、フォーマットされた後の文字数を返す{{fnref(index=1)}}。

C99(N1256) 7.19.6.5 The snprintf functionより一部を引用
 
> `int snprintf(char * restrict s, size_t n, const char * restrict format, ...);`
>
> The `snprintf` function is equivalent to `fprintf`, except that the output is written into an array (specified by argument `s`) rather than to a stream. If `n` is zero, nothing is written, and `s` may be a null pointer. (snip)
>
> The `snprintf` function returns the number of characters that would have been written had `n` been sufficiently large, not counting the terminating null character, or a negative value if an encoding error occurred.

これを利用するわけだ。あとはフォーマット後の文字数 **+1** バイトのメモリを確保して、そこに書き込んでやればよい。

これをコードにすると例えば下記のようになる

```c
int myasprintf(char **bufp,const char * restrict format,...){
    va_list arg;

    va_start(arg, format);
    int len = vsnprintf(NULL,0,format,arg) + /* '\0' */ 1;
    va_end(arg);

    if (len < 0) return -1;

    *bufp = malloc(len);
    if(!*bufp) return -1;
    
    va_start(arg, format);
    int ret = vsnprintf(*bufp,len,format,arg);
    va_end(arg);
    
    if (ret < 0) free(*bufp);
    
    return ret;
}
```
FreeBSDの実装だとエラーの際には `strp` (この実装における `bufp` )を `NULL` にセットするらしい{{fnref(index=2)}}が、これは恐らくエラーの際に `strp` を通して渡したポインタを `free` に突っ込めるようにするためであろうから、今回はエラーが起きたときには確保したメモリを関数側で開放しておくこととした。あと、 `malloc` の部分で `char` のサイズをかけ忘れているように見えるかもしれないが、 `sizeof(char)` は常に1であるから{{fnref(index=3)}}、これで問題ない。この点を明示的に `len * sizeof(char)` とするかだとか、 `malloc` の返り値をキャストするかだとか、ifの後をブロックにするかなどは各自の思想に合わせて調整していただきたい。上記コードはCC0とする。

---

{{fndef(index=1,content="終端ヌル文字は含まれていないことに注意されたい")}}

{{fndef(index=2,content="man asprintfより一部を引用

> The FreeBSD implementation sets `strp` to NULL on error.
")}}

{{fndef(index=3,content="
C99(N1256) 6.5.3.4 The sizeof operatorより一部を引用

> When applied to an operand that has type `char`, `unsigned char`, or `signed char`,
(or a qualified version thereof) the result is 1. 
")}}
