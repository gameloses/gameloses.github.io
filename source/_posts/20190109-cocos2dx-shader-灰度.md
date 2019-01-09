---
title: cocos2dx灰度shader
comments: true
date: 2018-12-31 17:57:11
tags:
- cocos2dx
categories:
- cocos2dx
---

### 灰度shader

最近在学习shader，就把cocos2d-x 3.x版本中的很简单也很常用的灰度shader拿出来学习一下。

```
#ifdef GL_ES
precision mediump float; // ES版本的精度限定符，精度变低后可以提高效率
#endif

varying vec4 v_fragmentColor;
varying vec2 v_texCoord;

void main(void)
     vec4 c = texture2D(CC_Texture0, v_texCoord);
     gl_FragColor.xyz = vec3(0.2126*c.r + 0.7152*c.g + 0.0722*c.b);
     gl_FragColor.w = c.w;
}
```

### 代码分析

`precision mediump float`是open es特有的精度限定符，原本的浮点数精度是`double`，opengl es为了提高渲染效率，限定精度为`float`类型。

`v_fragmentColor`是从顶点着色器设置的颜色经过光栅化阶段的线性插值后传给片段着色器的颜色。

`v_texCoord`同样是经过线性插值而来的纹理坐标。

下面是顶点着色器的代码：

```
const char* ccPositionTextureColor_noMVP_vert = STRINGIFY(
attribute vec4 a_position;
attribute vec2 a_texCoord;
attribute vec4 a_color;

\n#ifdef GL_ES\n
varying lowp vec4 v_fragmentColor;
varying mediump vec2 v_texCoord;
\n#else\n
varying vec4 v_fragmentColor;
varying vec2 v_texCoord;
\n#endif\n

    void main()
    {
        gl_Position = CC_PMatrix * a_position;
        v_fragmentColor = a_color;
        v_texCoord = a_texCoord;
    }
);
```

`CC_Texture0`是一个采样器，在load shader的时候，引擎会预先把这些`uniform`变量给加载进来。
下面这部分代码就是引擎预先加载进来的`uniform`变量：

```
static const char * COCOS2D_SHADER_UNIFORMS =
        "uniform mat4 CC_PMatrix;\n"
        "uniform mat4 CC_MVMatrix;\n"
        "uniform mat4 CC_MVPMatrix;\n"
        "uniform mat3 CC_NormalMatrix;\n"
        "uniform vec4 CC_Time;\n"
        "uniform vec4 CC_SinTime;\n"
        "uniform vec4 CC_CosTime;\n"
        "uniform vec4 CC_Random01;\n"
        "uniform sampler2D CC_Texture0;\n"
        "uniform sampler2D CC_Texture1;\n"
        "uniform sampler2D CC_Texture2;\n"
        "uniform sampler2D CC_Texture3;\n"
        "//CC INCLUDES END\n\n";
```

这些变量在shader里面如果没有用到的话，会被引擎给优化掉。

`texture2D()`是shader的内建方法，作用是从`CC_Texture0`采样器中进行纹理采样，得到当前片段的颜色值。

`gl_FragColor`是shader的内建变量，表示当前片段的颜色，代码中是把从采样器中拿到的颜色值进行一个变灰处理后，最后得到的颜色值再赋值给`gl_FragColor`。`gl_FragColor`就是最终的颜色。

这个shader很简单，就是改变了一下rgb的值。`0.2126`，`0.7152`，`0.0722`这几个系数据说是根据人眼对rgb这三种基本颜色识别的强弱算出来的。

### 使用示例

在cocos2dx 3.x版本中sprite变灰的代码例子：

```
auto sprite = Sprite::create("HelloWorld.png");
sprite->setGLProgramState(GLProgramState::getOrCreateWithGLProgramName(GLProgram::SHADER_NAME_POSITION_GRAYSCALE));
```

效果如下图所示：

![灰度shader](https://img-blog.csdnimg.cn/20190109181721144.png)