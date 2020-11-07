# Tomcat缺失字体问题

本篇主要介绍解决`线上环境Tomcat缺失字体的问题`，如果知道操作系统中字体作用以及工作原理的同学，看到这个问题可能会觉得这个问题是不成立的，因为`Tomcat`不存在字体与不是字体的问题，它只是一个容器而已，存在字体问题都是因为里面的代码使用了字体库，但是环境中没有才会导致，所以和是不是`Tomcat`没有关系，`控制台`程序也会有问题，这是`JRE`的问题。字体问题和编码问题不是同一类问题，出现乱码时该有的位置都是奇怪的特殊字符，缺少字体时会存在一个一个方块无法显示。




----------------


## 一、问题现象描述

最近在工作过程中接到前方同学的提问，在`Windows`环境中写的一个生成图片水印的代码运行的好好的，但是放在`Linux 的Docker容器中`发现中文位置全都是方块,很明显是缺少字体的问题。现象如下图所示

![](images/缺少字体1.png)

问题代码如下
```java
import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.font.*;
import java.awt.geom.Rectangle2D;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
import java.io.FileInputStream;
public class AppsData {
  //args[0]  水印的文字内容
  //args[1]  水印图片的文件存储位置
  //args[2]  自定义字体位置
  public static void main(String[] args) throws IOException,FontFormatException {
    //Font font=Font.createFont(Font.TRUETYPE_FONT,new File(args[2]));
    Font font=new Font("宋体",1,24);
    GraphicsEnvironment ge =GraphicsEnvironment.getLocalGraphicsEnvironment();
    System.out.print(ge.registerFont(font));
    int with=300;
    int hight=300;
    BufferedImage image=new BufferedImage(with,hight,BufferedImage.TYPE_INT_ARGB);
    Graphics2D g2d=image.createGraphics();
    image=g2d.getDeviceConfiguration().createCompatibleImage(with,hight,3);
    g2d.dispose();
    g2d=image.createGraphics();
    g2d.setColor(new Color(10));
    g2d.setStroke(new BasicStroke(1.0F));
    FontRenderContext context=g2d.getFontRenderContext();
    Rectangle2D bounds=font.getStringBounds(args[0],context);
    double x=(with-bounds.getWidth())/2.0;
    double y=(hight-bounds.getHeight())/2.0;
    double ascent=-bounds.getCenterY();
    double baseY=y+ascent;
    g2d.rotate(Math.toRadians(-45.0D),with/2,hight/2);
    g2d.drawString(args[0],(int)x,(int)baseY);
    g2d.dispose();
    ImageIO.write(image,"png",new File(args[1]));
  }
}
```

## 二、问题解决思路

代码主要部分就是调用了`JDK`的图形库，使用`Graphics2D`类来画一个图像，水印使用`宋体`字体。针对前方人员反馈的问题来看，最简单的解决方法就是在当前环境中添加一个字符集即可，对于`Centos`来说直接扔到`/usr/share/fonts/chinese`这个路径下即可(如果目录没有直接新建)，完了重启计算机，对于`Ububtu或者Arch`来说可能就在其他地方了，如果是使用的`Docker`的环境，对于`alpine`镜像可能又是另外一个位置，我们总不可能把所有的系统都熟悉一遍，或者要求所有的环境都添加一个字体库吧。如果能够我们的程序中自带这个字体就好了，这样随便你是什么系统什么环境我都使用我自己的字体，这才是最好的解决方案。所以摆在我们面前的就是两个方法解决这个问题：

- 在一个环境中的字体读取位置添加字体，例如在`操作系统的字体库`或者在`JRE`的字体库中
- 我们程序中自己自带字体库，使用我们指定位置的字体文件，例如读取一个目录下的字体，这样我们还可以使用我们自己优化过的字体更加的美观




## 三、解决办法


### 3.1、 在`JRE`中添加字体

如何在`Linux`环境下添加字体的方法网上有很多帖子都提到了，大家可以网上搜搜就可了，这里我们着重说一下如何给`JRE`添加字体，因为`JRE`的环境相对而言可控性更好点，及时在不同的`Docker`容器中我们相对而言也比较好控制。在网上搜索后发现很多的帖子都是在讲如何添加系统字体的，或者使用`第三方组件`代码中如何自定字体，其实`Oracle JDK`的官网文档中还真的有比较详细的描述如何`JRE`中添加字体的方法，[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/intl/font.html)。
如官方文档所述
> The JRE supports TrueType and PostScript Type 1 fonts.
> Physical fonts need to be installed in locations known to the Java runtime environment. The JRE looks in two locations: the lib/fonts directory within the JRE itself, and the normal font location(s) defined by the host operating system. If fonts with the same name exist in both locations, the one in the lib/fonts directory is used.
> Users can add physical fonts that use a supported font technology by installing them either in the lib/fonts directory within the JRE, or by installing them in a way supported by the host operating system. The recommended location to add per-user fonts on Solaris or Linux is the $HOME/.fonts directory which is searched by the platform's libfontconfig, and which is in turn used by the JDK.

`JDK`本身支持两种类型的字体`TrueType`和`PostScript Type 1`，所以在放字体时请注意格式问题，不是随便下载的都是可放在里面用的。`JVM`在启动是会读取`${JRE_HOME}/lib/fonts`目录下的字体和操作系统目录下的字体，当出现字体重复定义时使用`lib/fonts`目录中的字体覆盖。在实际的环境中我们也可以看到确实在该目录下放着几个默认的字体文件。所以我们可以把我们需要的字体放在`${JRE_HOME}/lib/fonts`目录下即可，但是笔者测试后发现，事情并没有那么简单，还是不可以，经过在`Windows`和`MAC`以及`Linux`下查看发现都是不可以的，笔者一度怀疑自己的英文理解能力，但是仔细往下看文档发现如下的描述
> Users can add a physical font as a fallback font to logical fonts used in Java 2D rendering by installing it in the lib/fonts/fallback directory within the JRE.

瞬间发现之前的两三个小时都在干嘛，原来`2D`的图形库自定义字体文件是放在`${JRE_HOME}/lib/fonts/fallback`，看到此处果断新建`/lib/fonts/fallback`目录，放入字体撸一把。


![](images/添加JRE环境字体.png)

如上图所示，已经成功添加字体，随后在`Docker`环境中测试也是可以的。



但是其实如果再仔细阅读文档还会发现这么一段解释

```bash
The fonts are installed in the Java SE Runtime Environment's lib/fonts directory as the following files (not all of them may be present):

LucidaSansDemiBold.ttf
LucidaSansRegular.ttf
LucidaTypewriterBold.ttf
LucidaTypewriterRegular.ttf
LucidaBrightDemiBold.ttf
LucidaBrightDemiItalic.ttf
LucidaBrightItalic.ttf
LucidaBrightRegular.ttf

```

虽然在`JRE`中放了这些字体文件，但是也不知道那些被加载了那些没有被加载。如果有看过这部分源码的同学欢迎提给`ISSUES`，我们一起学习一下。



### 3.2、 在程序中指定字体

接着我们刚才的思路，如果能够在程序中自定义一个目录，把我们的程序中的要用到的字体放在里面这样就和环境没有关系了，提高了程序的可移植性，这才是比较优雅的实现方式，答案就在`Graphics`库中，在官方文档中很快就可以找到代码[实例与说明](https://docs.oracle.com/javase/tutorial/2d/images/drawimage.html)以及关于自定义`2D`字体文件的[方法说明](https://docs.oracle.com/javase/tutorial/2d/text/fonts.html)，摘要如下
```bash
Bundling Physical Fonts with Your Application
Sometimes, an application cannot depend on a font being installed on the system, usually because the font is a custom font that is not otherwise available. In this case, you must bundle the font files with your application.

Use one of these methods to create a Font object from an existing physical font:

Font java.awt.Font.createFont(int fontFormat, InputStream in);
Font java.awt.Font.createFont(int fontFormat, File fontFile);
To create a Font object from a TrueType font, the formal parameter fontFormat must be the constant Font.TRUETYPE_FONT. The following example creates a Font object from the TrueType font file A.ttf:

Font font = Font.createFont(Font.TRUETYPE_FONT, new File("A.ttf"));
Accessing the font directly from a file is simpler and more convenient. However, you might require an InputStream object if your code is unable to access file system resources, or if the font is packaged in a Java Archive (JAR) file along with the rest of the application or applet.

The createFont method creates a new Font object with a point size of 1 and style PLAIN. This base font can then be used with the Font.deriveFont methods to derive new Font objects with varying sizes, styles, transforms and font features. For example:

try {
     //Returned font is of pt size 1
    Font font = Font.createFont(Font.TRUETYPE_FONT, new File("A.ttf"));  //在制定字体时，我们可以使用Font.createFont方法自定义一个字体文件即可

     //Derive and return a 12 pt version:
     //Need to use float otherwise
     //it would be interpreted as style

     return font.deriveFont(12f);

} catch (IOException|FontFormatException e) {
     // Handle exception
}

```

至此现在的方法已经明确了不要使用`Font font=new Font("宋体",1,24);`这种方法来指定宋体，而是使用`Font font = Font.createFont(Font.TRUETYPE_FONT, new File("A.ttf"));`这种方法来构造即可。所以代码最后变成这样

```java
import javax.imageio.ImageIO;
import java.awt.*;
import java.awt.font.*;
import java.awt.geom.Rectangle2D;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
import java.io.FileInputStream;
public class AppsData {


  public static void main(String[] args) throws IOException,FontFormatException {
    Font font=Font.createFont(Font.TRUETYPE_FONT,new File(args[2]));
    int with=300;
    int hight=300;
    BufferedImage image=new BufferedImage(with,hight,BufferedImage.TYPE_INT_ARGB);
    Graphics2D g2d=image.createGraphics();
    image=g2d.getDeviceConfiguration().createCompatibleImage(with,hight,3);
    g2d.dispose();
    g2d=image.createGraphics();
    g2d.setColor(new Color(10));
    g2d.setStroke(new BasicStroke(1.0F));
    FontRenderContext context=g2d.getFontRenderContext();
    Rectangle2D bounds=font.getStringBounds(args[0],context);
    double x=(with-bounds.getWidth())/2.0;
    double y=(hight-bounds.getHeight())/2.0;
    double ascent=-bounds.getCenterY();
    double baseY=y+ascent;
    g2d.rotate(Math.toRadians(-45.0D),with/2,hight/2);
    g2d.drawString(args[0],(int)x,(int)baseY);
    g2d.dispose();
    ImageIO.write(image,"png",new File(args[1]));
  }
}
```



经过测试也是可以的，这里还要记录一个小坑，虽然官方文档中记录了两个重载的`Font.createFont()`方法，但是只有一个方法是可以使用的，所以在使用过程中还需要注意一下

```java
Font java.awt.Font.createFont(int fontFormat, InputStream in); //这个方法是不可以使用的
Font java.awt.Font.createFont(int fontFormat, File fontFile);  //使用该方法能够成功

```



## 四、 参考文档

- [官方文档1](https://docs.oracle.com/javase/8/docs/technotes/guides/intl/font.html)
- [官方文档2](https://docs.oracle.com/javase/8/docs/technotes/guides/intl/fontconfig.html#unix)
- [官方文档3](https://docs.oracle.com/javase/tutorial/2d/text/fonts.html)
- [官方文档4](http://www.oracle.com/technetwork/articles/javase/headless-136834.html)
- [官方文档5](http://www.oracle.com/technetwork/java/javase/java8locales-2095355.html#jfc-table)
- [官方文档6](https://docs.oracle.com/javase/tutorial/2d/images/drawimage.html)



