---
title: cocos2dx使用shader
comments: true
date: 2018-01-09 18:43:08
tags:
- cocos2dx
categories:
- cocos2dx
---

#### 着色器

认识.vsh,.fsh 这两个文件在被编译和链接后就可以产生可执行程序与GPU交互。

 .vsh 是 vertex shader，用与顶点计算，可以理解为控制顶点的位置，在这个文件中我们通常会传入当前顶点的位置，和纹理的坐标。 

.fsh 
是片段shader，或者叫片元shader。在这里面我可以对于每一个像素点进行重新计算。

#### .vsh和.fsh在opengl 渲染的流程


着色器对象关联着色器代码，glShaderSource 
把着色器源代码编译成目标代码，glCompileShader 
验证着色器是否已经变异通过， glGetShaderivgl GetShaderInfoLog 
创建一个着色器程序，glCreatePragram 
把着色器链接到着色器程序中， glAttachShader 
链接着色器程序， glLinkProgram 
验证着色器程序是否链接成功， glGetProgramiv glGetProgramInfoLog

#### cocos2dx中使用shader

auto sprite = Sprite::create("afei.png"); 
sprite->setPosition(vec2(300, 400)); fNode->addChild(sprite); auto 

//创建program

program = GLProgram::createWithFilenames("afei.vsh", "afei.fsh");

//绑定内置统一变量的位置  
program->bindAttribLocation(GLProgram::ATTRIBUTE_NAME_POSITION, 
GLProgram::VERTEX_ATTRIB_POSITION); 
program->bindAttribLocation(GLProgram::ATTRIBUTE_NAME_COLOR, 
GLProgram::VERTEX_ATTRIB_COLOR); 
program->bindAttribLocation(GLProgram::ATTRIBUTE_NAME_TEX_COORD, 
GLProgram::VERTEX_ATTRIB_TEX_COORD); 

//链接着色器

program->link();

//定义内置的统一变量

program->updateUniforms();

 sprite->setGLProgram(program); 
简单解释下，每个Node 对象都会 有一个GLProgram 
对象，设定一系列的OPGL参数之后，在Node渲染函数里面进行渲染，无外乎就是颜色，位置，透明度，uv等一系列的变换。 
比如dota传奇里面的冰霜效果，具体的渲染原理就是，顶点不变，将纹理（对应着图片）的颜色值改变，说白了就是修改r,g,b值。