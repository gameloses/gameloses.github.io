---
title: cocos2dx使用shader-2
comments: true
date: 2018-01-09 19:34:42
tags:
- cocos2dx
categories:
- cocos2dx
---

#### Cocos2dx的Shader架构
Cocos2dx的Shader由GLProgram、GLProgramState、GLProgramCache、GLProgramStateCache组成。 
GLProgram是Cocos2dx对Program的封装，一般提供一些静态方法，被多个节点复用。而GLProgramState是对GLProgram的封装，提供更加便捷的操作接口，内部记录了Uniform和Attribute，可以看作一个动态的对象。GLProgram和GLProgramState可以对比理解类和对象的区别。 
它们分别对应的xxCache类顾名思义用来做缓冲的，如Cocos2dx中所有内置Shader的名字，就在GLProgramCache的loadDefaultGLProgram()方法中，将所有内置的Shader都创建了出来并缓存。

#### Cocos2dx内置Shader规则
看看Cocos2dx对Shader是怎么封装的，它与一般OpenGL编写的Shader脚本不同。 
首先是GLProgram的compileShader()方法中，编译Shader前使用了precision关键字来设定数值的精度。总共有低精度lowp、中精度mediump、高精度highp可以选择，可以为浮点数或整数指定精度，精度越高效果越好，但消耗也越大。另外，顶点着色器支持的精度要比片段着色器高，片段着色器的highp精度的支持对于显卡而言，是可选的。 
然后，它还会在Shader脚本前定义一系列Uniform变量。这些都是Cocos2dx内置的Uniform，在Shader中可以直接使用。

```
bool GLProgram::compileShader(GLuint* shader, GLenum type, const GLchar* source, const std::string& convertedDefines)
{
    GLint status;

    if (!source)
    {
        return false;
    }
    
    const GLchar *sources[] = {
#if CC_TARGET_PLATFORM == CC_PLATFORM_WINRT
        (type == GL_VERTEX_SHADER ? "precision mediump float;\n precision mediump int;\n" : "precision mediump float;\n precision mediump int;\n"),
#elif (CC_TARGET_PLATFORM != CC_PLATFORM_WIN32 && CC_TARGET_PLATFORM != CC_PLATFORM_LINUX && CC_TARGET_PLATFORM != CC_PLATFORM_MAC)
        (type == GL_VERTEX_SHADER ? "precision highp float;\n precision highp int;\n" : "precision mediump float;\n precision mediump int;\n"),
#endif
        COCOS2D_SHADER_UNIFORMS,
        convertedDefines.c_str(),
        source};
    
    *shader = glCreateShader(type);
    glShaderSource(*shader, sizeof(sources)/sizeof(*sources), sources, nullptr);
    glCompileShader(*shader);
    
    glGetShaderiv(*shader, GL_COMPILE_STATUS, &status);
    
    if (! status)
    {
        GLsizei length;
        glGetShaderiv(*shader, GL_SHADER_SOURCE_LENGTH, &length);
        GLchar* src = (GLchar *)malloc(sizeof(GLchar) * length);
    
        glGetShaderSource(*shader, length, nullptr, src);
        CCLOG("cocos2d: ERROR: Failed to compile shader:\n%s", src);
    
        if (type == GL_VERTEX_SHADER)
        {
            CCLOG("cocos2d: %s", getVertexShaderLog().c_str());
        }
        else
        {
            CCLOG("cocos2d: %s", getFragmentShaderLog().c_str());
        }
        free(src);
    
        return false;
    }
    
    return (status == GL_TRUE);
}

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
在初始化GLProgram时，Cocos2dx会自动根据Shader是否使用了相应的Uniform变量来初始化_flags成员变量相对应的属性。在编译Shader时，GLSL的编译器会自动将没有使用到的Uniform移除(用glGetUniformLocation()方法判断，返回-1就是没有使用)。

```
void GLProgram::updateUniforms()
{
​    _builtInUniforms[UNIFORM_AMBIENT_COLOR] = glGetUniformLocation(_program, UNIFORM_NAME_AMBIENT_COLOR);
​    _builtInUniforms[UNIFORM_P_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_P_MATRIX);
​    _builtInUniforms[UNIFORM_MV_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_MV_MATRIX);
​    _builtInUniforms[UNIFORM_MVP_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_MVP_MATRIX);
​    _builtInUniforms[UNIFORM_NORMAL_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_NORMAL_MATRIX);
    _builtInUniforms[UNIFORM_TIME] = glGetUniformLocation(_program, UNIFORM_NAME_TIME);
    _builtInUniforms[UNIFORM_SIN_TIME] = glGetUniformLocation(_program, UNIFORM_NAME_SIN_TIME);
    _builtInUniforms[UNIFORM_COS_TIME] = glGetUniformLocation(_program, UNIFORM_NAME_COS_TIME);
    
    _builtInUniforms[UNIFORM_RANDOM01] = glGetUniformLocation(_program, UNIFORM_NAME_RANDOM01);
    
    _builtInUniforms[UNIFORM_SAMPLER0] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER0);
    _builtInUniforms[UNIFORM_SAMPLER1] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER1);
    _builtInUniforms[UNIFORM_SAMPLER2] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER2);
    _builtInUniforms[UNIFORM_SAMPLER3] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER3);
    
    _flags.usesP = _builtInUniforms[UNIFORM_P_MATRIX] != -1;
    _flags.usesMV = _builtInUniforms[UNIFORM_MV_MATRIX] != -1;
    _flags.usesMVP = _builtInUniforms[UNIFORM_MVP_MATRIX] != -1;
    _flags.usesNormal = _builtInUniforms[UNIFORM_NORMAL_MATRIX] != -1;
    _flags.usesTime = (
                       _builtInUniforms[UNIFORM_TIME] != -1 ||
                       _builtInUniforms[UNIFORM_SIN_TIME] != -1 ||
                       _builtInUniforms[UNIFORM_COS_TIME] != -1
                       );
    _flags.usesRandom = _builtInUniforms[UNIFORM_RANDOM01] != -1;
    
    this->use();
    
    // Since sample most probably won't change, set it to 0,1,2,3 now.
    if(_builtInUniforms[UNIFORM_SAMPLER0] != -1)
       setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER0], 0);
    if(_builtInUniforms[UNIFORM_SAMPLER1] != -1)
        setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER1], 1);
    if(_builtInUniforms[UNIFORM_SAMPLER2] != -1)
        setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER2], 2);
    if(_builtInUniforms[UNIFORM_SAMPLER3] != -1)
        setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER3], 3);
}
```
#### 这些Uniform是什么时候被设置的呢？它们的值又是多少呢？ 
如果是纹理Uniform，在一开始就会被设置为固定的值，上面源码中也可以看到，同时可以使用4个纹理，在渲染的时候，Cocos2dx会将纹理绑定到指定的纹理采样器中，我们在Shader脚本中可以访问这些采样器，而采样器对应的纹理，由Cocos2dx调用OpenGL的glBindTexture()等方法绑定，因此这个Uniform的值不需要我们设置。 
而其它的如矩阵、时间、法线、随机数等Uniform，是在运行的时候动态设置的，这些属性是实时变化的，在程序运行时，Cocos2dx会每帧调用setUniformsForBuiltins()自动设置它们。

```
void GLProgram::setUniformsForBuiltins(const Mat4 &matrixMV)
{
​    const auto& matrixP = _director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION)
    if (_flags.usesP) setUniformLocationWithMatrix4fv(_builtInUniforms[UNIFORM_P_MATRIX], matrixP.m, 1);
    if (_flags.usesMV) setUniformLocationWithMatrix4fv(_builtInUniforms[UNIFORM_MV_MATRIX], matrixMV.m, 1);
    
    if (_flags.usesMVP)
    {
        Mat4 matrixMVP = matrixP * matrixMV;
        setUniformLocationWithMatrix4fv(_builtInUniforms[UNIFORM_MVP_MATRIX], matrixMVP.m, 1);
    }
    if (_flags.usesNormal)
    {
        Mat4 mvInverse = matrixMV;
        mvInverse.m[12] = mvInverse.m[13] = mvInverse.m[14] = 0.0f;
        mvInverse.inverse();
        mvInverse.transpose();
        GLfloat normalMat[9];
        normalMat[0] = mvInverse.m[0];normalMat[1] = mvInverse.m[1];normalMat[2] = mvInverse.m[2];
        normalMat[3] = mvInverse.m[4];normalMat[4] = mvInverse.m[5];normalMat[5] = mvInverse.m[6];
        normalMat[6] = mvInverse.m[8];normalMat[7] = mvInverse.m[9];normalMat[8] = mvInverse.m[10];
        setUniformLocationWithMatrix3fv(_builtInUniforms[UNIFORM_NORMAL_MATRIX], normalMat, 1);
    }
    
    if (_flags.usesTime) {
        // This doesn't give the most accurate global time value.
        // Cocos2D doesn't store a high precision time value, so this will have to do.
        // Getting Mach time per frame per shader using time could be extremely expensive.
        float time = _director->getTotalFrames() * _director->getAnimationInterval();
    
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_TIME], time/10.0, time, time*2, time*4);
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_SIN_TIME], time/8.0, time/4.0, time/2.0, sinf(time));
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_COS_TIME], time/8.0, time/4.0, time/2.0, cosf(time));
    }
    
    if (_flags.usesRandom)
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_RANDOM01], CCRANDOM_0_1(), CCRANDOM_0_1(), CCRANDOM_0_1(), CCRANDOM_0_1());
}
```
#### 除了以上内置的Uniform外，Cocos2dx还内置了一些Attribute，

但这些变量是需要在Shader脚本中手动输入的，主要是颜色、坐标、纹理坐标、法线等常用属性。它们被定义在CCGLProgram.cpp中，在Renderer进行渲染时，会调用glVertexAttribPointer()设置它们。

```
// Attribute names
const char* GLProgram::ATTRIBUTE_NAME_COLOR = "a_color";
const char* GLProgram::ATTRIBUTE_NAME_POSITION = "a_position";
const char* GLProgram::ATTRIBUTE_NAME_TEX_COORD = "a_texCoord";
const char* GLProgram::ATTRIBUTE_NAME_TEX_COORD1 = "a_texCoord1";
const char* GLProgram::ATTRIBUTE_NAME_TEX_COORD2 = "a_texCoord2";
const char* GLProgram::ATTRIBUTE_NAME_TEX_COORD3 = "a_texCoord3";
const char* GLProgram::ATTRIBUTE_NAME_NORMAL = "a_normal";
const char* GLProgram::ATTRIBUTE_NAME_BLEND_WEIGHT = "a_blendWeight";
const char* GLProgram::ATTRIBUTE_NAME_BLEND_INDEX = "a_blendIndex";
const char* GLProgram::ATTRIBUTE_NAME_TANGENT = "a_tangent";
const char* GLProgram::ATTRIBUTE_NAME_BINORMAL = "a_binormal";

整个Cocos2dx的Shader不同的规则大体就是这样了，主要是内置了一系列的变量，由引擎自动设置值，我们可以在自己Shader中直接使用。

Cocos2dx中使用Shader示例
我们就编写一个常用的灰化Shader来演示下Cocos2dx中使用Shader的流程。

灰化原理：灰白图片的特征就是每个像素的RGB三个分量的值均相等，值越高颜色越白，反之颜色越黑。常用的算法有3种，平均值、最大值和加权平均值。

首先，我们得分别编写顶点Shader和片段Shader：

//顶点Shader
attribute vec4 a_position;   //位置(内置)
attribute vec2 a_texCoord;   //纹理坐标(内置)
varying vec2 v_texCoord;     //自定义易变变量(写入值)
void main()
{
    gl_Position = CC_PMatrix * a_position;
    v_texCoord = a_texCoord;

//片段着色器
//使用加权平均值算RGB分量，即给每个分量指定一个比例，指定比例没有写死，通过一个Uniform变量来设置
varying vec2 v_texCoord;    //接收顶点着色器传递的值(接收值)
uniform vec4 u_grayParam;   //指定rgb颜色分量的比例
void main()
{
    vec4 texColor = texture2D(CC_Texture0, v_texCoord);
    texColor.rgb = texColor.r * u_grayParam.r + texColor.g * u_grayParam.g + texColor.b * u_grayParam.b;
    gl_FragColor = texColor;

分别命名为gray.vert和gray.frag保存到磁盘。 
好，脚本写完了，下面来看看怎么使用：

#include "renderer/CCGLProgramStateCache.h"
//置灰接口
void grayNode(Node* node)
{
    //对应xxCache的运用，先成缓存中找，不需要每次都重新创建
    GLProgram* program = GLProgramCache::getInstance()->getGLProgram("MyGrayShader");
    if(nullptr == program)
    {
        //本地磁盘加载shader创建program,还一种字符串的形式效率要高createWithByteArrays()
        program = GLProgram::createWithFilenames("gray.vert", "gray.frag");
        //将GLProgram对象缓存到Cache中
        GLProgramCache::getInstance()->addGLProgram(program, "MyGrayShader");
    }
    //通过program创建programState,用它来设置上面自定义的Uniform变量的值
    GLProgramState* programState = GLProgramState::getOrCreateWithGLProgram(program);
    programState->setUniformVec4("u_grayParam", Vec4(0.2f,0.3f,0.5f,1.0f));
    //调用node节点的setGLProgramState，以后此节点渲染就会使用这个Shader了
    node->setGLProgramState(programState);
//使用
auto sprite = Sprite::create(xx.png);
grayNode(sprite);
```

至此，Cocos2dx中使用Shader流程就讲完了～ 
最后说下，怎么把置灰的图片再还原回去呢？OpenGL是个状态机的运行机制，所以只需要把设置的Shader还原回去重新设置就行了。这里Sprite对象默认的Shader是GLProgram::SHADER_NAME_POSITION_TEXTRUE_COLOR_NO_MVP，可以通过GLProgramState::getOrCreateWithGLProgramName()传入上面字符串常量来获取它的GLProgramState.
