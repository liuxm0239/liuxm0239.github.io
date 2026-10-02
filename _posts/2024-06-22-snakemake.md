---
layout: post
title: 
date: 2024-06-22 11:16:52 +0800
categories: ['基因组学','操作系统强迫症']
tag: ['Bioinfo', 'Linux', 'python' ]
---
* content
  {:toc}

---


Snakemake 搭建流程

# 介绍

对于生信分析，大部分人的第一印象就是流程化。确实，在依托于强大的计算机系统，许多常规的分析任务都可以自动化运行并进行相应的分析。例如对于测序数据来说，常见的上游分析便是数据质控、去接头、序列比对、 PCR 去重矫正、基因表达定量、突变和拷贝数分析等。这些分析步骤中，每一步都有很多不同的软件和算法，选择多样化。而对于数据分析者来说，这些上游的步骤并不是他们所关注的，他们可能更注重一些下游的个性化分析，所以前面的数据处理和分析工作完全可以让流程自动完成。
所谓流程，便是能够将分析步骤进行串联，多样本并行计算。常见的流程化一般使用 shell脚本，将所有分析任务写入到一个脚本中，这种方法扩展性较差，程序臃肿。而 Python 是一门非常灵活的语言，非常适合用来编写模块化的流程，但比较麻烦的一点是，需要自己进行一些命令的封装。幸好，已经有人帮我们做好了。


Snakemake 是一种基于 Python 的工作流管理系统，能够用于创建可重复和可扩展的数据分析的工具。其编写的流程可以无缝部署到服务器、集群等环境，而无需对流程进行修改。依托于 conda 包管理系统，不需要我们手动安装依赖软件，可以做到只修改配置文件便能运行整个流程。会自动检查文件时间戳，能够从上次中断的位置继续往下跑，而不用重复执行。

# 安装

安装 snakemake 最好的方式是使用包管理工作conda或mamba，mamba是 conda 的C++ 实现版，使用多线程下载缓存，更快的解决包依赖关系，极大提供包搜索速度，其使用方式与 conda 一样。

## 安装 mamba

conda install -n base -c conda-forge mamba
将 snakemake 安装到虚拟环境
mamba create -c conda-forge -c bioconda -n snakemake snakemake
进入环境，查看是否安装成功
conda activate snakemake
snakemake --help

# 定义流程

受 Make 的启发，snakemake 使用 rule 字段来定义一个从输入文件到输出文件的规则而不同规则之间的依赖关系是通过输入和输出文件名称进行匹配，隐式地进行处理。一个rule 可以看作是一个操作步骤，比如序列比对、表达定量等调用特定软件的步骤就可以定义一个 rule 来处理。

1. 基本语法
   snakemake 使用冒号和缩进来定义规则，并用一些关键字来进行规则声明，同时也可以兼容Python 代码。
   在rule 声明的规则中，有很多关键字用于定义不同的规则和文件信息，例如
2. 定义 rule一个 rule 定义了流程中的一项数据分析步骤，使用关键字 rule 声明，包括规则名称，输入文件、输出文件以及输入文件映射到输出文件的 shell 命令或一个处理脚本。例如

    rule NAME:
        input: 
            "path/to/inputfile1", 
            "path/to/inputfile2"
        output:
            ["path/to/outputfile1", "path/to/outputfile2"]
        shell:
            "command {input} {output}"

    规则的名称是可选的，可以省略，表示一个匿名规则。也可以通过设置规则的 name 属性来
    覆盖。
    其中 input 和 output 可以是元组或者列表，最后一行的 shell 表示需要执行的 shell命令，其中花括号表示引用，即使用 input 和 output 关键字中定义的值来进行字符串替换。可以使用双花括号来取消这种占位符替换，会被转换为单花括号字符。在这里输入和输出列表会被转换为空格分隔的字符串(类似于·'.join(input))。
    也可以使用字典的方式定义键值对，例如
    rule NAME:
        input:
            fastq1k= "path/to/inputfile"
            fastq2 ="path/to/other/inputfile"
        output:
            "path/to/outputfile",
            somename ="path/to/another/outputfile"
        shell:
            "command {input.fastq1} {input.fastq2} {output[0]}'
    在花括号内使用点加属性名称或使用数组索引的方式访问
    当要同时运行多个 shell 命令或编写 Python 代码时，可以使用 run
    rule NAME: 
        input:
            "path/to/inputfile".
            "path/to/other/inputfile"
        output:
            "path/to/outputfile",
            somename ="path/to/another/outputfile"
        run:
            for f in input:
            with open(output[0], "w") as out:
                out.write(...)with open(output.somename, "w") as out.out.write(...)
            shell('cp {input[1]} {output.somename}')
在 run 字段中，可以使用列表或属性获取的方式来访问input和 output 中定义的值。
### 通配符
有时候，我们的输入或输出文件并不是固定的名称，比如我们分析一批数据不可能只有一个样本，而且每次分析样本的名称肯定不一样，因此将输入或输出指定为一种可变的，或者可替换的字符是很有必要的，这也是 snakemake 的一个优势所在。例如下面这个例子
    rule complex_conversion:
        input:
            "input/{sample}_file"
        output:
            "output/{sample}/file.{group}.txt"
        shell:
            "somecommand -group {wildcards.group}<{input}> {output}"
在这里，我们定义了两个通配符 sample 和 group，这两个通配符相当于正则表达式中的.+，可以匹配除空字符外的任意字符串。
在 shell 字段中，可以使用wildcards 对象来访问对应的通配符，该对象是一个内置对象，所有通配符均为其属性。
但有时，通配符可能会导致歧义，例如，对于通配符{sample}.{group}.txt，从字符串A.R.1.txt 中无法确定 sample 是 A.R 还是 A。这时，我们需要添加一些约束条件，可以有三种方式来添加
- 在通配符后面使用逗号并添加限定的正则表达式
    output:"{sample}.{group,\d+}.txt"
- 使用 wildcard_constraints 关键字
    rule complex_conversion:
        input:
            "input/{sample}_file"
        output:
            "output/{sample}/file.{group}.txt"
        wildcard_constraints:
            group ="\d+"
        shell:
            "somecommand --group {wildcards.group} < {input}> {output}"
- 定义全局的通配符约束
    wildcard_constraints:
        group="\d+"

    rule a:
        ...
    rule b:
        ...
那这些通配符的值是怎么推断出来的呢?
一种方式是在终端运行文件时指定输出文件的路径，会通过文件路径进行推断
    snakemake -s Snakefile output/A/file.Rl.txt
其中 rule 规则定义在 Snakefile 中。也可以在 rule all 中的 input 中定义
    rule all:
        input:
            "output/A/file.R1.txt"
all 这个规则名称时固定的，而且只要在 input 中定义需要执行的输出文件， snakemake会自动运行相应的规则，及其依赖的规则。也就是说如果该文件的输出需要其他规则，那些规则也会自动执行，其最终的目的就是必须获取到 input 里面定义的文件。
snakemake 默认只会执行文件中第一个 rule ，可能有些规则虽然定义了，但是没有指定输出文件，也不会被执行。可以设置在rule 中设置 default_target:True 将其标注为默认执行的规则，但我们一般还是会在input 中进行指定

### 聚合
输入和输出文件可以是 Python 列表，我们可以使用列表推导式来定义输入文件。例如
    rule aggregate:
        input:
            ["{dataset}/a.txt",format(dataset=dataset) for dataset inDATASETS]
        output:
            "aggregated.txt"

        shell:
            ...

其中， dataset 不会被认为是通配符，而是和 Python 中的格式化字符串的占位符
snakemake 提供两种简洁的方式来实现同样的功能
### expand 函数
该函数类似于 Python 字符串的 format 方法，使用关键字的方式指定需要替换的值
    rule aggregate:
        input:
            expand("{dataset}/a.txt", dataset=DATASETS)
        output:
            "aggregated.txt"
        shell:
            ...
该函数提供的功能也比较丰富，可以同时替换多个字符串。例如
    expand(["{dataset}/a.{ext}","{dataset}/b.{ext}"],dataset=['GE0','KEGG'],ext=['txt','csv'])
代表的文件列表为
    ["GE0/a.txt","GE0/b.txt","KEGG/a.txt","KEGG/b.txt""GE0/a.csv"，"GE0/b.csv"，"KEGG/a.csv"，"KEGG/b.csv"]
该函数默认使用 Python 中 itertools 包中的 product 函数对 dataset 和 ext 取笛卡尔积，然后将数据填充进去，我们也可以使用其他组合函数或自定义函数。例如，使用 zip函数
    expand(["{dataset}/a.{ext}","{dataset}/b.{ext}"],
      zip,dataset=['GE0','KEGG'],ext=['txt','csv'])
将会得到
    ["GE0/a.txt","GE0/b.txt","KEGG/a.csv"，"KEGG/b.csv"]
### multiext 函数
对于名称相同但是扩展名不同的文件列表，可以使用 multiext 函数，它是 expand 函数的变体，例如
    rule plot:
    input:
        ...
    output:
        multiext("some/plot", ".pdf", ".svg", ".png")
    shell:
        ...
第一个参数定义文件名，后面的参数为不同的文件后缀。上面的代码相当于
    expand("some/plot.{ext}", ext=[".pdf", ".svg", ".png"])

### 线程
线程数通过 threads 关键字指定，指定程序使用的最大线程数，该值不会大于总的可用线程
    rule NAME:
        input:
            "path/to/inputfile"，
            "path/to/other/inputfile"
        output:
            "path/to/outputfile", 
            "path/to/another/outputfile"
        threads: 8
        shell:
            "somecommand --threads {threads} {input} {output}"
### 资源
可以使用 resources 关键字来指定程序使用的资源，例如内存资源、磁盘空间、gpu 等。调度程序会确保运行任务不会超出给定资源数，注意资源数是被指定为每个作业的总数，而不是每个线程的资源数。
    rule a:
        input:
        output: ...
        resources:
            mem_mb=100,
            nvidia_gpu=1
        shell:
            ...
### 消息

snakemake 运行流程时，控制台会输出每个规则的简短摘要信息，我们可以使用 message参数来为每个规则指定运行时的输出信息
    ruLe NAME:
        input:
            "path/to/inputfile",
            "path/to/other/inputfile"
        output: 
            "path/to/outputfile", 
            "path/to/another/outputfile"
        threads:8
        message: 
            "Executing somecommand with {threads} threads on thefollowing files {input}."
        shell:
            "somecommand --threads {threads} {input} {output}"
在 message 中也可以像 shell 一样，使用 wildcards 对象访问通配符
### 优先级
可以使用 priority 关键字为规则添加数字优先级，默认情况下，每个规则的优先级都为
8，数值越大优先级越高
    rule :
        input: ...
        output: ...
        priority: 50
        shell: ...
### 日志
每个规则也可以使用 log 关键字定义日志文件
    rule abc:
        input: "input.txt"
        output: "output.txt"
        log: "logs/abc.log"
        shell:"somecommand -log {log} {input} {output}"
其与 output 类似，日志文件可以使用通配符，也支持文件列表形式。
    rule abc:
        input: "input.txt"
        output: "output.txt"
        log:log1="logs/{sample}/abc.log",log2="logs/xyz.log'
        shell:"somecommand -log {log.Log1} METRICS_FILE-{log.log2} {input}{output}"
不同点在于当程序运行出错时，输出文件会被删除，而 1og 文件将会保留，有助于排查程序问题

### 参数
input 和 output 都是指定文件，但是我们所使用的程序一般不仅仅只有输入和输出参数,还有其他控制参数，这些参数是无法通过这两个参数指定的。此时，可以使用 params 关键
    rule:
        input:
        params:
            prefix="somedir/{sample}"
        output:
            "somedir/{sample}.csv'
        shell:
            "somecommand -o {params.prefix}"
参数值不仅可以是字符串，还可以使用函数获取值
    prefix=lambda wildcards, output: output[e][:-4]
函数的第一个参数必须为 wildcards，还可以接受四个参数 input、 output、threads和 resources ，这几个参数的位置可以任意顺序。如果需要将通配符作为参数传入到运行的脚本中，需要使用函数的形式，例如
    rule annotatePeaks:
        input:
            peakfile = "macs2/narrow/{sample}_peaks.narrowPeak",gtf = config['data']['gtf']
        output:
            expand('macs2/anno/{{sample}}.peakAnno.{ext}', ext=['pdf', 'txt'])
        params :
            sample = lambda wildcards:{}'.format(wildcards.sample),
            output = config['workdir']+"/macs2/anno/"
        script:
            'annoPeaks.R'
其中 script 关键字用于指定需要运行的脚本

### 外部脚本
规则中定义的命令，不仅可以是 shell 命令或内联的 Python 代码，也可以指定外部脚本。支持 Python、R、Julia、Rust 和 Bash 脚本，使用 script 关键字指定。
这些脚本在运行时都会传入一个 snakemake 对象(Bash 中传入的是一些关联数组，没有对象)，该对象包含属性:input、output、params、wildcards、log、 threads、resources、config。可以在脚本中进行访问
在 Python 中，使用 snakemake 对象
    def do_something(data_path, out_path, threads, myparam):
        # python code
    do_something(snakemake.input[0],snakemake.output[0],
            snakemake.threads, snakemake.config["myparam"])
在R和R Markdown 中，是一个名为snakemake 的 S4 对象

    do_something<function(data_path, out_path, threads, myparam){
        #R code
    }

    do_something(snakemake@input[[1]],snakemakeQoutput[[1]],
            snakemakeGthreads,snakemake@config[["myparam"]])

Julia 和 Rust 中也有与 Python 类似的 snakemake 对象，而在 Bash 中就不太一样。d于没有面向对象，所以提供了名为 snakemake_<directive>的数组，其中 directive 可以是:input、output、log、wildkards、resources、params、config 。没有包含的关键词使用 snakemake 访问。例如
    rule align:
        input:
            "{sample}.fq",
            reference="ref.fa",
        output:
            "{sample}.sam"
        params:
            opts="-a-x map-ont",
        threads: 4
        Log:
            "align/{sample}.log"
        conda:
            "envs/align.yaml"
        script:
            "scripts/align.sh"
如果要在 Bash 脚本中访问命名的属性，需要直接将属性名称作为索引来访问
    #!/usr/bin/env bash
    echo "Aligning sample ${snakemake_wildcards[sample]} with minimap2" 2>"${snakemake_log[0]}"
    minimap2 ${snakemake_params[opts]}-t ${snakemake[threads]}"${snakemake_input[reference]}" \
    "$isnakemake_input[0]}">"${snakemake_output[0]}" 2>>"${snakemake_log[e]}"
其中 snakemake_wildcards[sample]访问的是名为 sample 的通配符，snakemake[threads]访问的数线程数，snakemake_params[opts]访问的是 params内的 opts 属性的值。

当输入文件是一个列表时，由于 Bash 中并没有嵌套数组，所以需要将其转换为空格分隔的字符串，再转换为数组。例如  
    rule align:
        input:
            reads=["{sample}_R1.fq","{sample}_R2.fq]"],
            reference="ref.fa",
        output:
            reads=(${snakemake_input[reads]})#不要用双引号
            r1="${reads[o]}"
            r2="${reads[1]}"
保护文件和临时文件
一些比较耗时且大型的文件，我们可能想保护它免受意外删除或覆盖，可以使用 protected
    rule NAME:
        input:
            "path/to/inputfile"
        output:
            protected("path/to/outputfile")
        shell:
            "somecommand {input} {output}"
        或者有些临时文件，希望执行完程序之后就立马删除，释放空间，可以使用temp
        rule NAME:
            input:
                "path/to/inputfile"
            output:
                temp("path/to/outputfile")
            shell:
                "somecommand {input} {output}"
### 输出目录
有时候也可以将目录作为输出，但是最好加上 directory 可以避免一些删除操作将重要文件删了。而且一些对目录下文件的修改会更新文件夹的时间戳，snakemake 会对每个规则的输入和输出文件的时间戳进行比较，从而判断是否需要重新执行该规则
    rule NAME:
        input:
            "path/to/inputfile"
        output:
            directory("path/to/outputdirn")
        shell:
            "somecomand {input} {output}"
        使用 ancient 函数可以忽略时间戳，始终认为该文件早于所有输出文件
    rule NAME:
        input:
            ancient("path/to/inputfile")
        output:
            "path/to/outputfile"
        shell:
            "somecommand {input} {output}"
输出文件将仅被创建一次，重复运行该规则不会再创建，即便输入文件后面也被修改了，只有强制执行回重新创建该文件。

### 文件检查
使用 ensure 函数可以对文件是否为空或校验码进行检查
    rule NAME:
        output:
            ensure("test.txt",non_empty=True)
            ensure("test.txt"，sha256="u98a9cjsd98saud090923Bkpoask6f9B32")
        shell:
            "somecommand {output}"
校验码也可以是一个函数
    def get_checksum(wildcards):
        return my_checksums[wildcards.sample]

    rule NAME:
        output:
            ensure("test/{sample}.txt",sha256 get_checksum)
        shell:
            "somecommand {output}"

### 重试
有些时候，某些规则并不是总能成功运行，有一定的失败概率。比如，进行网络访问或下载时可能会出现网络波动或故障。对于这种情况，可以通过retries 关键字为该规则中的每个作业定义重试的次数
    rule a:
        output :
            "test.txt"
        retries: 3
        shell:
            "curl https://some.unreliable.server/test.txt > {output}"
将函数作为输入
输入文件可以是单个字符串或字符串列表的形式，还可以使用返回值是一个或多个输入文件的函数，例如，使用通配符获取输入文件

    def myfunc(wildcards):
        return【... a list of input files depending on given wildcards ...]
  
    rule:
        input :
            myfunc
        output:
            "someoutput.{somewildcard}.txt"
        shell:
该函数必须以通配符对象作为第一个参数。
如果雨数的返回值是一个字典，则需要在雨数的外面嵌套一个 unpack 兩数，其作用相当于用键值对定义了输入文件
    def myfunc(wildcards):
        return {'foo':'{wildcards.token}.txt'.format(wildcards=wildcards)}
    
    rule:
        input:
            unpack(myfunc)
        output:
            "someoutput.{token}.txt
        shell:
            "... {input.foo}”
    如果不是直接使用函数名，而是以调用雨数的方式，则需要使用 Python 的解包操作
    def myfunc1():
        return ['foo.txt']

    def myfunc2():
        return {'foo': 'nowildcards.txt'}

    rule:
        input :
            *myfunc1(),
            **myfunc2(),
        output:
            ...
        shell:
            ...
### 规则依赖
在我们编写项目时，基本上都会涉及到前一个规则的输出文件作为后一个规则的输入文件，如果我们只是将文件名拷贝一份，那么当需要修改文件名时就会变得非常麻烦。我们可以使用 rules 对象来引用其他规则中的值，例如
rule a:
input:"patn/to/input
output:a="path/to/output",b="path/to/output2”
shell:
rule b:
input: rules.a.output.a
output:"path/to/output/of/b"
shell:a
规则 b 中，直接引用了规则 a 的输出文件，修改 a 中的输出文件名，并不会影响其他引用到它的规则。
本地任务
在集群环境中，并非所有规则都需要提交到集群中进行计算，比如一些下栽规则， a11 规则等一些不需要密集计算的任务。可以使用关键字 localrules 将规则标记为本地，这样它就不会提交到集群而是在主机节点上执行
localrules: all, foo
rule all:
input:
rule foo:
rule bar:

上面的规则中只有 bar 会提交到集群上。注意，localrules 指令可以多次使用，会自动将所有结果合并
### 拆分与合并
当有些文件较大时，我们可以将其拆分为较小的部分，然后分部分计算，最后将计算后的结果进行合并。这种操作一般在序列比对时较常用到，当比对的 fastq 数据过大时，可以将其拆分为 10-20 个小的 fastq，然后进行并行比对，加快比对的速度。这时，我们可以使用scattergather 定义全局的拆分，例如
    scattergather:
        split=8
然后，在 scatter 和 gather 对象中会有一个名为 split 这个拆分和聚合方法。这个两个对象中调用的方法名称与 scattergather 中定义的拆分名称一样
    rule all:
        input:
        "gathered/all.txt"

        rule solit:
        input:
        "content.txt"
        output:
        scatter.split("splitted/{scatteritem}.txt")#拆分params:
        chunks=8shell:
        "split -n {params.chunks}-numeric-suffixes=1 --suffixlength=1-additional-suffix=-of-8,txt {input} splitted/"
        rule intermediate:
        input:
        "splitted/{scatteritem}.txt"
        output:
        "splitted/{scatteritem}.post.txt"
        shell:
        "cp finput} {output}"

        rule gather:
        input:
        gather.split("splitted/{scatteritem} .post.txt")output:
        "gathered/all.txt"
        shell:
        "cat {input} > {output}"
其中 scatteritem 会变成 i-of-n，其中 n 就是前面定义的 split 拆分的数量，1 为 1-n 之间的值。所以需要保证拆分出来的文件名称正确

### 规则继浜
可以从先前定义的规则继承，以修改的方式重用现有规则。例如
    rule a:
        output:
            "test.out"
        shell:
            "echo test > {output}
    use rule a as b with:
        output:
            "test2.out"
使用 se 语句，规则 b承自 a，并使用 with 语句修改其中的输出文件为test2.out 。除了规则中的执行命令或脚本，其他项的值都是可修改的。

## 3. 配置文件
Snakemake 允许使用配置文件来定义一些用户自定义变量的值，如样本信息、数据路径、软件参数和一些流程控制参数等，可以使工作流程更加灵活。
### 标准配置文件
snakemake 支持 JSON 和 YAML 格式的配置文件，可以使用 configfile 关键字来声明配置文件
    configfile:"path/to/config.yaml
在流程中处于该声明之后的地方，都可以使用 config 全局字典变量对配置信息进行访问。如果要在 shell 命令添加占位符，需要将健名两侧的引号去掉。例如
    parals:
        path = config['path']
    shell:
        "mycommand {config[foo]}
如果未使用 configfile 语句声明，则 config 变是将是一个空数组。除了 configfile语句之外，还可以通过命令行覆盖配置项的值，例如:
snakemake--config yourparam=1.5
或者使用 --configfile 命令行参数指定配置文件。这三种方式设置的配置项会汇总在一起，如果存在相同的配置项，则命令行的值优先。
表格配置文件
当然并不是所有信息都适合用 YAHL或 JS0N 来表示，像样本信息这种通常以表格数据的形式展现会比较好，可以使用 pandas 来直接读取文件中的数据
    import pandas as pd
    samples = pd.read_table(config["samples_info"]).set_index("samples",drop=False)
这可以作为配置文件的补充，从配置文件中指定的样本信息文件中读取样本的所有信息，免去在配置文件中进行指定。

### 验证
我们编写的流程时，应该只将配置文件作为接口，通过修改配置文件信息就可以运行流程，因此，对配置文件的格式进行验证，是保证流程顺利运行的关键，snakemake 提供了nakemake.utils.validate 函数可以对配置文件(也可以是 pandas 的数据框)进行验证。其验证方式通过 JSON schemas 指定，例如
    import pandas as pd
    from snakemake.utils import validate
    configfile: "config.yaml"
    validate(config,"config.schema.yam")
    samples = pd,read_table(config["samples"]).set index("sample"drop=False)
    validate(samples,"samples.schema.yaml")

    rule all:
        input:
            expand("test.{sample}.txt",sample=samples.index)
   
    rule a:
        output:
            "test.{sample}.txt"
            76
        shell:
            "touch {output}"
分别对配置文件和数据框的格式进行验证，配置文件需要根其中包含的字段设置验证，例如配置文件 config.schema.yamL中有GENOME、GENES 和FASTA 三个字段， GENOME 单面又包含两个字段，里面的字段也需要添加一个properties 去验证
    $schema:"http://json-schema.org/draft-86/schema#
    description: config file schema
    properties:
        GENOME:
            type: object
        description: path to a bowtie2 index
        properties:
            name :
                type: string
            path:
                type: string
        GENES:
            type: string
            description: path to a BED6 file with gene name as the 4th column
        FASTA:
            type: string
            description: path to a FAsTQ file corresponding to the GENOME
    required:
        - GENOME
        - GENES
        - FASTA
数据框的验证内容需要对每一列进行验证，例如saples.schema.vaml
    $schema: "https://json-scheaa.org/draft-86/schema#"
    description: an entry in the sample sheet
    properties:
        sample:
            type: string
            description: sample name/identifier
        condition;
            type: string
        description:sample condition that will be compared duringdifferential expression analysis (e.g. a treatment, a tissue time, adisease)
case:
type: boolean
default: true

description: boolean that indicates if sample is case or control

required:
- sample
-condition
### 环境变量
有些配置信息并不适合写入到文本文件中，例如账号密码和 token 等，我们可以将这些信息先写入到系统的环境变量中，然后在流程中使用
envvars:
"SOME_VARIABLE"
rule do_something:
output:
"test.txt"
parans:
x=os.environ["SOME_VARIABLE"]
shell:
"echo iparams.x}> {output}"

snakemake 文件中所指定的所有路径相对路径都是相对于当前运行目录而言的，可以通过在文件中使用关键字 workdir 来指定工作目录

workdir:"path/to/workdir"
工作路径也可以使用配置文件指定，然后在此处设置为配置文件中所设置的配置项
## 4.模块化

snakemake模块化主要分为 4个层级
1. 最底层的封装是包装器，将实际运行的代码进行封装，只暴露出输入输出接口。并通过与 conda结合，将软件的依赖性部署到一个隔离的环境。这些封装器可以发布到Snakemake Wrapper Repository 或直接使用上面写好的封装器
2. 将功能细化，执行同一功能的规则写在一个 Snakefile 里面，并在主要的那个Snakefile 里面使用 include 将该功能导入，所有的 Snakefile 共享同一个配置文件
3. 通过 mudule 语句可以任意组合和重用规则
4. 定义子流程，但是不推荐
#### 封装器
使用 wrapper 关键字声明封装器，例如
    rule samtools_sort:
        input:
            "mapped/{sample}.bam"
        output:
            "mapped/{sample}.sorted.bam"
        parans :
            #-m GG*
        threads: 8
        wrapper:
            "0.0.8/bio/samtools/sort"
上面使用了 Snakemake Wrapper Rcpository 中定义的对 bam 文件排序的封装器，其封装了samtools sort 程序的功能，会自动解析传入的输入输出文件，以及额外的参数等，解析后执行相应的排序指令。
其中 0.0.8 表示的是封装的版本，上面封装了一些常用的生信分析软件，为了省事可以直接使用上面的封装器而不需要自己再重写相应的命令或脚本。
当然，我们也可以提供完整的 URL 或本地路径来添加自定义或其他人提供的封装器。如果是本地的封装器，可以使用绝对路径(file://path/to/wrapper)和相对路径两种方式( file:path/to/wrapper )
在线的则需要提供完整的文件URL，例如使用上传到 GitHub 上的封装器，注意是raw 版本
    wrapper:
        https://github.com/snakemake/snakemake
        wrappers/raw/8.8.8/bio/samtools/sort
封装器的文件结构一般是，有三个文件是必须的environment.yaml、meta.yaml 和wrapper.py .
    |--environment.yaml
    |-- meta.yaml
    |-- test
    |-- a.bam
    |  |-- a.bam,bai
    |  |-- Snakefile
    |-- wrapper.py
其中 environment.yaml 定义了该封装所依赖的环境，涉及使用到的包及软件， conda 将根据该文件搭建环境，其内容主要包括
    channels:
        -conda-forge
        - bioconda
        - nodefaults
    dependencies:
        - samtools =1.14
meta.yaml 是关于该封装的一些基本信息的描述。 wrapper.py 是该封装的具体处理代码，该文件不仅仅可以是 Python 脚本，也可以是其他编程语言(R、Julia、 Rust 和Bash )编写的脚本。

在一个 Snakefile 文件中导入另一个 $nakefile 中定义的规则，使用 include 关键字会一次将所有规则导入
include: "path/to/other/snakefile'

## 模块
使用 module 关键字可以将其他流程(可以是本地文件也可以是 URL)定义为一个模块
    from snakemake.utils import min_version
    module other_workflow:
    snakefile:
    "other workflow/snakefile"
    use rule * from other_workflow exclude ruleC as other_*
其中 use rule * from other_workflow exclude rulec 表示导入模块中除了 rulec 之外的所有规则， as other_*表示将这些导入的规则都添加了一个 other_ 前缀，可以防止规则重名造成的冲突。其中 *表示的是所有，如果只要导入某个规则，可以将其替换成对应的规则名称。
我们也可以为模块指定配置参数，覆盖模块中原先指定的配置文件，同时可以在module中跳过参数的检查，也可以使用 with 修改规则
    from snakemake.utils import min_version
    configfile: "config/config.yaml*
    module other_workflow:
    snakefile:"other_workflow/Snakefile"config: config["other-workflow"]
    skip_validation: True
    use rule * from other_workflow as other_*

    use rule some_task from other_workflow as other_some_task with:output:
    "results/some-result.txt"

## 5.状态通知
我们可以在流程运行的不同状态下，进行不同的操作，例如在启动分析时，打印消息，在分析成功或失败是输出提示，可以用到三个关键字
    onstart:
    print("workflow start")
    onsuccess:
    print("Workflow finished, no error")
    onerror:
    print("An error occurred")

    shell("mail-s "an error occurred")
# 执行流程
在这里，我们主要介绍一些如何运行前面定义的流程，snakemake 是通过命令行来运行，可以运行在本地或集群上
## 命令行
### 1.命令行参数
定义流程使用的核心数
    snakemake -cores 1
Snakemake 默认会执行在同一日录中名为 Snakefile 的文件所定义的工作流，也可以通过参数-s 绐出 Snakefile 。
试运行流程
    snakemake -n
此外，可以打印每个规则执行的原因
    snakemake -n -r
为特定的规则重新指定线程数和资源

snakemake--set-threads myrule=2cores
snakemake--set-resources myrule:partition="foocores 4
### 2. profile
snakemake 包含非常多的参数，如果都写在命令行将会非常臃肿，且不够简洁且容易出错此时，可以使用 -profile 参数来设置一个配置文件，将參数写入到文件中方便使用和查
snakemake -profile myprofile
这里传入的 myprofile 其实是一个文件夹，文件夹中需要一个名为 config.yaml 的配置文件，这个文件夹可以是绝对路径也可以是相对路径，如果该路径添加到了环境变量中，也可以直接传入该变量名。
例如， config.yaml 文件中存贮如下信息
cluster: qsub
jobs: 100
表示使用 qsub 提交到集群，且并行任务不超过 10日个，也可以使用 -cluster 参数，例
snakemake --cluster gsub--jobs 188
两种方式是等价的，但通常我没可能还需要指定一些其他参数，所以使用 profile 的形式会更好k
由于我使用的是 slurm 任务调度工具，下面我简单介绍一些其配置。根据官方推荐的使用方式，我们可以直接从模板中构建自己的配置信息。首先需要安装ookiecutter
mamba install cookiecutter
然后选择一个合适的目录来存储相应的 prof1le，例如
mkdir -p config
cd config
cookiecutter https://github.com/snakemake-Profiles/slurm.git

运行时会有提示信息，用于设置一些常用的参数值，新建成功之后在目录下会有一个名为slurm 文件夹，里面包含如下文件
CookieCutter.py
config.yaml
settings.json
slurm-jobscript.sh
slurm-sidecar.py
slurm-status.py
slurm-submit.py
slurm_utils.py
其中 config.yaml 里面包含了刚才设置的值，也可以自行添加或删除相应的参数。里面的参数可能是像这样的。由于我的系统中没有开发 sacct软件的使用，所以我把它注释了
jobscript:"slurm-jobscript.sh
cluster:"slurm-submit.py"
# custer status:"slurm-status.py"
max-jobs-per-second:“1”
max-status-checks-per-second:“10"
local-cores:1
latency-wait:"5"
use-conda: "True"
use-singularity: "False"
10jobs: "100"
11
12
printshellcmds: "False"
rerun-incomplete:"True
### 3.可视化
使用 --dag参数，可以将整个流程以流程图的形式展现
snakemake-dagldot -Tpdf > dag.pdf

snakemake--rulegraph|dot -Tpdf > dag_rules.pdf # 规则之间的运行关系
# 项目展示
通过学习上面的内容，根据网上搜索到的一些流程，我自己也写了一个简单的 cut&Tag 数据分析流程，其大致流程如下


主要包含一些上游的数据处理和分析，后续会陆续添加一些分析方法，不断完善。整个项目的布局如下
