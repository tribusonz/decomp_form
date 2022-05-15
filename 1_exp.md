# 1. 指数関数の分解公式

　指数関数を知るについて，言語実装の観点から見てみよう。  
　現在，浮動小数点規格として`IEEE-754`がある。この規格では実数が限りなく0に近い状態でも持ちこたえるようにと非正規化数 *denormalized number* が組み込まれ，保証されている。そのため，これに対応しているならば， `(1 / MAX)` は本来には0.0であるにも関わらず，数値として持ちこたえている。  
　この規格は，`1.0`から離れれば離れるほど正確性を失っていき，`MIN`や`MAX`に近づくにつれ原型は保たれなくなる。それで，10進に戻せる桁数として，単精度では6，倍精度では15，拡張倍精度では18，4倍精度では33が保証されている。  
　この正確性は33などといったごく身近な数値であっても，指数関数では天文学のような数値に及びやすい。ゆえに，`exp(MAX)`ではむしろオーバフローしてくださいとベンチマークを図っているに等しいものになる。  
　したがって，すでに原型のない非正規化数の指数を取ろうことは，アンダフローから何かサンプルを採取しようとしているに近い。  

　ついで，`C/C++`のプリミティブ型の実装状況でこれを鑑みてみる。`exp2()`は最小・最大は各々`-1019`，`1023`となりうる。範囲外は，負はアンダフローとして`0.0`，正はオーバフローとして`Infinity`を返すようコンテクストスイッチさせるのが望ましい。しかしながら，ベンチマークでは面白い結果を得られた。

```
#include <float.h>
#include <math.h>

#include <stdio.h>

int
main(void)
{
	int i;

	puts("Calculation Range(MIN..MAX) of:");
	
	for (i = 0; 0.0 != expf(i); i--);
	printf("expf(): %d", i);
	for (i = 0; (1.0/0.0) != expf(i); i++);
	printf("..%d", i);
	printf(" (Logical: %d..%d)\n",
		   (int)(FLT_MIN_10_EXP / log10(M_E)),
		   (int)(FLT_MAX_10_EXP / log10(M_E)));
	
	for (i = 0; 0.0 != exp(i); i--);
	printf("exp(): %d", i);
	for (i = 0; (1.0/0.0) != exp(i); i++);
	printf("..%d", i);
	printf(" (Logical: %d..%d)\n",
		   (int)(DBL_MIN_10_EXP / log10(M_E)),
		   (int)(DBL_MAX_10_EXP / log10(M_E)));
	
	for (i = 0; 0.0 != expl(i); i--);
	printf("expl(): %d", i);
	for (i = 0; (1.0/0.0) != expl(i); i++);
	printf("..%d", i);
	printf(" (Logical: %d..%d)\n",
		   (int)(LDBL_MIN_10_EXP / log10(M_E)),
		   (int)(LDBL_MAX_10_EXP / log10(M_E)));

	for (i = 0; 0.0 != exp2f(i); i--);
	printf("exp2f(): %d", i);
	for (i = 0; (1.0/0.0) != exp2f(i); i++);
	printf("..%d", i);
	printf(" (Logical: %d..%d)\n",
		   (int)(FLT_MIN_10_EXP / log10(2)),
		   (int)(FLT_MAX_10_EXP / log10(2)));
	
	for (i = 0; 0.0 != exp2(i); i--);
	printf("exp2(): %d", i);
	for (i = 0; (1.0/0.0) != exp2(i); i++);
	printf("..%d", i);
	printf(" (Logical: %d..%d)\n",
		   (int)(DBL_MIN_10_EXP / log10(2)),
		   (int)(DBL_MAX_10_EXP / log10(2)));
	
	for (i = 0; 0.0 != exp2l(i); i--);
	printf("exp2l(): %d", i);
	for (i = 0; (1.0/0.0) != exp2l(i); i++);
	printf("..%d", i);
	printf(" (Logical: %d..%d)\n",
		   (int)(LDBL_MIN_10_EXP / log10(2)),
		   (int)(LDBL_MAX_10_EXP / log10(2)));
	
	return 0;
}
```

　結果は以下:  

```
Calculation Range(MIN..MAX) of:
expf(): -104..89 (Logical: -85..87)
exp(): -746..710 (Logical: -706..709)
expl(): -11400..11357 (Logical: -11354..11356)
exp2f(): -150..128 (Logical: -122..126)
exp2(): -1075..1024 (Logical: -1019..1023)
exp2l(): -16446..16384 (Logical: -16380..16383)
```

　なんと非正規化数も`0.0`としてではなく演算結果としている。これは意外だった。筆者はてっきり`MAX_EXP`や`MIN_EXP`などの機械定数*machine constant*の敷居を守っていると思っていたからである。  
　言語実装を考慮すると，計算できる限界を計測しつつ，計測不能であれば `x < 0 ? 0.0 : HUGE_VAL;` としてオーバフローを防ぐようコードする必要がある。  
 
 
