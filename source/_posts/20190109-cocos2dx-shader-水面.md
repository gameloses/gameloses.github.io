---
title: cocos2dx水面shader
comments: true
date: 2019-01-09 18:36:38
tags:
- cocos2dx
categories:
- cocos2dx
---

#### 原理

原理还是采用2D模拟方法 利用法线贴图模拟光照 简单的实现了一下水面折射效果以及简单的UV动画

####  片段着色器

片段Shader，因为顶点Shader我们用的是Cocos自带的ccPositionTextureColor_noMVP_vert

```
//将法线贴图映射至普通的Sprite之上 简单的实现了一下水面的折射效果以及简单的normal UV动画
varying vec4 v_fragmentColor;  
varying vec2 v_texCoord;  
uniform sampler2D u_normalMap; 

vec3 waveNormal(vec2 p) {  
    ``vec3 normal = texture2D(u_normalMap, p).xyz;  
    ``normal = -1.0 + normal * 2.0;  
    ``return` `normalize(normal);  
}  
  
void` `main() {  
    ``float` `timeFactor = 0.1;  
    ``float` `offsetFactor = 0.5;  
    ``float` `refractionFactor = 0.7;  
      
    ``// simple UV animation  
    ``vec3 normal = waveNormal(v_texCoord + vec2(CC_Time.y * timeFactor, CC_Time.x * timeFactor));  
      
    ``// simple calculate refraction UV offset  
    ``vec2 p = -1 + 2 * v_texCoord;  
    ``vec3 eyePos = vec3(0, 0, 10); ``//眼睛位置 位于中心点正上方  
    ``vec3 inVec = normalize(vec3(p, 0) - eyePos);  
    ``vec3 refractVec = refract(inVec, normal, refractionFactor);  ``//根据入射向量，法线，折射系数计算折射向量
    ``vec2 v_texCoordN = v_texCoord;
    ``v_texCoordN += refractVec.xy * offsetFactor;    
    ``//v_texCoordN.x -= CC_Time.y*timeFactor *0.6; //移动水面贴图，可选
      
    ``gl_FragColor = texture2D(CC_Texture0, v_texCoordN);  
} 

```

 