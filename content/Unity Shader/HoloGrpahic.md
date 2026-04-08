## 1.Change Color Based On View Direction
使用viewDirection，切换到切线空间
使用viewDirection的x和y分量进行计算，最后用于Hue的offset值
![[Pasted image 20241217005404.png]]


## 2.Foil Pattern
读取图案，对图案进行power加强处理
![[Pasted image 20241217005602.png]]

图案示例
![[Pasted image 20241217005734.png]]


## 3.叠加效果
根据图案的黑白，与1中的color相乘，只有图案有值
最后与卡片图片叠加颜色
![[Pasted image 20241217005807.png]]