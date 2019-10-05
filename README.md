# Auto-Encoder-GAN
Generator通过噪声输入生成假图片，Discriminator通过学习训练图像集来判断假图片的真假，两者对抗训练，最终训练完成Generator生成的图片通过AutoEncoder去噪生成新的图像样本

1、数据集为MNIST数据集
2、Auto-Encoder和Dense-GAN部分超参数可自定义修改
3、训练过程为设置batch_size后全部训练样本循环训练完为一次epoch，约100次epoch后生成新图像样本可视化结果较好
4、运行中会显示两个可视化结果，分别是generator_loss和discriminator_loss的曲线图以及10次epoch后的新图像样本RGB图
'''
Author:oubahe
Github:oubahe
Email:oubahe@126.com
FileName:DCGAN.py
Date:2019/10/2
'''
##################################### tensorflow实现mnist数据集的GAN生成 ################################################
import numpy as np
import tensorflow as tf
import matplotlib.pyplot as plt
import os
import argparse
from tensorflow.examples.tutorials.mnist import input_data
import matplotlib.gridspec as gridspec

# 定义参数初始化器xiaver的标准差
def xavier_init(size):
    dimen = size[0]
    xavier_stddev = tf.sqrt(2.0/dimen)
    return xavier_stddev

# 读取mnist数据
def read_mnist():
    mnist = input_data.read_data_sets("MNIST/data/", one_hot=True)
    return mnist

# 定义可视化函数
def plot(samples):
    fig = plt.figure(figsize=(8, 8))
    gs = gridspec.GridSpec(8, 8)
    gs.update(wspace = 0.05, hspace = 0.05)
    for i, sample in enumerate(samples[:64]):
        ax = plt.subplot(gs[i])
        plt.axis('off')
        ax.set_xticklabels([])
        ax.set_yticklabels([])
        plt.imshow(sample.reshape((28, 28)), cmap='Greys_r')
    return fig

# 定义卷积自编码器进行去噪
class ConvEncoder:
    def __init__(self):
        self.x = tf.placeholder(tf.float32, shape = [None, 28, 28, 1])

    def cae(self):
        # Encoder
        x1 = tf.layers.conv2d(self.x,
                              filters = 32,
                              kernel_size = (3, 3),
                              strides=(1, 1),
                              padding='same',
                              activation=tf.nn.relu)
        x1 = tf.layers.max_pooling2d(x1,
                                     pool_size=(2, 2),
                                     strides = (2, 2),
                                     padding='same')
        x2 = tf.layers.conv2d(x1,
                              filters = 16,
                              kernel_size = (3, 3),
                              strides=(1, 1),
                              padding='same',
                              activation=tf.nn.relu)
        x2 = tf.layers.max_pooling2d(x2,
                                     pool_size = (2, 2),
                                     strides = (2, 2),
                                     padding='same')
        # Decoder
        x3 = tf.image.resize_nearest_neighbor(x2,
                                              size = (x1.get_shape().as_list()[1],
                                                      x1.get_shape().as_list()[2]))
        x3 = tf.layers.conv2d_transpose(x3,
                                        filters = 16,
                                        kernel_size = (3, 3),
                                        strides = (1, 1),
                                        padding='same',
                                        activation=tf.nn.relu)
        x4 = tf.image.resize_nearest_neighbor(x3,
                                              size=(self.x.get_shape().as_list()[1],
                                                    self.x.get_shape().as_list()[2]))
        x4 =  tf.layers.conv2d_transpose(x4,
                                        filters = 32,
                                        kernel_size = (3, 3),
                                        strides = (1, 1),
                                        padding='same',
                                        activation=tf.nn.relu)
        logits = tf.layers.conv2d(x4,
                                  filters=1,
                                  kernel_size=(3, 3),
                                  strides=(1, 1),
                                  padding='same',
                                  activation=None)
        self.decoder_out = tf.nn.sigmoid(logits)
        # loss
        self.loss = tf.reduce_mean(tf.pow(self.decoder_out-self.x, 2))
        # optimizer
        self.train_op = tf.train.AdamOptimizer(0.001).minimize(self.loss)

    def iteration_train(self, xtest, batch_size = 128, epochs = 1):
        self.cae()
        mnist = read_mnist()
        xtrain = mnist.train.images
        init = tf.global_variables_initializer()
        with tf.Session() as sess:
            sess.run(init)
            obj = []
            steps = len(xtrain)//batch_size
            for _ in range(epochs):
                for i in range(steps):
                    batch = xtrain[i*batch_size:(i+1)*batch_size]
                    batch = np.reshape(batch, newshape=[batch.shape[0], 28, 28, 1])
                    _, batch_loss = sess.run([self.train_op, self.loss],
                                                  feed_dict = {self.x:batch})
                    print('************The batch is %d***************'%i)
                    print("The batch loss is: ",batch_loss)
                    obj.append(batch_loss)
                    if len(obj)>2 and obj[-2]-obj[-1]<1e-5:
                        break
                if len(obj) > 2 and obj[-2] - obj[-1] < 1e-5:
                    break
            reconstruct_x = sess.run([self.decoder_out], feed_dict={self.x:xtest})
        return reconstruct_x

## 定义DCGAN网络
def get_generator(noise_img, reuse=False):
    """
    生成器

    noise_img: 生成器的输入
    n_units: 隐层单元个数
    out_dim: 生成器输出tensor的size，这里应该为32*32=784
    alpha: leaky ReLU系数
    """
    with tf.variable_scope("generator", reuse=reuse):
        # hidden layer
        hidden1 = tf.layers.dense(noise_img, 128)
        # leaky ReLU
        hidden1 = tf.nn.leaky_relu(hidden1, 0.01)
        # dropout
        hidden1 = tf.layers.dropout(hidden1, rate=0.2)
        # logits & outputs
        logits = tf.layers.dense(hidden1, 784)
        outputs = tf.tanh(logits)
        return logits, outputs

def get_discriminator(img, reuse=False):
    """
    判别器
    n_units: 隐层结点数量
    alpha: Leaky ReLU系数
    """
    with tf.variable_scope("discriminator", reuse=reuse):
        # hidden layer
        hidden1 = tf.layers.dense(img, 128)
        hidden1 = tf.nn.leaky_relu(hidden1, 0.01)
        # logits & outputs
        logits = tf.layers.dense(hidden1, 1)
        outputs = tf.sigmoid(logits)
        return logits, outputs

tf.reset_default_graph()
batch_size = 128
smooth = 0.01
mnist = read_mnist()
xtrain = mnist.train.images
steps = len(xtrain)//batch_size
batch_x = tf.placeholder(tf.float32, shape = [None, 784])
z = tf.placeholder(tf.float32, shape = [None, 100])
# generator
g_logits, g_outputs = get_generator(z, reuse=False)
# discriminator
d_logits_real, d_outputs_real = get_discriminator(batch_x)
d_logits_fake, d_outputs_fake = get_discriminator(g_outputs, reuse=True)
# discriminator的loss
# 识别真实图片
d_loss_real = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=d_logits_real,
                                                                     labels=tf.ones_like(d_logits_real)) * (1 - smooth))
# 识别生成的图片
d_loss_fake = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=d_logits_fake,
                                                                     labels=tf.zeros_like(d_logits_fake)))
# 总体loss
d_loss = tf.add(d_loss_real, d_loss_fake)

# generator的loss
g_loss = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(logits=d_logits_fake,
                                                                labels=tf.ones_like(d_logits_fake)) * (1 - smooth))

train_vars = tf.trainable_variables()
# generator中的tensor
g_vars = [var for var in train_vars if var.name.startswith("generator")]
# discriminator中的tensor
d_vars = [var for var in train_vars if var.name.startswith("discriminator")]

# optimizer
d_train_opt = tf.train.AdamOptimizer(0.001).minimize(d_loss, var_list=d_vars)
g_train_opt = tf.train.AdamOptimizer(0.001).minimize(g_loss, var_list=g_vars)

# sess
all_d_loss = []; all_g_loss = []
with tf.Session() as sess:
    sess.run(tf.global_variables_initializer())
    for epoch in range(301):
        for i in range(steps):
            batch = mnist.train.next_batch(batch_size)
            batch_images = batch[0].reshape((batch_size, 784))
            # 对图像像素进行scale，这是因为tanh输出的结果介于(-1,1),real和fake图片共享discriminator的参数
            batch_images = batch_images * 2 - 1
            # generator的输入噪声
            batch_noise = np.random.uniform(-1, 1, size=(batch_size, 100))
            # Run optimizers
            sess.run(d_train_opt, feed_dict={batch_x: batch_images, z: batch_noise})
            sess.run(g_train_opt, feed_dict={z: batch_noise})
        # 每一轮结束计算loss
        train_loss_d = sess.run(d_loss,
                                feed_dict={batch_x: batch_images,
                                           z: batch_noise})
        # real img loss
        train_loss_d_real = sess.run(d_loss_real,
                                     feed_dict={batch_x: batch_images,
                                                z: batch_noise})
        # fake img loss
        train_loss_d_fake = sess.run(d_loss_fake,
                                     feed_dict={batch_x: batch_images,
                                                z: batch_noise})
        # generator loss
        train_loss_g = sess.run(g_loss,
                                feed_dict={z: batch_noise})
        all_g_loss.append(train_loss_g)
        all_d_loss.append(train_loss_d_fake+train_loss_d_real)
        # print the loss
        print('********************** The epoch %d*********************' % epoch)
        print('The generator loss is : ', train_loss_g)
        print('The discriminator loss is : ', train_loss_d_fake+train_loss_d_real)
        # plot the loss
        plt.ion()
        plt.plot(all_g_loss, '-b')
        plt.plot(all_d_loss, '--g')
        plt.legend(['generator_loss', 'discriminator_loss'])
        plt.pause(5)
        plt.close('all')
        # print the new image
        if epoch%10==0:
            samples = sess.run(g_outputs, feed_dict={z:np.random.uniform(0, 1.0, [128, 100])})
            samples = np.reshape(samples, newshape=[samples.shape[0], 28, 28, 1])
            # 载入自编码器去噪
            reconstruct_samples = ConvEncoder().iteration_train(xtest=samples,
                                                                 batch_size=128,
                                                                 epochs=1)[0]
            reconstruct_samples = np.reshape(reconstruct_samples, newshape=[reconstruct_samples.shape[0], 784])
            plt.ion()
            for i in range(9):
                plt.subplot(3, 3, i+1)
                plt.imshow(reconstruct_samples[i].reshape((28, 28)))
            plt.pause(5)
            plt.close('all')

