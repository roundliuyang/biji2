# 在 IDEA 中使用 Debug



## 1. Debug 简介

我们写程序不要一遇到问题就写一堆 system 打印语句，真的很浪费时间。

而使用 Debug 可以追踪程序的执行过程，快速定位程序异常的位置，帮助我们快速找到出错的代码。





## 2. 开启 Deubg



### 2.1 Debug 模式下的界面

我们先看下 IDEA 中 Debug 模式下的界面：

![img](在 IDEA 中使用 Debug.assets/cbc3dee9ce274be2a299627cec22ed58_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



- 以 Debug 模式启动服务。在开发中，我一般会直接以 Debug 模式运行程序，方便随时调试代码。

- 
  断点，我们可以在行数栏左侧直接单击设置，也可以使用快捷键 Ctrl+F8 设置或者取消断点。

- 
  Debug 窗口：当请求到达第一个断点后，Debug 窗口会被激活。

- 
  调试 Debug 按钮：我们在调试过程中主要使用这几个按钮，鼠标悬浮按钮上面可以显示快捷键。

- 
  Debug 服务按钮：在这里我们可以开启、关闭 Debug 服务等。

- 
  方法区：这里会显示调试过程中执行的方法。




### 2.2 开启 Debug



先设置一个断点，然后以 Debug 模式运行：

![img](在 IDEA 中使用 Debug.assets/c5a62360e8a2435dba7466c8fdb966fb_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

注：我们还可以在执行程序的过程中添加/删除断点。





## 3. Debug 中常用调试按钮



![img](在 IDEA 中使用 Debug.assets/6eb8d12664f04cf2b48bab5817104ec9_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





### 3.1 跳转到当前执行代码的行

![img](在 IDEA 中使用 Debug.assets/8cca8132e88746ef955ec9c5a07f5deb_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





我们先在一个页面设置一个断点，然后再切换到其他页面，点击这个按钮，发现又跳转到了执行代码所在的行：

![img](在 IDEA 中使用 Debug.assets/ad063c4c709b439bab5b25332cadd88c_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





### 3.2 步过



![img](在 IDEA 中使用 Debug.assets/cf1df3d890584d32a08211b5b0276467_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

`步过`就是一步一步往下走，跳过所有方法：

![img](在 IDEA 中使用 Debug.assets/0e3055b328244b95bf969afa12d38c23_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



上面的例子中即便是遇到了 system 打印方法和 test1 方法，也会跨过去继续往下执行。





### 3.3 步入



![img](在 IDEA 中使用 Debug.assets/ba73286c10974a90a20c9e9b655e5a02_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



在执行的过程中如果遇到了`自定义`的方法，可以进入方法内部，不会进入 JDK 类库中的方法。

![img](在 IDEA 中使用 Debug.assets/463ceb7a16934b2ab62adc2717761cd8_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

上面的例子中，遇到了 system 方法会自动跨过去，但是遇到了自定义的方法，则会进入到方法中执行，等执行完则会返回到方法的调用处。





### 3.4 强制步入



![img](在 IDEA 中使用 Debug.assets/e3b69a556b8a4cb5953b6660ccb32a47_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



不管遇到 JDK 的类库方法还是自定义方法，都会进入到方法中执行。

![img](在 IDEA 中使用 Debug.assets/a9fdde0416c0467b93a51f55274c01b8_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

上面的例子中不管遇到了JDK 类库中的 system 方法还是自定义的 test1方法，都会进入到方法中执行。





### 3.5 步出



![img](在 IDEA 中使用 Debug.assets/458d905ee81c4587a38b663ad36fd08f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



`步出`就是从进入的方法内部退回到方法调用处。

![img](在 IDEA 中使用 Debug.assets/a78a3d288a0f4279af03edc3ba1223ca_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

上面的例子中我们进入到了 test1 方法的内部，当点击**步出**按钮后，又回到了调用 test1 方法的地方。





### 3.6 回退断点处



![img](在 IDEA 中使用 Debug.assets/f2b165642baf41acac2cb7fd216c935a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



`回退断点处`意思就是可以回退到指定方法的调用处。

![img](在 IDEA 中使用 Debug.assets/16d1b085fe73499e839799a3135b9dc0_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



上面的例子中我们依次执行了 test1、test2、method2 方法，但是我们可以选择直接回退到 test1 方法的调用处。

> 步出和回退断点的区别：

- 都是回到方法的调用处
- 步出只能回到当前方法的调用处
- 回退断点可以回到指定方法的调用处，前提是该方法已经被执行过。





### 3.7 定位到光标处



![img](在 IDEA 中使用 Debug.assets/80043baa5ec64085b407a970aacaa7e1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

如果我们写的代码有几百行，一步一步往下执行也比较费时间。这时候我们可以先把光标放到一个指定位置，然后点击`定位光标处`按钮，这时候代码就会立即执行到光标处了。

![img](在 IDEA 中使用 Debug.assets/2a8609ddf7d444c6a9243cdbe2247d9d_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



上面的例子中我们把鼠标光标移动到了下面某一行，然后点击`定位光标处`按钮，代码立刻执行到了这一行。





### 3.8 计算表达式



![img](在 IDEA 中使用 Debug.assets/e1a71d8fde634f1bafccd36cf4877437_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



`计算表达式`可以帮助我们计算一些表达式的返回值。

![img](在 IDEA 中使用 Debug.assets/d2f4d7cdd0644633b67141800e68b486_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



从上面的例子中可以看出，我们可以在调用某些方法之前使用一些自定义参数去计算该方法的返回值。





## 4. 查看参数



### 4.1 参数所在行后面显示

![img](在 IDEA 中使用 Debug.assets/6403494622ea4a688779d181f78acbb3_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





### 4.2 光标悬浮查看

光标悬浮到参数上，显示当前变量的信息，我经常使用这种方法，特别方便。

![img](在 IDEA 中使用 Debug.assets/e4cbca3201f74037ad6019a2a36de80c_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



### 4.3 在 Variables 里查看

这里显示当前方法里的所有变量。

![img](在 IDEA 中使用 Debug.assets/96ca244421084e489449831c16a2bfcc_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)







### 4.4 在 Watches 里查看



在 Watches 里，点击 New Watch，输入需要查看的变量:

![img](在 IDEA 中使用 Debug.assets/25bad2a27e4f4370a35c44a224ba4749_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



![img](在 IDEA 中使用 Debug.assets/20bcaa07f9d941018788578512a37bf6_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



## 5. 条件断点



有时候我们代码中会包含很多 for 语句，但是使用断点调试的时候会执行很多次。这时候我们可以选择断点，鼠标右键设置一个条件，只有满足该条件时，断点才会执行到此处。

![img](在 IDEA 中使用 Debug.assets/68c384ab2b6d4792a8291e7407f82aaf_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

上面的例子中我们给断点设置一个条件：i==50，所以以 Debug 模式运行该程序的时候我们发现此时 i 就是50。







## 6. Debug 服务设置按钮



### 6.1 执行到下一个断点处直到结束



![img](在 IDEA 中使用 Debug.assets/da5d93b4fac44943b4f2a6f4f92391c9_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

该按钮的作用：如果下面有断点，就跳转到下一个断点处。如果没有，程序就执行结束。

![img](在 IDEA 中使用 Debug.assets/5ea57d1f54674876b2ff013d38f6a199_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





### 6.2 断点静音



![img](在 IDEA 中使用 Debug.assets/1f9bd7bc7c1546f68c494c669c9c38b7_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



有时候我们在执行到某一步的时候已经知道了结果，但是后面还有一堆断点。我想让这些断点失效，但是第二次跟踪我还想用这些断点，这时候就可以使用`断点静音`。

![img](在 IDEA 中使用 Debug.assets/b86d891bb4fc4eabbfe70250f76a390e_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





### 6.3 查看/清除断点



![img](在 IDEA 中使用 Debug.assets/08ab6dd510b4423e9855e85e73f1ebb3_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)



我们在执行完代码后要清除所有断点，但是一个个去清除太浪费时间，这时候就可以使用这个按钮查看所有设置的断点，或者清除所有断点。

![img](在 IDEA 中使用 Debug.assets/163db9033b434c94a1b6bc62bfca4ebe_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)





### 6.4 返回到第一个断点的地方



![img](在 IDEA 中使用 Debug.assets/b34785b150564aad93de1bb9661214d1_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

代码执行到某一行想返回到第一个断点处：

![img](在 IDEA 中使用 Debug.assets/142f182d60e44534812e99cb7caa3a48_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)









## 7 多线程调试



因为 CPU 执行线程的顺序是随机的，但是我们使用断点调试可以自定义执行下一个线程。

首先将两个线程的断点都设置成线程模式：

![img](在 IDEA 中使用 Debug.assets/4d7cce1f5a294b07b908ebc5ed015b09_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

然后在方法区选择指定的线程去执行：

![img](在 IDEA 中使用 Debug.assets/6fb8a64164fa47a9bf027b76c0ab7e8a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

