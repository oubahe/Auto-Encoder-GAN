# Auto-Encoder-GAN
Generator通过噪声输入生成假图片，Discriminator通过学习训练图像集来判断假图片的真假，两者对抗训练，最终训练完成Generator生成的图片通过AutoEncoder去噪生成新的图像样本

Attention:
  1、数据集为MNIST数据集
  2、Auto-Encoder和Dense-GAN部分超参数可自定义修改
  3、训练过程为设置batch_size后全部训练样本循环训练完为一次epoch，约100次epoch后生成新图像样本可视化结果较好
  4、运行中会显示两个可视化结果，分别是generator_loss和discriminator_loss的曲线图以及10次epoch后的新图像样本RGB图
  5、直接运行Dense-GAN即可
