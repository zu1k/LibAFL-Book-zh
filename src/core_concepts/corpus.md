# Corpus

语料库是存储测试案例的地方。我们将一个测试案例定义为一个输入和一组相关的元数据，例如，执行时间。

语料库可以用不同的方式存储测试用例，例如在磁盘上，或在内存中，或实现缓存以加快磁盘存储。

通常，当一个测试案例被认为是有趣的时候，它就会被添加到语料库中，但是语料库也被用来存储实现目标的测试案例（比如说，测试程序崩溃）。

与语料库相关的是，模糊检验器应要求从语料库中挑选下一个测试案例进行模糊检验。LibAFL中的分类法是CorpusScheduler，该实体代表了从语料库中提取测试案例的策略，例如FIFO。

Speaking about the code, [`Corpus`](https://docs.rs/libafl/0/libafl/corpus/trait.Corpus.html) and [`CorpusScheduler`](https://docs.rs/libafl/0/libafl/corpus/trait.CorpusScheduler.html) are traits.
