---
title: cocos2dx中毒冰冻shader
comments: true
date: 2018-01-09 18:58:42
tags:
- cocos2dx
categories:
- cocos2dx
---

#### 中毒效果
![中毒](https://img-blog.csdnimg.cn/20190109190412592.jpg)
```
#ifdef GL_ES   
precision mediump float;   
#endif   
uniform sampler2D u_texture;   
varying vec2 v_texCoord;   
varying vec4 v_fragmentColor;   
void main(void)   
{   
    gl_FragColor = texture2D(u_texture, v_texCoord) * v_fragmentColor;  
    gl_FragColor.r *= 0.8;  
    gl_FragColor.r += 0.08 * gl_FragColor.a;  
    gl_FragColor.g *= 0.8;  
    gl_FragColor.b *= 0.8;  
    gl_FragColor.g += 0.2 * gl_FragColor.a;  
   //gl_FragColor= vec4(color.r,color.g, color.b,color.a) ;  
}  
```

#### 冰冻效果
![冰冻](https://img-blog.csdnimg.cn/20190109190441320.jpg)
```
#ifdef GL_ES   
precision mediump float;   
#endif   
uniform sampler2D u_texture;   
varying vec2 v_texCoord;   
varying vec4 v_fragmentColor;   
void main(void)   
{   
    vec4 normalColor = v_fragmentColor * texture2D(u_texture, v_texCoord);  
    normalColor *= vec4(0.8, 0.8, 0.8, 1);  
    normalColor.b += normalColor.a * 0.2;  
    gl_FragColor = normalColor;  
}  
```