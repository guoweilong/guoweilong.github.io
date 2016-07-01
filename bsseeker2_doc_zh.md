

亚硫酸盐测序(bisulfite sequencing)的序列比对的基本流程
=============================

##### by Guo, Weilong; 更新于2016年7月1日; 联系邮箱: guoweilong[艾特]126[点]com

目前，DNA甲基化研究中的主要测序文库包括：WGBS（whole genome bisulfite sequencing）和RRBS（reduced representation bisulfite sequencing）。其中，WGBS的文库以MethylC-seq和PBAT为主。

用于比对BS-seq数据并绘制DNA甲基化图谱的主流工具有：[BS-Seeker2](http://pellegrini.mcdb.ucla.edu/BS_Seeker2/)、bismark和BSMAP。本文以BS-Seeker2([source](https://github.com/BSSeeker/BSseeker2))为例，介绍BS-seq数据分析的[基本流程](http://pellegrini.mcdb.ucla.edu/BS_Seeker2/BSseeker2_pipeline.jpg)。



步骤一：首先建立针对BS-seq数据的index
-------------------------

* BS-Seeker2和Bismark的基本策略相似，都是将基因组和测序数据看作"三碱基"再进行比对；在序列的比对过程中调用bowtie或者bowtie2。

* 和bowtie/bowtie2相似，BS-Seeker2也需要首先建立一个index。但要注意的是，BS-Seeker2不能直接使用bowtie的index，而是需要创建特定的index。

* BS-Seeker2针对WGBS和RRBS设计了不同的创建index的策略。简言之，BS-Seeker2建立RRBS的index远小于为WGBS建立的index。该策略将大幅缩小序列比对的搜索空间，提高计算效率。[>>QA](https://github.com/BSSeeker/BSseeker2#qa11)


	    # 建立WGBS版本的index
	    bs_seeker2-build.py -f genome.fa
	    
	    # 建立RRBS版本的index
	    bs_seeker2-build.py -f genome.fa -r
	    
	    # 建立自定义fragment长度的RRBS版本index
	    bs_seeker2-build.py -f genome.fa -r -l 40 -u 400
	    
	    # 如果你的RRBS的酶切位点不是C^CGG，比如是 A^CGT和G^CWGC的组合
	    bs_seeker2-build.py -f genome.fa -r -c A-CGT,G-CWGC


* 默认情况下，index会被建立在BS-Seeker2的 “./bs_utils/reference_genomes/” 目录下；如果需要改变目录，请使用 -d <path>（或--db=<path>）参数。

* 另外，BS-Seeker2可选择bowtie或者bowtie2作为aligner（参数：--aligner=bowtie 或 --aligner=bowtie2）。利用不同的aligner建立的index也是不一样的。

* 以人基因组为例，创建index需要花几个小时的时间。当然，这个步骤“一劳永逸”。



步骤二： 对测序数据进行序列比对
-------------------------

* 获得测序数据，如果已经建立了index，那么就可以直接进入比对步骤。这一步骤既是“最花时间”，也是“最不花时间”的。序列比对在所有的二代测序分析中都要花费大量的时间，但是这个步骤可以通过将大数据分成小块在集群上并行运行。因而也是最容易压缩时间、提高效率的阶段。

	    # split the large input file into small pieces
	    split -l 4000000 input.fq spt_ 
	    # align all the splitted files using bs_seeker2-align.py
	    # when all the splitted files are aligned
	    # merge all the generated bam files
	    samtools merge merge.bam spt_*.bam
		

* 序列比对阶段最简单的例子如下：

	    bs_seeker2-align.py -i test.fa -o test.bam -g genome.fa

* 当然，-i后的输入文件可以是fastq、fasta、qseq或者 纯序列格式（即一行一个序列）。BS-Seeker2可以通过读入文本自动识别出文件的格式。如果输入文件是gzip压缩文件，请将该文件后缀名设置为“.gz”，BS-Seeker2将可直接读入压缩格式，而无须用户亲自解压。[>>QA](https://github.com/BSSeeker/BSseeker2#qa23)

	    bs_seeker2-align.py -i test.fa.gz  -o test.bam -g genome.fa

* BS-Seeker2自带去除adapter序列的功能，只需要用户将adapter序列(如AGATCGGAAGAGCACACGTC)写入文件（如 adatper.txt），然后设定参数 "-a"即可。[>>QA](https://github.com/BSSeeker/BSseeker2#qa71)

	    bs_seeker2-align.py -i test.fa -o test.bam -g genome.fa -a adapter.txt

* 现在主流的BS-seq 文库都是directional的。如果你的数据是un-directional的，可以额外设定参数“-t Y”。

* 设定mismatch的参数（含举例）：

	    
	    -m 4     #允许一个read至多有4个mismatch
	    -m 0.04  #允许一个read至多有4%的错配：如果是150bp，即允许6个错配
				 #如果设定去掉adapter，那么reads有可能长短不一，该情况可设定百分比


* BS-Seeker2输出的bam文件中只包含unique best hit。[>>QA](https://github.com/BSSeeker/BSseeker2#qa51)

	- 如果你需要被过滤掉的multiple hits的read的信息，可指定参数“-M XX （或--multiple-hit=XX）”。
	- 如果你需要没有任何比对位置的read的信息，可指定参数“-u XX (或--unmapped=XX)”。


* 考虑到用户可能同时提交上百个bs_seeker2-align.py程序同时进行比对。我们将临时文件目录默认设置在各个运行节点的本地目录上 /tmp，以减少集群I/O的负担。但是如果您的服务器/tmp的空间确实有限，请利用参数“--temp_dir=XXX”手动设置临时文件存放点。在程序正常运行结束的情况下，BS-Seeker2会自动清空临时文件目录。[>>QA](https://github.com/BSSeeker/BSseeker2#qa14)


* BS-Seeker2在比对中，默认情况下会执行4个进程（directional library需要比对两套基因组，每套基因组启动两个线程：相当于 --bt-p 2 或 --bt2-p 2）。用户可以通过 --bt-p N (--bt2-p N)去设定程序实际执行过程中调用的进程数目。如果你的计算机有8个核，那么你可以将 N 设置为4。[>>QA](https://github.com/BSSeeker/BSseeker2#qa13)


* 为了用户更灵活地配置内部的bowtie或bowtie2的执行参数，BS-Seeker2提供了向bowtie/bowtie2传递参数的接口（见bs_seeker2-align.py的参数说明）。


* 根据经验，针对双端测序的数据，比对过程中利用单端模式相对于双端模式可以比对地“更多、更准、更高效”。虽然BS-Seeker2和其它同类软件一样，也提供了双端比对模式，如果测序读长大于80bp的话，建议使用单端模式对每个mate进行比对，最后再将生成的所有bam文件合并在一起。[>>QA](https://github.com/BSSeeker/BSseeker2#qa62)


步骤三： 用于计算各个碱基位点的甲基化水平
-------------------------

* 下面给出基本的执行命令：

	    bs_seeker2-call_methylation.py -i test.bam -o output --db <BSseeker2_path>/bs_utils/reference_genomes/genome.fa_bowtie/

* 如果数据是WGBS或者PBAT，且基因组比较大（比如人和小鼠），该步骤的计算时间会比较长，约十个小时左右。

* 在该步骤中，BS-Seeker2读入bam文件后，首先对bam文件进行sort，并建立bai文件索引。如果读入的文件是已经经过排序的，就可以先指定参数"--sorted"，为计算节省时间。

* 该步骤中将输出WIG，CGmap和ATCGmap格式的文件；后缀名如果是“.gz”，那么输出的文件将是压缩格式的，可大大节省磁硬盘空间。

* 在这一步骤中，BS-Seeker2还提供了额外的功能。

  1. BS-Seeker2在align步骤中，利用CH位点的甲基化信息判断read是否failed in bisulfite-conversion，并給予标签XS:i:0 或 XS:i:1。在call methylation步骤中，设置参数 “-x”（或“--rm-SX”）可移除被标记为 XS:i:1 的read。

  2. 对于RRBS来说，C^CGG位点上的对链是后面步骤合成并添加的，计算甲基化水平的过程中应去掉该位置。bs_seeker2-call_methylation.py 程序提供了参数“--rm-CCGG”可以在最终的结果中移除CCGG位点。[>>QA](https://github.com/BSSeeker/BSseeker2#qa72)

  3. 对于双端测序来说，两个mate之间可能有overlap；这种情况下使用参数“--rm-overlap”，保证同一对reads的重合的区域只被计算一次。[>>QA](https://github.com/BSSeeker/BSseeker2#qa63)




推荐示例（仅供参考）
-------------------------

* WGBS

	    # build index
	    bs_seeker2-build.py -f hg18.fa
	    
	    # for single-end
	    bs_seeker2-align.py -i input.fq.gz -o output.bam -g hg18.fa
		bs_seeker2-call_methylation.py -i test.bam -o output -x \
		      --db <BSseeker2_path>/bs_utils/reference_genomes/hg18.fa_bowtie/
	    
	    # for paired-end
	    bs_seeker2-align.py -i R1.fq.gz -o R1.bam -g hg18.fa
	    Antisense.py -i R2.fq.gz -o R2_antisense.fq.gz 
	    bs_seeker2-align.py -i R2_antisense.fq.gz -o R2.bam -g hg18.fa
	    samtools merge merge.bam R1.bam R2.bam
		bs_seeker2-call_methylation.py -i merge.bam -o output --rm-overlap -x \
		      --db <BSseeker2_path>/bs_utils/reference_genomes/hg18.fa_bowtie/


* RRBS

	    # build index
	    bs_seeker2-build.py -f hg18.fa -r

	    # for single-end
	    bs_seeker2-align.py -i input.fq.gz -o output.bam -g hg18.fa -r
		bs_seeker2-call_methylation.py -i test.bam -o output --rm-CCGG -x \
		    --db <BSseeker2_path>/bs_utils/reference_genomes/hg18.fa_rrbs_20_500_bowtie/

	    # for paired-end
	    bs_seeker2-align.py -i R1.fq.gz -o R1.bam -g hg18.fa -r
	    Antisense.py -i R2.fq.gz -o R2_antisense.fq.gz 
	    bs_seeker2-align.py -i R2_antisense.fq.gz -o R2.bam -g hg18.fa -r
	    samtools merge merge.bam R1.bam R2.bam
		bs_seeker2-call_methylation.py -i test.bam -o output --rm-CCGG --rm-overlap -x \
		    --db <BSseeker2_path>/bs_utils/reference_genomes/hg18.fa_rrbs_20_500_bowtie/


* PBAT

	    # build index
	    bs_seeker2-build.py -f hg18.fa 

	    # for single-end
	    Antisense.py -i input.fq.gz -o input_antisense.fq.gz 
	    bs_seeker2-align.py -i input_antisense.fq.gz -o output.bam -g hg18.fa -r
		bs_seeker2-call_methylation.py -i test.bam -o output --rm-CCGG -x \
		    --db <BSseeker2_path>/bs_utils/reference_genomes/hg18.fa_rrbs_20_500_bowtie/

	    # for paired-end
	    Antisense.py -i R1.fq.gz -o R1_antisense.fq.gz 
	    bs_seeker2-align.py -i R1_antisense.fq.gz -o R1.bam -g hg18.fa
	    bs_seeker2-align.py -i R2.fq.gz -o R2.bam -g hg18.fa
	    samtools merge merge.bam R1.bam R2.bam
		bs_seeker2-call_methylation.py -i merge.bam -o output --rm-overlap -x \
		    --db <BSseeker2_path>/bs_utils/reference_genomes/hg18.fa_bowtie/






