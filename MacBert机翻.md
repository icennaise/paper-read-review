变换器双向编码表示法（Bidirectional Encoder Representations from Transformers，BERT）在各种NLP任务中显示出惊人的改进，并且连续的变体已经被提出来进一步提高预训练语言模型的性能。在本文中，我们的目标是重新审视中文预训练语言模型，以检验其在非英语语言中的有效性，并向社区发布中文预训练语言模型系列。我们还提出了一个简单而有效的模型，叫做MacBERT，它在几个方面改进了RoBERTa，特别是采用MLM作为校正的掩蔽策略（Mac）。我们在八个中文NLP任务上进行了广泛的实验，重新审视了现有的预训练语言模型以及提出的MacBERT。实验结果表明，MacBERT可以在许多NLP任务上取得最先进的性能，我们也消除了一些细节，这些发现可能有助于未来的研究。

# 1 引言
变换器的双向编码表示（Bidirectional Encoder Representations from Transformers，BERT）（Devlin et al.，2019）在最近的自然语言处理研究中已经大受欢迎，并被证明是有效的，它利用大规模的无标签训练数据，生成丰富的上下文表示。当我们穿越几个流行的机器阅读理解基准，如SQuAD（Rajpurkar等人，2018）、CoQA（Reddy等人，2019）、QuAC（Choi等人，2018）、NaturalQuestions（Kwiatkowski等人，2019）、RACE（Lai等人。2017），我们可以看到大多数表现最好的模型都是基于BERT及其变体（Dai等人，2019；Zhang等人，2019；Ran等人，2019），这表明预训练的语言模型已经成为自然语言处理领域新的基本组成部分。

从BERT开始，社区在优化预训练语言模型方面取得了巨大而迅速的进展，如ERNIE（Sun等人，2019a）、XLNet（Yang等人，2019）、RoBERTa（Liu等人，2019）、SpanBERT（Joshi等人，2019）、ALBERT（Lan等人，2019）、ELECTRA（Clark等人，2020）等等。然而，训练基于Transformer（Vaswani等人，2017）的预训练语言模型并不像我们用来训练词嵌入或其他传统神经网络那样容易。通常情况下，训练一个强大的BERT-大型模型，它有24层Transformer，有3.3亿个参数，到收敛需要高内存的计算设备，如TPU，这是很昂贵的。另一方面，虽然已经发布了各种预训练的语言模型，但大多数都是基于英语的，而在其他语言上建立强大的预训练语言模型的努力很少。

在本文中，我们旨在建立中文预训练语言模型系列并向公众发布，以促进研究界的发展，因为中文和英文是世界上使用最多的语言之一。我们重新审视了现有的流行预训练语言模型，并将其调整为中文，以了解这些模型在英语以外的语言中是否具有良好的通用性。此外，我们还提出了一个新的预训练语言模型，称为MacBERT，它将原来的MLM任务替换为MLM as correction（Mac）任务，并缓解了预训练和微调阶段的差异。在八个流行的中文NLP数据集上进行了广泛的实验，范围从句子级到文档级，如机器阅读理解、文本分类等。实验结果表明，与其他预训练的语言模型相比，所提出的MacBERT可以在大多数任务中获得明显的收益，并给出了详细的消解方法，以更好地检查改进的构成。本文的贡献列举如下。

- 进行了广泛的实证研究，通过仔细分析，重新审视了中文预训练语言模型在各种任务上的表现。

- 我们提出了一种新的预训练语言模型，名为MacBERT，通过用相似的词掩盖该词，缓解了预训练和微调阶段的差距，这在下游任务中被证明是有效的。

- 为了进一步加快中文NLP的未来研究，我们创建了中文预训练语言模型系列并向社区发布。

# 2 相关工作 

在本节中，我们重新审视了最近自然语言处理领域中具有代表性的预训练语言模型的技术。表1描述了这些模型的总体比较，以及所提出的MacBERT。我们在下面的小节中详细说明它们的关键部分。

## 2.1 BERT 

BERT（Bidirectional Encoder Representations from Transformers）（Devlin等人，2019）已被证明在自然语言处理研究中是成功的。BERT旨在通过在所有Transformer层中共同调节左右语境来预训练深度双向表征。主要而言，BERT由两个预训练任务组成。屏蔽语言模型（MLM）和下句预测（NSP）。

- MLM：随机掩盖输入中的一些标记，目标是仅根据其上下文来预测原词。

- NSP：预测B句是否是A句的下一句。

后来，他们进一步提出了一种叫做全词屏蔽（wwm）的技术，用于优化MLM任务中的原始屏蔽。在这个设定中，我们不是随机选择WordPiece（Wu et al., 2016）令牌进行掩蔽，而是总是一次性掩蔽所有对应于整个单词的令牌。这将明确地迫使模型在MLM预训练任务中恢复整个单词，而不是仅仅恢复WordPiece标记（Cui等人，2019a），这更具挑战性。由于整词掩蔽只影响预训练过程中的掩蔽策略，它不会给下游任务带来额外的负担。此外，由于训练预训练语言模型的计算成本很高，他们还公布了所有的预训练模型以及源代码，这刺激了社区对预训练语言模型研究的极大兴趣。

## 2.2 ERNIE 

ERNIE（Enhanced Representation through kNowledge IntEgration）（Sun等人，2019a）旨在优化BERT的屏蔽过程，包括实体级屏蔽和短语级屏蔽。与选择输入中的随机词不同，实体级屏蔽将屏蔽命名的实体，这些实体通常由几个词组成。短语级屏蔽是屏蔽连续的词，这与N-gram屏蔽策略相似（Devlin等人，2019；Joshi等人，2019）2。

## 2.3 XLNet 

Yang等人（2019）认为，现有的基于自动编码的预训练语言模型，如BERT，存在着预训练和微调阶段的差异，因为屏蔽符号[MASK]永远不会出现在微调阶段。为了缓解这个问题，他们提出了XLNet，它是基于Transformer-XL的（Dai等人，2019）。XLNet主要以两种方式进行修改。首先是在输入的因式分解顺序的所有排列上最大化预期似然，在这里他们称之为排列语言模型（PLM）。另一种是将自动编码语言模型改为自回归模型，这与传统的统计语言模型类似。

## 2.4 RoBERTa 

RoBERTa（Robustly Optimized BERT Pretraining Approach）（Liu et al., 2019）旨在采用原始的BERT架构，但做了更精确的修改，以显示BERT的强大，这一点被低估了。他们对BERT中的各个组成部分进行了仔细的比较，包括掩蔽策略、训练步骤等。经过全面的评估，他们得出了几个有用的结论，使BERT更加强大，主要包括：1）用更大的批次和更长的序列在更多的数据上进行更长时间的训练；2）取消下一句话的预测，使用动态屏蔽。

## 2.5 ALBERT 

ALBERT（A Lite BERT）（Lan等人，2019）主要解决了BERT的内存假设较高和训练速度慢的问题。ALBERT引入了两种参数减少技术。第一个是因子化嵌入参数化，将嵌入矩阵分解为两个小矩阵。第二个是跨层参数共享，在ALBERT的每一层共享Transformer权重，这将大大减少参数。此外，他们还提出了句序预测（SOP）任务来代替传统的NSP预训练任务。3 我们也训练了中文XLNet，但它只在阅读理解数据集上表现出有竞争力的性能。我们把这些结果放在附录中。

## 2.6 ELECTRA 

ELECTRA（高效学习编码器，准确分类标记替换）（Clark等人，2020）采用了一个新的生成器-判别器框架，与GAN（Goodfellow等人，2014）类似。生成器通常是一个小型的MLM，学习预测被掩盖的标记的原话。鉴别器被训练来鉴别输入标记是否被生成器替换。请注意，为了实现有效的训练，判别器只需要预测一个二进制标签来表示 "替换"，而不像MLM那样应该预测准确的被屏蔽的词。在微调阶段，只使用判别器。

# 3 中文预训练的语言模型 

虽然我们相信前面的工作中的大部分结论在英语条件下是正确的，但我们想知道这些技术在其他语言中是否仍然有很好的概括性。在这一节中，我们说明了现有的预训练语言模型是如何适应中文的。此外，我们还提出了一个名为MacBERT的新模型，它采用了以前模型的优点以及新设计的组件。需要注意的是，由于这些模型都是源于BERT，没有改变输入的性质，所以在微调阶段不需要修改就可以适应这些模型，相互替换起来非常灵活。

## 3.1 BERT-wwm & RoBERTa-wwm 

在最初的BERT中，使用WordPiece tokenizer（Wu等人，2016）将文本分割成WordPiece tokens，其中一些单词将被分割成几个小片段。全词屏蔽（wwm）缓解了只屏蔽整个单词的一部分的缺点，这对模型来说更容易预测。在中文条件下，WordPiece tokenizer不再将单词分割成小片段，因为汉字不是由类似字母的符号构成的。我们使用传统的中文词语分割（CWS）工具，将文本分割成若干个词语。通过这种方式，我们可以采用中文中的整词屏蔽，而不是单个汉字的屏蔽。在实施过程中，我们严格遵循原有的整词屏蔽代码，并没有改变其他的组成部分，比如词语屏蔽的百分比等。我们使用LTP（Che等人，2010）进行中文单词分割来识别单词的边界。请注意，整个单词的屏蔽只影响到预训练阶段的屏蔽标记的选择。BERT的输入仍然使用WordPiece tokenizer来分割文本，这与原始的BERT是相同的。

同样，全词屏蔽也可以应用于RoBERTa，其中没有采用NSP任务。图1中描述了一个整词屏蔽的例子。

## 3.2 MacBERT 

在本文中，我们利用以前的模型，提出了一个简单的修改，导致微调任务的显著改善，在这里我们把这个模型称为MacBERT（MLM作为修正BERT）。MacBERT与BERT共享相同的预训练任务，并作了一些修改。对于MLM任务，我们进行了以下的修改。

- 我们使用全词屏蔽以及Ngram屏蔽策略来选择屏蔽的候选标记，对于词级的unigram到4-gram的百分比为40%，30%，20%，10%。

- 在微调阶段，[MASK]标记从未出现过，我们建议使用类似的词来进行遮蔽，而不是用[MASK]标记进行遮蔽。相似词是通过使用同义词工具包（Wang and Hu, 2017）获得的，它是基于word2vec（Mikolov等人，2013）相似性计算的。如果选择了一个N-gram进行屏蔽，我们将单独找到类似的词。在极少数情况下，当没有类似的词时，我们会退化到使用随机词替换。

- 我们使用15%的输入词进行掩蔽，其中80%将用相似词替换，10%用随机词替换，其余的10%保持原词。

对于类似NSP的任务，我们执行ALBERT（Lan等人，2019年）引入的句子顺序预测（SOP）任务，其中负面样本是通过切换两个连续句子的原始顺序创建的。我们在第6.1节中消减了这些修改，以更好地展示每个组件的贡献。

# 4 实验设置 

## 4.1 预训练语言模型的设置 

我们下载了维基百科dump4（截至2019年3月25日），并按照Devlin等人（2019）的建议用WikiExtractor.py进行了预处理，得到了1307个提取文件。我们在这个转储中同时使用了简体中文和繁体中文。在清理原始文本（如删除html标签器）和分离文件后，我们获得了大约0.4B个单词。由于中文维基百科的数据相对较少，除了中文维基百科，我们还使用扩展训练数据来训练这些预训练的语言模型（模型名称中标有ext）。内部收集的扩展数据包含百科全书、新闻和问题回答网络，有54亿字，比中文维基百科大十倍以上。请注意，我们总是使用MacBERT的扩展数据，而省略ext标记。为了识别中文单词的边界，我们使用LTP（Che等人，2010）进行中文单词分割。我们使用官方创建的pretraining data.py将原始输入文本转换为预训练实例。

为了更好地从现有的预训练语言模型中获取知识，我们没有从头开始训练我们的基础级模型，而是使用官方的中文BERT-base，继承其词汇和权重。然而，对于大级别的模型，我们必须从头开始训练，但仍然使用基础级别模型所提供的相同词汇。

对于训练BERT系列，我们采用了Devlin等人（2019）建议的方案，即先在最大长度为128个标记上进行训练，然后再在512个标记上进行训练。然而，我们根据经验发现，这将导致对长序列任务（如阅读理解）的不充分适应。在这种情况下，对于RoBERTa和MacBERT，我们在整个预训练过程中直接使用最大长度为512，这在Liu等人（2019）中被采用。对于小于1024的批次，我们采用BERT中带有权重衰减优化器的原始ADAM（Kingma和Ba，2014）进行优化，在较大的批次中使用LAMB优化器（You等人，2019）以获得更好的扩展性。根据模型的大小，预训练是在单个谷歌云TPU5 v3-8（相当于单个TPU）或TPU Pod v3-32（相当于4个TPU）上进行。具体来说，对于MacBERT-large，我们训练了2M步，批次大小为512，初始学习率为1e-4。

训练细节显示在表3中。为了清楚起见，我们没有列出 "ext "模型，其中其他参数与没有在扩展数据上训练的模型相同。

## 4.2 微调任务的设置 

为了彻底测试这些预训练的语言模型，我们对各种自然语言处理任务进行了广泛的实验，涵盖了广泛的文本长度，即从句子级到文档级。任务细节如表2所示。具体来说，我们选择了以下八个流行的中文数据集。

- 机器阅读理解（MRC）。CMRC 2018（Cui等人，2019b），DRCD（Shao等人，2018），CJRC（Duan等人，2019）。

- 单句分类（SSC）。ChnSentiCorp（Tan和Zhang，2008），THUCNews（Li和Sun，2007）。

- 句子对分类（SPC）。XNLI（Conneau等人，2018），LCQMC（Liu等人，2018），BQ语料库（Chen等人，2018）。

为了进行公平的比较，对于每个数据集，我们保持相同的超参数（如最大长度、热身步骤等），只将每个任务的初始学习率从1e-5调整到5e-5。请注意，初始学习率是在原始的中文BERT上调整的，通过单独调整学习率，有可能获得另一种收益。为了保证结果的可靠性，我们对同一实验进行了十次运行。最佳初始学习率是通过选择最佳的平均开发集性能来确定的。我们报告最高分和平均分，以同时评估峰值和平均性能。

对于除ELECTRA以外的所有模型，我们对每个任务使用相同的初始学习率设置，如表2所描述的。对于ELECTRA模型，我们按照Clark等人（2020）的建议，对基础级模型使用1e-4的通用初始学习率，对大级别模型使用5e-5。

由于现有的各种中文预训练语言模型，如ERNIE（Sun等人，2019a）、ERNIE 2.0（Sun等人，2019b）、NEZHA（Wei等人，2019）中的预训练数据有很大不同，我们只比较BERT（Devlin等人。2019）、BERT-wwm、BERT-wwm-ext、RoBERTawwm-ext、RoBERTa-wwm-ext-large、ELECTRA，以及我们的MacBERT，以确保不同模型之间的相对公平的比较，其中除了Devlin等人（2019）的原始中文BERT，所有模型都由我们自己训练。我们在TensorFlow框架（Abadi等人，2016）下进行了实验，并对Devlin等人（2019）提供的微调脚本6稍作修改，以更好地适应中文。

# 5 结果 

## 5.1 机器阅读理解 

机器阅读理解（MRC）是一个有代表性的文档级建模任务，需要根据给定的段落回答问题。我们主要在三个数据集上测试这些模型。CMRC 2018，DRCD，和CJRC。

- CMRC 2018。一个跨度提取的机器阅读理解数据集，与SQuAD（Rajpurkar等人，2016）类似，为给定的问题提取一个段落跨度。

- DRCD：这也是一个跨度提取的MRC数据集，但是是繁体中文。

- CJRC：类似于CoQA（Reddy等人，2019），它有是/否问题、无答案问题和跨度提取问题。数据收集自中国法律判决书。注意，我们只使用small-train-data.json进行训练。

结果在表4、5、6中描述。使用额外的预训练数据将导致进一步的改进，如BERT-wwm和BERT-wwm-ext之间的比较所示。这就是为什么我们对RoBERTa、ELECTRA和MacBERT使用扩展数据。此外，拟议的MacBERT在所有阅读理解数据集上都产生了明显的改进。值得一提的是，我们的MacBERT-large可以在CMRC 2018的挑战集上实现最先进的F1，即60%，这需要更深入的文本理解。

此外，应该注意的是，虽然DRCD是一个传统的中文数据集，但用额外的大规模简体中文进行训练也会有很大的积极作用。由于简体和繁体中文有许多相同的字符，使用一个强大的预训练语言模型，只用一些繁体中文数据也可以带来改进，而不需要将繁体中文字符转换成简体字符。

关于CJRC，其中文本是以关于中国法律的专业方式写成的，BERTwwm比BERT显示出适度的改进，但不是那么突出，表明在非一般领域的微调任务中需要进一步的领域适应。然而，通过增加一般的训练数据将导致改善，这表明当没有足够的领域数据时，我们也可以使用大规模的一般数据作为补救措施。

## 5.2 单句分类 

对于单句分类任务，我们选择ChnSentiCorp和THUCNews数据集。我们使用ChnSentiCorp数据集来评估情感分类，其中文本应该被分为正面或负面标签。THUCNews是一个包含不同类型的新闻的数据集，其中的文本通常很长。在本文中，我们使用了一个包含10个领域（均匀分布）的50K新闻的版本，包括体育、金融、技术等。7结果显示，我们的7 https://github.com/gaussic/text-classification-cnn-rnn MacBERT可以比基线有适度的改进，因为这些数据集已经达到了非常高的准确率。

## 5.3 句子对分类 

对于句对分类任务，我们使用了XNLI数据（中文部分），大规模中文问题匹配语料库（LCQMC）和BQ语料库，这些数据需要输入两个序列并预测它们的关系。我们可以看到MacBERT优于其他模型，但改进幅度不大，平均分略有提高，但峰值性能不如RoBERTa-wwm-ext-large好。我们怀疑这些任务对输入的细微差别不如阅读理解任务敏感。由于句对分类只需要生成整个输入的统一表示，因此导致了适度的改进。


6 讨论 

虽然我们的模型在各种中文任务上取得了明显的改进，但我们想知道这些改进的基本成分来自哪里。为此，我们对MacBERT进行了详细的消融，以证明它们的有效性，我们还比较了现有的预训练语言模型在英语中的说法，看看它们的修改在另一种语言中是否仍然成立。

6.1 MacBERT的有效性 

我们进行了消融，以检查MacBERT中每个组件的贡献，在所有微调任务中进行了彻底的评估。结果显示在表9中。总体平均分数是通过平均每个任务的测试分数得到的（EM和F1指标在总体平均之前是平均的）。从总体上看，删除MacBERT中的任何组件都会导致平均性能的下降，这表明所有的修改都有助于整体改进。具体来说，最有效的修改是N-gram masking和类似的单词替换，这是在屏蔽语言模型任务上的修改。当我们比较N-gram masking和相似词替换时，我们可以看到明显的优点和缺点，其中N-gram masking似乎在文本分类任务中更有效，而阅读理解任务的表现似乎更受益于相似词替换任务。通过将这两个任务结合起来，将相互弥补，在两种体裁上都有更好的表现。

NSP任务没有显示出像MLM任务那样的重要性，这表明设计一个更好的MLM任务来充分释放文本建模的力量更为重要。另外，我们还比较了下句预测（Devlin等人，2019）和句序预测（Lan等人，2019）任务，以更好地判断哪个任务更强大。结果显示，句序预测任务确实比原来的NSP表现得更好，尽管它不是那么突出。SOP任务需要识别两个句子的正确顺序，而不是使用一个随机句子，这对机器来说更容易识别。与文本分类任务相比，去掉SOP任务将导致阅读理解任务的明显下降，这表明有必要设计一个类似NSP的任务来学习两个片段之间的关系（例如，阅读理解任务中的段落和问题）。

6.2 对MLM任务的调查 

如上节所述，最主要的预训练任务是遮蔽语言模型及其变体。掩蔽语言模型任务依赖于两个方面。1）选择要被屏蔽的标记，和2）替换被选中的标记。在上一节中，我们已经证明了选择掩蔽标记的有效性，如整个单词掩蔽或N-gram掩蔽等。现在我们要研究的是，所选标记的替换将如何影响预训练的语言模型的性能。为了研究这个问题，我们绘制了CMRC 2018和DRCD在不同预训练步骤下的性能。具体来说，我们按照输入序列的原始屏蔽比例15%，其中10%的屏蔽标记保持不变。就剩下的90%的掩蔽标记而言，我们将其分为四类。

- MacBERT：80%的代币被替换成它们的相似词，10%被替换成随机词。

- 随机替换：90%的代币被替换成随机词。

- 部分屏蔽：原始的BERT实现，80%的代币被替换成[MASK]代币，10%被替换成随机词。

- 全部屏蔽：90%的代币被替换为[MASK]代币。

我们只绘制了从1M到2M的步骤，以显示比前1M步骤更稳定的结果。结果在图2中得到了描述。那些主要依靠使用[MASK]进行遮蔽的预训练模型（即部分遮蔽和全部遮蔽）导致了更差的性能，表明预训练和微调的差异是一个影响整体性能的实际问题。其中，我们还注意到，如果不留10%的原始令牌（即身份投射），也会出现一致的下降，说明用[MASK]令牌进行遮蔽，在没有身份投射的情况下，负样本训练的稳健性较差，容易出现问题。

令我们惊讶的是，一个快速修复方法，即完全放弃[MASK]标记，将所有90%的遮蔽标记替换成随机词，产生了比依赖[MASK]的遮蔽策略更一致的改进。这也加强了这样的说法，即依靠[MASK]标记的原始遮蔽方法，在微调任务中从未出现过，将导致差异和更差的性能。为了使这个问题更加微妙，在本文中，我们提出使用类似的词来进行掩蔽，而不是从词汇中随机挑选一个词，因为随机的词将不符合上下文，并可能破坏语言模型学习的自然性，因为传统的N-gram语言模型是基于自然的句子，而不是一个被操纵的影响句。然而，如果我们使用类似的词来进行掩饰，句子的流畅性就会比使用随机词要好得多，整个任务就转化为一个语法纠正任务，这就更自然了，也不会出现预训练和微调阶段的差异。从图表中，我们可以看到，MacBERT在四个变体中产生了最好的性能，这验证了我们的假设。


7 结论 

在本文中，我们重新审视了中文的预训练语言模型，看看这些最先进的模型中的技术在英语以外的其他语言中是否具有良好的通用性。我们创建了中文预训练语言模型系列，并提出了一个名为MacBERT的新模型，它将掩蔽语言模型（MLM）任务修改为语言纠正方式，并缓解了预训练和微调阶段的差异。在各种中文NLP数据集上进行了广泛的实验，结果表明，所提出的MacBERT可以在大多数任务中获得明显的收益，详细的消融表明，应该更加关注MLM任务，而不是NSP任务及其变体，因为我们发现类似NSP的任务并没有显示出彼此之间的压倒性优势。随着中文预训练语言模型系列的发布，我们希望它将进一步加速中文研究界的自然语言处理。

在未来，我们希望研究一种有效的方法来确定屏蔽率，而不是启发式的屏蔽率，以进一步提高预训练语言模型的性能。
