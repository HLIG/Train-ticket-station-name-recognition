# Train-ticket-station-name-recognition
Train ticket station name recognition method based on variant grid characteristics and Bayesian network

### Introduction
针对目前利用开源OCR引擎或者模板匹配方法直接识别火车票站名效果不理想的状况，设计一种通过边缘检测算法校正车票，利用车次位置定位目标区域，基于变种网格特征和贝叶斯方法识别火车票站名拼音，
接着利用模板匹配对易混淆字符进行再识别，然后根据车次匹对內建数据库对应站名拼音，依据一种字符串匹配算法计算匹配度，最终根据匹配度最高的站名拼音找到对应的中文站名的火车票站名识别算法。
算法移植到Jetson TK1上的效果表明，该算法能够在不同光照、阴影和是否抖动的情况下，实时准确地识别出火车票站名。

算法分为车票校正、文本检测、字符分割、字符识别、字符再识别和数据库匹对六部分，代码在VS2013基于OpenCV2.4.9利用C++进行编写，名字为code/[main.cpp](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/code/main.cpp)。

### 车票校正
由于拍摄角度和拍摄距离的变化，拍摄得到的火车票图片会存在一定角度倾斜和大小变化，且为了将车票从复杂的背景中分离出来，方便后续环节的处理，需要对每次拍摄的图片通过透视变换，旋转缩放成水平的角度和标准的尺寸。
车票校正过程首先需要对输入图像进行高斯模糊，然后灰度化，并对灰度图基于canny算法提取图像边缘，然后利用霍夫直线检测原理检测边缘直线，如[图1](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/1.jpg)所示。
然后，根据检测直线的角度和原点距离，筛选出车票的上下左右四条边，求四条边的两两交点，得到车票的四个顶点，如[图2](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/2.jpg)所示。
最后，通过透视变换得到一张900*600分辨率的图像，该图像中只包含水平放置的火车票，如[图3](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/3.jpg)所示。

### 文本检测
由于车票汉字采集困难的问题，因此，通过识别火车票站名拼音，然后利用数据库匹对的方法，找到对应的站名汉字。
目标区域定位过程首先对图像进行高斯模糊和灰度化处理，然后利用Sobel算子进行边缘检测，得到[图4](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/4.jpg)，
再利用OSTU二值化和一系列形态学操作之后，得到[图5](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/5.jpg)。
接着，检测图片轮廓及其最小外接矩形，如[图6](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/6.png)所示，根据面积、宽高比、高度、旋转角度、占空比等先验知识进行最小外接矩形筛选，
检测得到站名拼音文字块[图7](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/7.jpg)和[图8](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/8.jpg)。

### 字符分割
将文字块进行灰度化和OSTU二值化，计算二值图像的轮廓及最小外接矩形，根据面积、宽高比、高度、占空比等先验知识进行最小外接矩形筛选，对筛选后的的图片进行仿射变换输出每个单独的字符，如[图9](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/9.png)。

### 字符识别
字符识别分为特征提取和分类器设计两部分。提取特征时，先切除字符四周的空白部分，然后归一化到宽为30，高为42，如[图10](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/10.png)所示。将图像均等分为7行5列共35个网格，统计每个网格中白色像素的个数，并计算其与网格像素总数的比值，构成前面35为特征，
再分别统计每行（每列）非零像素的个数，计算其与该行（列）对应像素总数的比值，构成后面72为特征，最终提取得到一个107维的变种网格特征。分类器考虑实时性的因素，放弃采用模板匹配的方法，而采用贝叶斯方法进行设计。
<br>***注意：因为训练模型时是通过读取保存的.bmp格式图片作为模型的输入，为了保持训练和测试的一致性，字符识别时需要将切割好的字符保存成.bmp格式，然后读取字符图片进行识别***

### 字符再识别
a,g,n,u四种火车票印刷字符比较容易出现孔洞收缩或者孔洞小时的现象，贝叶斯模型对此识别效果不佳，因此，需要利用模板匹配的方法进行再识别。

### 数据库匹对
基于变种网络特征的贝叶斯网络进行分类加上模板匹配再识别得到的识别结果已经能够在绝大部分情况下识别成功。然而，将站名拼音和站名汉字进行匹对需要绝对正确的站名拼音，而由于各种干扰情况，会导致识别结果与绝对正确的站名拼音存在些许的差别，
需要设计一种站名拼音字符串匹配算法对识别错误的站名拼音字符串进行校正，字符串匹配算法代码利用C++进行编写，在文件夹[/code/String_matching](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/tree/master/code/String_matching)内。对数据库进行匹对时，若知道火车票的车次，则可在数据库中筛选出隶属于该车次的所有站名拼音，
逐个站名拼音与待匹配的站名拼音字符串进行匹配，匹配完成后，将匹配度最高对应的站名拼音对应的站名汉字，作为最终识别的火车票站名。

<br>![image](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/6.png)
<br>![image](https://github.com/deepthinking-qichao/Train-ticket-station-name-recognition/blob/master/Processed_image/9.png)

# ***Hope this help you***
