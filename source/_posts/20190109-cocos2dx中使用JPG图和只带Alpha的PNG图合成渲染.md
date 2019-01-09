---
title: cocos2dx中使用JPG图和只带Alpha的PNG图合成渲染
comments: true
date: 2019-01-09 19:19:04
tags:
- cocos2dx
categories:
- cocos2dx
---

#### 原理

将带Alpha通道的PNG图片分拆成RGB和Alpha分别保存，其中保存RGB的这张图把它转换成JPG格式的文件，保存Alpha图片的就用PNG格式的。原理是JPG格式的压缩率比较的高可以减小图片的大小，但是它没有Alpha，而Alpha数据单独拎出来一般比较小，所以直接用PNG格式来保存。问题是怎样分拆PNG图片，又怎样在程序中怎样将两张图片的数据合并起来以达到和直接用一张带Alpha的PNG图渲染出来是一样的效果。

### 拆分

将PNG图片分拆，也就是将一张PNG图片生成出一张带RGB的JPG+一张带Alpha的PNG，我使用的工具是imagemagick。这里以bg.png为例:　　　　

- 首先ImageMagick工具提取出Alpha通道，(命令: convert bg.png -channel A -alpha extract bgAlpha.png)　　　　

- 然后在将bg.png转成JPG格式 输出为bg.jpg。jpg格式已经不包含Alpha通道，而且jpg的压缩率比较高（这是采用这种方法可以减小图片大小的根本所在）。(convert bg.png bg.jpg)

- 这样就得到了bg.jpg 和bgAlpha.png,这两张图片就是我们需要的资源。这样转换之后bg.jpg+bgAlpha.png的大小大概比bg.png的 小2~3倍

#### 合并

由于在win，Android平台采用的JPG和PNG解析库和IOS上的不同，所以在程序中要分别处理，这里介绍下在IOS上的方法(win和Android平台解析参照CCImage的_initWithJpgData和_initWithPngData 方法,合并和IOS的平台类似,不同地方文中会指出), 这个方法添加在CCImage.mm 文件中即可使用

```
bool CCImage::initWithRgbAndAlpha(const char * jpgFilePath, const char * pngFilePath)
 {
     bool bRet = false;
     
    do {
     
         // 获得JPG文件的数据
         unsigned long nSizeJpg = 0;
         unsigned char* pBufferJpg = CCFileUtils::sharedFileUtils()->getFileData(
                                         CCFileUtils::sharedFileUtils()->fullPathForFilename(jpgFilePath).c_str(),
                                         "rb",
                                         &nSizeJpg);
         // 保证JPG文件数据必须是存在的
         if (pBufferJpg == NULL && nSizeJpg <= 0) {return true;}
         
         
         // 获得Alpha文件的数据
         unsigned long nSizePng = 0;
         unsigned char* pBufferPng = CCFileUtils::sharedFileUtils()->getFileData(
                                         CCFileUtils::sharedFileUtils()->fullPathForFilename(pngFilePath).c_str(),
                                         "rb",
                                         &nSizePng);
         
         tImageInfo jpgInfo = {0};
         
         jpgInfo.hasShadow = false;
         jpgInfo.hasStroke = false;
         
         
         // 利用CCImage原有的方法解析JPG
         bRet = _initWithData(pBufferJpg, nSizeJpg, &jpgInfo);
         unsigned char* jpgData = jpgInfo.data; 
         
         if (!bRet) {return false;}
         
         
         tImageInfo pngInfo = {0};
         
         pngInfo.hasShadow = false;
         pngInfo.hasStroke = false;
         
         //利用CCImage原有的方法解析Alpha文件  
         bRet = _initWithData(pBufferPng, nSizePng, &pngInfo);
         unsigned char* pngData = pngInfo.data;
         
         if (!bRet) {return false;}
         
       
         unsigned short jpgHeight = (short)jpgInfo.height;
         unsigned short jpgWidth = (short)jpgInfo.width;
         
         int jpgDataLen = jpgHeight * jpgWidth;
         
         int pngIndex = 0;
         int jpgIndex = 0;
         
         m_pData = new unsigned char[jpgHeight * jpgWidth * 4];
         unsigned int *tmpData = (unsigned int *)m_pData;
         
         // 合并两个文件的数据
         for(int i = 0; i < jpgDataLen; i++)
         { 
// 这个宏和CC_RGB_PREMULTIPLY_ALPHA是一样，作用是将R,G,B 三个颜色值分别乘Alpha值得到合并的R,G,B值, 
             *tmpData++ = CC_RGB_PREMULTIPLY_ALPHA_IOS(
                                 jpgData[jpgIndex], // R
                                 jpgData[jpgIndex + 1], // G
                                 jpgData[jpgIndex + 2], // B
                                 pngData[pngIndex]);  //A
 
             // 两张图片在解析之后都为32位了，所以+4 (Android或者win上 不带Alpha通道的png文件和jpg文件解析之后是24位的，这是主要区别的地方)
             jpgIndex += 4;
             pngIndex += 4;
         }
         
         m_nHeight = (short)jpgInfo.height;
         m_nWidth = (short)jpgInfo.width;
         
         m_nBitsPerComponent = jpgInfo.bitsPerComponent;
         m_bPreMulti = true;
         m_bHasAlpha = true;
         
         CC_SAFE_DELETE_ARRAY(pBufferPng);
         CC_SAFE_DELETE_ARRAY(pBufferJpg);
         CC_SAFE_DELETE_ARRAY(jpgData);
         CC_SAFE_DELETE_ARRAY(pngData);
         
         bRet = true;
         
     } while (0);
     
     return bRet;
}
```

