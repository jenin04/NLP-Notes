

# dataset

之前自己一直使用的是tf.data.dataset，这种方法是典型的结合pandas，从dataframe导入数据为tensor的方法，这种方法的优点是：特征处理特别灵活，因为dataframe的相关库或者函数特别多，能够实现很多精细化的特征处理。这种方法的缺点是：由于关于dataframe的操作只能在内存中执行，故如果数据量特别大时，内存代价难以接受，且若采用chunk，一些精细化的特征处理便无法正常使用，因为相关的库和函数是与单个dataframe有关，没法在chunk的dataframe起作用，如此特征处理就变烦琐了。且tf.data中相关的为了节省内存和提升运行速度而使用的多进程都没办法正常加入到代码中发挥作用。

tf.data.textlineDataset,这种方法是直接从文本文件读取数据的典型方法。这种方法的优点是：能够结合tf.data中节省内存和提升运行速度而设计的机制。这种方法的缺点是：中间有些map函数不是很好调bug，因为无法直接输入值而看值的变化，其次，对于实时精细化的特征处理需要辅助配置文件的帮助，没法很快地进行精细化特征处理。

# Feature column

<img src="https://upload-images.jianshu.io/upload_images/5690581-3e3cfd7f9a3f8037.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1172/format/webp" alt="img" style="zoom:50%;" />

## 数值列 - tf.feature_column.numeric_column



## 分桶列 - tf.feature_column.bucketized_column

注意，指定一个三元素边界矢量可创建一个四元素分桶矢量。

## 分类标识列 - tf.feature_column.categorical_column_with_identity

分类标识列映射是一种独热编码。正常情况下，我们要表示分类，4类可以用0，1，2，3这样的数字序列来表示。也可以用向量来表示, 将对应的位置为1，其他位置为0， 例如1表示为[0,1, 0, 0]，3表示为[0,0, 0, 1]。

<img src="https:////upload-images.jianshu.io/upload_images/5690581-4fb9b8f35bb7af4d.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/630/format/webp" alt="img" style="zoom: 50%;" />

## 分类词汇列 - tf.feature_column.categorical_column_with_vocabulary_list / tf.feature_column.categorical_column_with_vocabulary_file

在分类标识列中，我们将数字直接表示为向量。我们不能直接向模型中输入字符串。相反，我们必须首先将字符串映射到数值或分类值。分类词汇列提供了一种将字符串表示为独热矢量的好方法。例如：

<img src="https://upload-images.jianshu.io/upload_images/5690581-48158c9447fb45f4.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/820/format/webp" alt="img" style="zoom:50%;" />

当词汇表很长时，需要输入的内容太多了。对于此类情况，请改为调用`tf.feature_column.categorical_column_with_vocabulary_file`，以便将词汇放在单独的文件中。

## 分类哈希分桶列 - tf.feature_column.categorical_column_with_hash_bucket

通常，类别的数量非常大，以至于无法为每个词汇或整数设置单独的类别，因为这会消耗太多内存。对于此类情况，我们可以反问自己：“我愿意为我的输入设置多少类别？”实际上，，tf.feature_column.categorical_column_with_hash_bucke 函数使您能够指定类别的数量。对于这种类型的特征列，模型会计算输入的哈希值，然后使用模运算符将其置于其中一个 `hash_bucket_size` 类别中。

给定k各类别和hash_bucket_size，碰撞得到的概率为 1 - exp(-k*(k-1)/2/hash_bucket_size

<img src="https://upload-images.jianshu.io/upload_images/5690581-e9dcd64de352f6df.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:50%;" />

## 组合列 - tf.feature_column.crossed_column



## 嵌入列 - tf.feature_column.embedding_column

指标列将每个类别视为独热矢量中的一个元素，其中匹配类别的值为 1，其余类别为 0。当数据数量特别大是，这个向量非常大，变得不容易处理。

嵌入列并非将数据表示为很多维度的独热矢量，而是将数据表示为低维度普通矢量，其中每个单元格可以包含任意数字，而不仅仅是 0 或 1。通过使每个单元格能够包含更丰富的数字，嵌入列包含的单元格数量远远少于指标列。

假设我们的输入样本包含多个不同的字词（取自仅有 81 个字词的有限词汇表）。我们进一步假设数据集在 4 个不同的样本中提供了下列输入字词：

<img src="https:////upload-images.jianshu.io/upload_images/5690581-9e78472900618f27.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:50%;" />

下面公式提供了关于嵌入维度数量的一般经验法则(类别数量开4次方）：
$embedding_dimensions = number_of_categories**0.25$



<img src="https://upload-images.jianshu.io/upload_images/5690581-05fed24c8b1081b8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:50%;" />

