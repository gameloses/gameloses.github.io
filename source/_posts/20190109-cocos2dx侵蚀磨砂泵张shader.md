---
title: cocos2dx侵蚀膨胀磨砂shader
comments: true
date: 2018-01-09 19:12:04
tags:
- cocos2dx
categories:
- cocos2dx
---

```
// 磨砂
static` `const` `GLchar* s_szSharpenFSH =
    ``"                                                               \n\
    ``precision mediump ``float``;                                        \n\
    ``uniform sampler2D u_Texture;                                    \n\
    ``uniform vec2 u_TextureCoordOffset[25];                          \n\
    ``varying vec2 v_texCoord;                                        \n\
    ``varying vec4 v_fragmentColor;                                   \n\
                                                                    ``\n\
    ``void` `main(``void``)                                                 \n\
    ``{                                                               \n\
        ``vec4 sample[25];                                            \n\
                                                                    ``\n\
        ``for` `(``int` `i = 0; i < 25; i++)                             \n\
        ``{                                                           \n\
            ``sample[i] = texture2D(u_Texture,                        \n\
                ``v_texCoord.st + u_TextureCoordOffset[i]);           \n\
        ``}                                                           \n\
        ``//--------------------------------------------------------  \n\
        ``//   -1 -1 -1                                               \n\
        ``//   -1  9 -1                                               \n\
        ``//   -1 -1 -1                                               \n\
                                                                    ``\n\
        ``//gl_FragColor = (sample[4] * 9.0) -                        \n\
        ``//  (sample[0] + sample[1] + sample[2] +                    \n\
        ``//  sample[3] + sample[5] +                                 \n\
        ``//  sample[6] + sample[7] + sample[8]);                     \n\
        ``//--------------------------------------------------------  \n\
        ``// Sharpen weighting:                                       \n\
        ``//  0 -1 -1 -1  0                                           \n\
        ``// -1  2 -4  2 -1                                           \n\
        ``// -1 -4 13 -4 -1                                           \n\
        ``// -1  2 -4  2 -1                                           \n\
        ``//  0 -1 -1 -1  0                                           \n\
                                                                    ``\n\
        ``//gl_FragColor = (                                          \n\
        ``//(-1.0  * ( sample[1] + sample[2]  + sample[3] + sample[5] +   \n\
        ``//sample[9] + sample[10] + sample[14] + sample[15] +            \n\
        ``//sample[19] + sample[21] + sample[22] + sample[23]) ) +        \n\
        ``//(2.0 * (sample[6] + sample[8] + sample[16] + sample[18])) +   \n\
        ``//(-4.0 *(sample[7] + sample[11] + sample[13] + sample[17]))+   \n\
        ``//( 13.0 * sample[12] )                                     \n\
        ``//);                                                            \n\
                                                                                        ``\n\
        ``// 1  1  1  1  1                                                                \n\
        ``// 1  1  1  1  1                                                                \n\
        ``// 1  1 -14 1  1                                                                \n\
        ``// 1  1  1  1  1                                                                \n\
        ``// 1  1  1  1  1                                                                \n\
                                                                                        ``\n\
        ``gl_FragColor = -14.0 * sample[12];                                              \n\
                                                                                        ``\n\
        ``for` `(``int` `i = 0; i < 25; i++)                                                 \n\
        ``{                                                                               \n\
        ``if` `(i != 12)                                                                    \n\
        ``gl_FragColor += sample[i];                                                      \n\
        ``}                                                                               \n\
        ``gl_FragColor /= 14.0;                                                           \n\
    ``}";
//--------------------------------------------------------        
// 膨胀
static` `const` `GLchar* s_szDilateFSH =
    ``"                                                               \n\
    ``precision mediump ``float``;                                        \n\
    ``uniform sampler2D u_Texture;                                    \n\
    ``uniform vec2 u_TextureCoordOffset[25];                          \n\
    ``varying vec2 v_texCoord;                                        \n\
    ``varying vec4 v_fragmentColor;                                   \n\
                                                                    ``\n\
    ``void` `main(``void``)                                                 \n\
    ``{                                                               \n\
    ``vec4 sample[25];                                                \n\
    ``vec4 maxValue = vec4(0.0);                                                          \n\
                                                                                        ``\n\
    ``for` `(``int` `i = 0; i < 25; i++)                                                     \n\
    ``{                                                                                   \n\
        ``// Sample a grid around and including our texel                                 \n\
        ``sample[i] = texture2D(u_Texture, v_texCoord.st + u_TextureCoordOffset[i]);      \n\
        ``// Keep the maximum value                                                       \n\
        ``maxValue = max(sample[i], maxValue);                                            \n\
    ``}                                                                                   \n\
                                                                                        ``\n\
    ``gl_FragColor = maxValue;                                                            \n\
    ``}";
//-------------------------------------------------------- 
// 侵蚀效果
static` `const` `GLchar* s_szErodeFSH =
    ``"                                                               \n\
    ``precision mediump ``float``;                                        \n\
    ``uniform sampler2D u_Texture;                                    \n\
    ``uniform vec2 u_TextureCoordOffset[25];                          \n\
    ``varying vec2 v_texCoord;                                        \n\
    ``varying vec4 v_fragmentColor;                                   \n\
                                                                    ``\n\
    ``void` `main(``void``)                                                 \n\
    ``{                                                               \n\
    ``vec4 sample[25];                                                                \n\
    ``vec4 minValue = vec4(1.0);                                                      \n\
                                                                                    ``\n\
    ``for` `(``int` `i = 0; i < 25; i++)                                                 \n\
    ``{                                                                               \n\
        ``// Sample a grid around and including our texel                             \n\
        ``sample[i] = texture2D(u_Texture, v_texCoord.st + u_TextureCoordOffset[i]);  \n\
        ``// Keep the minimum value                                                   \n\
        ``minValue = min(sample[i], minValue);                                        \n\
    ``}                                                                               \n\
                                                                                    ``\n\
    ``gl_FragColor = minValue;                                                        \n\
    ``}";
```