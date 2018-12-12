---
title: lua实现continue
comments: true
date: 2018-12-11 12:11:32
tags:
- lua
categories:
- lua
excerpt: 实现continue,在repeat untile true 中break,实现continue的功能
---

```
for i = 1， 10 do
   repeat
     if i == 5 then
        break
     end
   until true
end

for i= 1， 10 do
   while true do
   		break
   end
end
```

满足条件时会跳出repeat的内层循环，继续外层循环相当于continue.

