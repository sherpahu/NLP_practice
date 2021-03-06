# 本资源由微信公众号：光明顶一号，分享
句法分析是自然语言处理中的关键技术之一，其基本任务是确定句子的句法结构或者句子中词汇之间的依存关系。主要包括两方面的内容，一是确定语言的语法体系，即对语言中合法句子的语法结构给予形式化的定义；另一方面是句法分析技术，即根据给定的语法体系，自动推导出句子的句法结构，分析句子所包含的句法单位和这些句法单位之间的关系。

依存关系本身是一个树结构，每一个词看成一个节点，依存关系就是一条有向边。本文主要通过清华大学的句法标注语料库，来实现基于 CRF 的中文句法依存分析模型。

### 清华大学句法标注语料库

清华大学的句法标注语料，包括训练集（train.conll）和开发集合文件（dev.conll）。训练集大小 5.41M，共185541条数据。测试集大小为
578kb，共19302条数据。

语料本身格式如下图所示：

![enter image description
here](https://images.gitbook.cn/ea8a9910-97fa-11e8-8841-e548fce76345)

通过上图，我们可以看出，每行语料包括有8个标签，分别是
ID、FROM、lEMMA、CPOSTAG、POSTAG、FEATS、HEAD、DEPREL。详细介绍如下图：

![enter image description
here](https://images.gitbook.cn/fa233670-97fa-11e8-b78f-09922e3c574f)

### 模型的实现

通过上面对句法依存关键技术的定义，我们明白了，句法依存的基本任务是确定句子的句法结构或者句子中词汇之间的依存关系。同时，我们也对此次模型实现的语料有了基本了解。

有了这些基础内容，我们便可以开始着手开发了。

本模型的实现过程，我们将主要分为训练集和测试集数据预处理、语料特征生成、模型训练及预测三大部分来实现，最终将通过模型预测得到正确的预测结果。

本次实战演练，我们选择以下模型和软件：

  * Sklearn_crfsuite
  * Python3.6
  * Jupyter Notebook

#### 训练集和测试集数据预处理

由于上述给定的语料，在模型中，我们不能直接使用，必须先经过预处理，把上述语料格式重新组织成具有词性、方向和距离的格式。

首先，我们通过一个 Python 脚本 `get_parser_train_test_input.py`，生成所需要的训练集和测试集，执行如下命令即可：

    
    
        cat train.conll | python get_parser_train_test_input.py > train.data 
        cat dev.conll | python get_parser_train_test_input.py > dev.data 
    

上面的脚本通过 cat 命令和管道符把内容传递给脚本进行处理。这里需要注意的是，脚本需要在 Linux 环境下执行，且语料和脚本应放在同一目录下。

`get_parser_train_test_input.py` 这一脚本的目的，就是重新组织语料，组织成可以使用 CRF
算法的格式，具有词性、方向和距离的格式。我们认为，如果词 A 依赖词 B，A 就是孩子，B
就是父亲。按照这种假设得到父亲节点的粗词性和详细词性，以及和依赖次之间的距离。

我们打开该脚本，看看它的代码，如下所示，重要的代码给出了注释。

    
    
        #coding=utf-8
        '''词A依赖词B，A就是孩子，B就是父亲'''
        import sys 
    
        sentence = ["Root"]
        def do_parse(sentence):
            if len(sentence) == 1:return 
            for line in sentence[1:]:
                line_arr = line.strip().split("\t")
                c_id = int(line_arr[0])
                f_id = int(line_arr[6])
                if f_id == 0:
                    print("\t".join(line_arr[2:5])+"\t" + "0_Root")
                    continue
                f_post,f_detail_post = sentence[f_id].strip().split("\t")[3:5] #得到父亲节点的粗词性和详细词性
                c_edge_post = f_post #默认是依赖词的粗粒度词性，但是名词除外；名词取细粒度词性
                if f_post == "n":
                    c_edge_post = f_detail_post
                #计算是第几个出现这种词行
                diff = f_id - c_id #确定要走几步
                step = 1 if f_id > c_id  else -1 #确定每一步方向
                same_post_num = 0 #中间每一步统计多少个一样的词性
                cmp_idx = 4 if f_post == "n" else 3  #根据是否是名词决定取的是粗or详细词性
                for i in range(0, abs(diff)):
                    idx = c_id + (i+1)*step
                    if sentence[idx].strip().split("\t")[cmp_idx] == c_edge_post:
                        same_post_num += step
    
                print("\t".join(line_arr[2:5])+"\t" + "%d_%s"%(same_post_num, c_edge_post))
            print("")
    
        for line in sys.stdin:
            line = line.strip()
            line_arr = line.split("\t")
            if  line == "" or line_arr[0] == "1":
                do_parse(sentence)
                sentence = ["Root"]
            if line =="":continue 
            sentence.append(line)
    

整个脚本按行读入，每行按 Tab 键分割，首先得到父亲节点的词性，然后根据词性是否是名词 n 进行判断，默认是依赖词的粗粒度词性，如果是名词取细粒度词性。

脚本处理完，数据集的格式如下：

![enter image description
here](https://images.gitbook.cn/6ceea0a0-984a-11e8-911c-dd974a01956f)

根据依存文法，决定两个词之间依存关系的主要有两个因素：方向和距离。正如上图中第四列类别标签所示，该列可以定义为以下形式：

> [+|-]dPOS

其中，`[+|-]` 表示中心词在句子中相对坐标轴的方向；POS 代表中心词具有的词性类别；d 表示与中心词词性相同的词的数量，即距离。

#### 语料特征生成

语料特征提取，主要采用 N-gram 模型来完成。这里我们使用 3-gram
完成提取，将词性与词语两两进行匹配，分别返回特征集合和标签集合，需要注意整个语料采用的是 UTF-8 编码格式。

整个编码过程中，我们首先需要引入需要的库，然后对语料进行读文件操作。语料采用 UTF-8 编码格式，以句子为单位，按 Tab 键作分割处理，从而实现句子
3-gram 模型的特征提取。具体实现如下。

    
    
        import sklearn_crfsuite
        from sklearn_crfsuite import metrics
        from sklearn.externals import joblib
    

首先引入需要用到的库，如上面代码所示。其目的是使用模型 `sklearn_crfsuite .CRF`，metrics 用来进行模型性能测试，joblib
用来保存和加载训练好的模型。

接着，定义包含特征处理方法的类，命名为 CorpusProcess，类结构定义如下：

    
    
    class CorpusProcess(object):
    
        def __init__(self):
            """初始化"""
            pass
    
        def read_corpus_from_file(self, file_path):
            """读取语料"""
            pass
    
        def write_corpus_to_file(self, data, file_path):
            """写语料"""
            pass
    
        def process_sentence(self,lines):
            """处理句子"""
            pass   
    
        def initialize(self):
            """语料初始化"""
            pass   
    
        def generator(self, train=True):
            """特征生成器"""
            pass  
    
        def extract_feature(self, sentences):
            """提取特征"""
            pass
    

下面介绍下 CorpusProcess 类中各个方法的具体实现。

第1步， 实现 init 构造函数，目的初始化预处理好的语料的路径：

    
    
        def __init__(self):
                """初始化"""
                self.train_process_path =  dir +  "data//train.data"   #预处理之后的训练集
                self.test_process_path =  dir +  "data//dev.data"  #预处理之后的测试集
    

这里的路径可以自定义，这里的语料之前已经完成了预处理过程。

第2-3步，`read_corpus_from_file` 方法和 `write_corpus_to_file` 方法，分别定义了语料文件的读和写操作：

    
    
        def read_corpus_from_file(self, file_path):
            """读取语料"""
            f = open(file_path, 'r',encoding='utf-8')
            lines = f.readlines()
            f.close()
            return lines
    
        def write_corpus_to_file(self, data, file_path):
            """写语料"""
            f = open(file_path, 'w')
            f.write(str(data))
            f.close()
    

这一步，主要用 open 函数来实现语料文件的读和写。

第4-5步，`process_sentence` 方法和 initialize 方法，用来处理句子和初始化语料，把语料按句子结构用 list
存储起来，存储到内存中：

    
    
        def process_sentence(self,lines):
                """处理句子"""
                sentence = []
                for line in lines:
                    if not line.strip():
                        yield sentence
                        sentence = []
                    else:
                        lines = line.strip().split(u'\t')
                        result = [line for line in lines]
                        sentence.append(result)   
    
            def initialize(self):
                """语料初始化"""
                train_lines = self.read_corpus_from_file(self.train_process_path)
                test_lines = self.read_corpus_from_file(self.test_process_path)
                self.train_sentences = [sentence for sentence in self.process_sentence(train_lines)]
                self.test_sentences = [sentence for sentence in self.process_sentence(test_lines)] 
    

这一步，通过 `process_sentence` 把句子收尾的空格去掉，然后通过 initialize 函数调用上面
`read_corpus_from_file` 方法读取语料，分别加载训练集和测试集。

第6步，特征生成器，分别用来指定生成训练集或者测试集的特征集：

    
    
        def generator(self, train=True):
            """特征生成器"""
            if train: 
                sentences = self.train_sentences
            else: 
                sentences = self.test_sentences
            return self.extract_feature(sentences)    
    

这一步，对训练集和测试集分别处理，如果参数 train 为 True，则表示处理训练集，如果是 False，则表示处理测试集。

第7步，特征提取，简单的进行 `3-gram` 的抽取，将词性与词语两两进行匹配，分别返回特征集合和标签集合：

    
    
        def extract_feature(self, sentences):
                """提取特征"""
                features, tags = [], []
                for index in range(len(sentences)):
                    feature_list, tag_list = [], []
                    for i in range(len(sentences[index])):
                        feature = {"w0": sentences[index][i][0],
                                   "p0": sentences[index][i][1],
                                   "w-1": sentences[index][i-1][0] if i != 0 else "BOS",
                                   "w+1": sentences[index][i+1][0] if i != len(sentences[index])-1 else "EOS",
                                   "p-1": sentences[index][i-1][1] if i != 0 else "un",
                                   "p+1": sentences[index][i+1][1] if i != len(sentences[index])-1 else "un"}
                        feature["w-1:w0"] = feature["w-1"]+feature["w0"]
                        feature["w0:w+1"] = feature["w0"]+feature["w+1"]
                        feature["p-1:p0"] = feature["p-1"]+feature["p0"]
                        feature["p0:p+1"] = feature["p0"]+feature["p+1"]
                        feature["p-1:w0"] = feature["p-1"]+feature["w0"]
                        feature["w0:p+1"] = feature["w0"]+feature["p+1"]
                        feature_list.append(feature)
                        tag_list.append(sentences[index][i][-1])
                    features.append(feature_list)
                    tags.append(tag_list)
                return features, tags    
    

经过第6步，确定处理的是训练集还是测试集之后，通过 `extract_feature` 对句子进行特征抽取，使用 3-gram
模型，得到特征集合和标签集合的对应关系。

#### 模型训练及预测

在完成特征工程和特征提取之后，接下来，我们要进行模型训练和预测，要预定义模型需要的一些参数，并初始化模型对象，进而完成模型训练和预测，以及模型的保存与加载。

首先，我们定义模型 ModelParser 类，进行初始化参数、模型初始化，以及模型训练、预测、保存和加载，类的结构定义如下：

    
    
        class ModelParser(object):
    
            def __init__(self):
                """初始化参数"""
                pass
    
            def initialize_model(self):
                """模型初始化"""
                pass
    
            def train(self):
                """训练"""
                pass
    
            def predict(self, sentences):
                """模型预测"""
                pass
    
            def load_model(self, name='model'):
                """加载模型 """
                pass
    
            def save_model(self, name='model'):
                """保存模型"""
                pass
    

接下来，我们分析 ModelParser 类中方法的具体实现。

第1步，init 方法实现算法模型参数和语料预处理 CorpusProcess 类的实例化和初始化：

    
    
        def __init__(self):
                """初始化参数"""
                self.algorithm = "lbfgs"
                self.c1 = 0.1
                self.c2 = 0.1
                self.max_iterations = 100
                self.model_path = "model.pkl"
                self.corpus = CorpusProcess()  #初始化CorpusProcess类
                self.corpus.initialize()  #语料预处理
                self.model = None
    

这一步，init 方法初始化参数以及 CRF 模型的参数，算法选用 LBFGS，c1 和 c2
分别为0.1，最大迭代次数100次。然后定义模型保存的文件名称，以及完成对 CorpusProcess 类 的初始化。

第2-3步，`initialize_model` 方法和 train 实现模型定义和训练：

    
    
         def initialize_model(self):
                """模型初始化"""
                algorithm = self.algorithm
                c1 = float(self.c1)
                c2 = float(self.c2)
                max_iterations = int(self.max_iterations)
                self.model = sklearn_crfsuite.CRF(algorithm=algorithm, c1=c1, c2=c2,
                                                  max_iterations=max_iterations, all_possible_transitions=True)
    
            def train(self):
                """训练"""
                self.initialize_model()
                x_train, y_train = self.corpus.generator()
                self.model.fit(x_train, y_train)
                labels = list(self.model.classes_)
                x_test, y_test = self.corpus.generator(train=False)
                y_predict = self.model.predict(x_test)
                metrics.flat_f1_score(y_test, y_predict, average='weighted', labels=labels)
                sorted_labels = sorted(labels, key=lambda name: (name[1:], name[0]))
                print(metrics.flat_classification_report(y_test, y_predict, labels=sorted_labels, digits=3))
                self.save_model()
    

这一步，`initialize_model` 方法实现 了 `sklearn_crfsuite.CRF` 模型的初始化。然后在 train 方法中，先通过
fit 方法训练模型，再通过 `metrics.flat_f1_score` 对测试集进行 F1 性能测试，最后将模型保存。

第4-6步，分别实现模型预测、保存和加载方法，具体代码大家可以访问
[Github](https://github.com/sujeek/chinese_nlp)。

最后，实例化类，并进行模型训练：

    
    
        model = ModelParser()
        model.train()
    

对模型进行预测，预测数据输入格式为三维，表示完整的一句话：

> [[['坚决', 'a', 'ad', '1_v'],

>  
>  
>      ['惩治', 'v', 'v', '0_Root'],

>  
>      ['贪污', 'v', 'v', '1_v'],

>  
>      ['贿赂', 'n', 'n', '-1_v'],

>  
>      ['等', 'u', 'udeng', '-1_v'],

>  
>      ['经济', 'n', 'n', '1_v'],

>  
>      ['犯罪', 'v', 'vn', '-2_v']]]

>  

模型预测的结果如下图所示：

![enter image description
here](https://images.gitbook.cn/87377f80-985a-11e8-b78f-09922e3c574f)

预测的结果，和原始语料预处理得到的标签格式保持一致。

语料和代码下载，请访问：[Github](https://github.com/sujeek/chinese_nlp)。

### 总结

本文通过清华大学的句法标注语料库，实现了基于 CRF 的中文句法依存分析模型。借此实例，相信大家对句法依存已有了一个完整客观的认识。

**参考文献及推荐阅读**

  1. [使用 CoNLL 2002 数据的英文句法依存分析](https://github.com/scrapinghub/python-crfsuite/blob/master/examples/CoNLL%202002.ipynb)
  2. [CRF++ 依存句法分析](http://x-algo.cn/index.php/2016/03/02/crf-dependency-parsing/)
  3. [依存分析：基于序列标注的中文依存句法分析模型实现](https://blog.csdn.net/sinat_33741547/article/details/79321401)


## 更多资源下载交流请加微信：Morstrong
# 本资源由微信公众号：光明顶一号，分享,一个用技术共享精品付费资源的公众号！