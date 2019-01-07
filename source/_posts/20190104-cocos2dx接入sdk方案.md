---
title: cocos2dx接入sdk方案
comments: true
date: 2017-01-04 18:55:46
tags:
- cocos2dx
categories:
- cocos2dx
---

使用jni java反射机制 和callfunc

jni:java c++相互调用。

反射机制：主要使用Class Filed Constructor来动态的获取类的方法属性构造器进一步调用对象的方法属性以及构造对象。

callfunc：函数调用。



java->c++ : 

通过jni调用CPPNativeCallHandler 调用c++的 NDKHelper::handleMessage 通过查找名字管理的callfunc最终调用callfunc.execute()执行c++方法。



c++调用java:

通过jni调用ReceiveCppMessage：利用反射机制实现方法调用。

最终实现的结果类型一种消息机制，省去类写jni的大量流程。



```
//  NDKMessage.java 

package com.easyndk.classes;
import java.lang.reflect.Method;
import org.json.JSONObject;

public class NDKMessage
{
   public Method methodToCall;
   public JSONObject methodParams;
}

//AndroidNDKHelper 

package com.easyndk.classes;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import org.json.JSONException;
import org.json.JSONObject;
import android.os.Handler;
import android.os.Message;

public class AndroidNDKHelper {

 private static Object callHandler = null;
 private static Handler NDKHelperHandler = null;

 private static native void CPPNativeCallHandler(String json);    //调用c++方法
 private static String __CALLED_METHOD__                 = "calling_method_name";
 private static String __CALLED_METHOD_PARAMS__ = "calling_method_params";
 private final  static int __MSG_FROM_CPP__ = 5;

//设置java层接受c++方法调用的类
 public static void SetNDKReceiver(Object callReceiver)
 {
    callHandler = callReceiver;
    NDKHelperHandler = new Handler()
   {
          public void handleMessage(Message msg)
         {
            switch(msg.what)
            {
            case __MSG_FROM_CPP__:
               NDKMessage message = (NDKMessage)msg.obj;
               message.methodToCall.invoke(AndroidNDKHelper.callHandler,  message.methodParams);
            }
            break;
             }
        }
    };
 }

//java 调用c++方法
 public static void SendMessageWithParameters(String methodToCall, JSONObject paramList)
 {
      JSONObject obj = new JSONObject();
       obj.put(__CALLED_METHOD__, methodToCall);
       obj.put(__CALLED_METHOD_PARAMS__, paramList);
       CPPNativeCallHandler(obj.toString());
 }

//接受c++调用 封装成method 进一步发送消息到接受的类中调用
 public static void ReceiveCppMessage(String json)
 {
      if (json != null)
      {
                JSONObject obj = new JSONObject(json);
                if (obj.has(__CALLED_METHOD__))
                {
                     String methodName = obj.getString(__CALLED_METHOD__);
                     JSONObject methodParams = null;
                     methodParams = obj.getJSONObject(__CALLED_METHOD_PARAMS__);

                    Method m = AndroidNDKHelper.callHandler.getClass().getMethod(methodName,
                                          new Class[] { JSONObject.class });

                    NDKMessage message = new NDKMessage();
                    message.methodToCall = m;
                    message.methodParams = methodParams;

                   Message msg = new Message();
                   msg.what = __MSG_FROM_CPP__;
                   msg.obj = message;

                   AndroidNDKHelper.NDKHelperHandler.sendMessage(msg);
              }
     }
}


--------------------------------------------------------------------------------------------------------
NDKHelper.h

#include "cocos2d.h"
#include "jansson\jansson.h"
#include "NDKCallbackNode.h"

class NDKHelper
{
public:
    static void addSelector(const char *groupName, const char *name, FuncNV selector, cocos2d::Node *target);
    static void printSelectorList();
    static void removeSelectorsInGroup(const char *groupName);
    
    static cocos2d::Value getValueFromJson(json_t *obj);
    static json_t *getJsonFromValue(cocos2d::Value value);
    
    static void handleMessage(json_t *methodName, json_t *methodParams);
    
private:
    static std::vector<NDKCallbackNode> selectorList;
};

extern "C"
{
    void sendMessageWithParams(std::string methodName, cocos2d::Value methodParams);
}

NDKHelper.cpp

#include "NDKHelper.h"

USING_NS_CC;

#define __CALLED_METHOD__           "calling_method_name"
#define __CALLED_METHOD_PARAMS__    "calling_method_params"

std::vector<NDKCallbackNode> NDKHelper::selectorList;

void NDKHelper::addSelector(const char *groupName, const char *name, FuncNV selector, Node *target)
{
    NDKHelper::selectorList.push_back(NDKCallbackNode(groupName, name, selector, target));
}

void NDKHelper::printSelectorList()
{
    for (unsigned int i = 0; i < NDKHelper::selectorList.size(); ++i) {
        std::string s = NDKHelper::selectorList[i].getGroup();
        s.append(NDKHelper::selectorList[i].getName());
    }
}

void NDKHelper::removeSelectorsInGroup(const char *groupName)
{
    std::vector<int> markedIndices;
    
    for (unsigned int i = 0; i < NDKHelper::selectorList.size(); ++i) {
        if (NDKHelper::selectorList[i].getGroup().compare(groupName) == 0) {
            markedIndices.push_back(i);
        }
    }
    
    for (long i = markedIndices.size() - 1; i >= 0; --i) {
        NDKHelper::selectorList.erase(NDKHelper::selectorList.begin() + markedIndices[i]);
    }
}

Value NDKHelper::getValueFromJson(json_t *obj)
{
    if (obj == NULL) {
        return Value::Null;
    }
    
    if (json_is_object(obj)) {
        ValueMap valueMap;
        
        const char *key;
        json_t *value;
        
        void *iter = json_object_iter(obj);
        while (iter) {
            key = json_object_iter_key(iter);
            value = json_object_iter_value(iter);
            
            valueMap[key] = NDKHelper::getValueFromJson(value);
            
            iter = json_object_iter_next(obj, iter);
        }
        
        return Value(valueMap);
    } else if (json_is_array(obj)) {
        ValueVector valueVector;
        
        size_t sizeArray = json_array_size(obj);
        
        for (unsigned int i = 0; i < sizeArray; i++) {
            valueVector.push_back(NDKHelper::getValueFromJson(json_array_get(obj, i)));
        }
        
        return Value(valueVector);
    } else if (json_is_boolean(obj)) {
        if (json_is_true(obj)) {
            return Value(true);
        } else {
            return Value(false);
        }
    } else if (json_is_integer(obj)) {
        int value = (int) json_integer_value(obj);
        
        return Value(value);
    } else if (json_is_real(obj)) {
        double value = json_real_value(obj);
        
        return Value(value);
    } else if (json_is_string(obj)) {
        std::string value = json_string_value(obj);
        
        return Value(value);
    }
    
    return Value::Null;
}

json_t *NDKHelper::getJsonFromValue(Value value)
{
    if (value.getType() == Value::Type::MAP) {
        ValueMap valueMap = value.asValueMap();
        
        json_t *jsonDict = json_object();
        
        for (auto &element : valueMap) {
            json_object_set_new(jsonDict, element.first.c_str(),
                                NDKHelper::getJsonFromValue(element.second));
        }
        
        return jsonDict;
    } else if (value.getType() == Value::Type::VECTOR) {
        ValueVector valueVector = value.asValueVector();
        
        json_t *jsonArray = json_array();
        
        size_t sizeVector = valueVector.size();
        
        for (unsigned int i = 0; i < sizeVector; i++) {
            json_array_append_new(jsonArray,
                                  NDKHelper::getJsonFromValue(valueVector.at(i)));
        }
        
        return jsonArray;
    } else if (value.getType() == Value::Type::BOOLEAN) {
        return json_boolean(value.asBool());
    } else if (value.getType() == Value::Type::INTEGER) {
        return json_integer(value.asInt());
    } else if (value.getType() == Value::Type::DOUBLE) {
        return json_real(value.asDouble());
    } else if (value.getType() == Value::Type::STRING) {
        return json_string(value.asString().c_str());
    }
    
    return NULL;
}

//接受java层调用
void NDKHelper::handleMessage(json_t *methodName, json_t *methodParams)
{
    if (methodName == NULL) {
        return;
    }
    
    const char *methodNameStr = json_string_value(methodName);
    
    for (unsigned int i = 0; i < NDKHelper::selectorList.size(); ++i) {
        if (NDKHelper::selectorList[i].getName().compare(methodNameStr) == 0) {
            Value value = NDKHelper::getValueFromJson(methodParams);
            
            FuncNV sel = NDKHelper::selectorList[i].getSelector();
            Node *target = NDKHelper::selectorList[i].getTarget();
            
            CallFuncNV *caller = CallFuncNV::create(sel);
            caller->setValue(value);
            
   /*
            if (target) {
                FiniteTimeAction *action = Sequence::create(caller, NULL);
                
                target->runAction(action);
            } else {
                caller->execute();
            }
   */
   if (!target)
   {
    CCAssert(false, "target is null");
   }
   caller->execute();
            
            break;
        }
    }
}

#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
#include "../../cocos2d/cocos/2d/platform/android/jni/JniHelper.h"
#include <android/log.h>
#include <jni.h>

#define LOG_TAG    "EasyNDK-for-cocos2dx"
#define CLASS_NAME "com/easyndk/classes/AndroidNDKHelper"
#endif

#if (CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
#import "IOSNDKHelper-C-Interface.h"
#endif

extern "C"
{
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    // Method for receiving NDK messages from Java, Android
    void Java_com_easyndk_classes_AndroidNDKHelper_CPPNativeCallHandler(JNIEnv *env, jobject thiz, jstring json) {
        /* The JniHelper call resulted in crash, so copy the jstring2string method here */
        //std::string jsonString = JniHelper::jstring2string(json);
        if (json == NULL) {
            return;
        }
        
        JNIEnv *pEnv = JniHelper::getEnv();
        if (!env) {
            return;
        }
        
        const char *chars = env->GetStringUTFChars(json, NULL);
        std::string ret(chars);
        env->ReleaseStringUTFChars(json, chars);
        
        std::string jsonString = ret;
        /* End jstring2string code */
        
        const char *jsonCharArray = jsonString.c_str();
        
        json_error_t error;
        json_t *root;
        root = json_loads(jsonCharArray, 0, &error);
        
        if (!root) {
            fprintf(stderr, "error: on line %d: %s\n", error.line, error.text);
            return;
        }
        
        json_t *jsonMethodName, *jsonMethodParams;
        jsonMethodName = json_object_get(root, __CALLED_METHOD__);
        jsonMethodParams = json_object_get(root, __CALLED_METHOD_PARAMS__);
        
        // Just to see on the log screen if messages are propogating properly
        // __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, jsonCharArray);
        
        NDKHelper::handleMessage(jsonMethodName, jsonMethodParams);
        json_decref(root);
    }
#endif
    

    // Method for sending message from CPP to the targeted platform
    //通过jni调用java的ReceiveCppMessage方法
    void sendMessageWithParams(std::string methodName, Value methodParams) {
        if (0 == strcmp(methodName.c_str(), "")) {
            return;
        }
        
        json_t *toBeSentJson = json_object();
        json_object_set_new(toBeSentJson, __CALLED_METHOD__, json_string(methodName.c_str()));
        
        if (!methodParams.isNull()) {
            json_t *paramsJson = NDKHelper::getJsonFromValue(methodParams);
            
            json_object_set_new(toBeSentJson, __CALLED_METHOD_PARAMS__, paramsJson);
        }
        
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
        JniMethodInfo t;
        
  if (JniHelper::getStaticMethodInfo(t,
                                           CLASS_NAME,
                                           "ReceiveCppMessage",
                                           "(Ljava/lang/String;)V")) {
            char *jsonStrLocal = json_dumps(toBeSentJson, JSON_COMPACT | JSON_ENSURE_ASCII);
            std::string jsonStr(jsonStrLocal);
            free(jsonStrLocal);
            
            jstring stringArg1 = t.env->NewStringUTF(jsonStr.c_str());
            t.env->CallStaticVoidMethod(t.classID, t.methodID, stringArg1);
            t.env->DeleteLocalRef(stringArg1);
     t.env->DeleteLocalRef(t.classID);
  }
#endif
        
#if (CC_TARGET_PLATFORM == CC_PLATFORM_IOS)
        json_t *jsonMessageName = json_string(methodName.c_str());
        
        if (!methodParams.isNull()) {
            json_t *jsonParams = NDKHelper::getJsonFromValue(methodParams);
            
            IOSNDKHelperImpl::receiveCPPMessage(jsonMessageName, jsonParams);
            
            json_decref(jsonParams);
        } else {
            IOSNDKHelperImpl::receiveCPPMessage(jsonMessageName, NULL);
        }
        
        json_decref(jsonMessageName);
#endif
        
        json_decref(toBeSentJson);
    }
}
```





