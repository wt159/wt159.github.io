---
title: Android系统之AMessage
tags: Android AMessage AHandler ALooper
---

# Android系统之AMessage

## 概述

Android中的ALooper、AHandler和AMessage是Native层实现的一套异步消息机制，主要用于多媒体框架（如MediaCodec、NuPlayer等）的线程间通信。其设计理念与Java层的Handler/Looper/Message机制类似，但面向C++实现，具有更高性能和控制力。

### 核心组件解析
1. **ALooper**  
   • **功能**：管理消息队列（MessageQueue）和线程循环，负责创建线程并持续从队列中取出消息分发给对应的AHandler。  
   • **关键方法**：  
     ◦ `start()`：启动线程并进入消息循环（通过`LooperThread`实现）  
     ◦ `registerHandler()`：将AHandler注册到ALooper，建立绑定关系  
   • **特点**：支持两种运行模式——在调用线程运行（`runOnCallingThread`）或独立线程运行。

2. **AHandler**  
   • **功能**：消息处理器基类，需继承并实现`onMessageReceived()`以定义具体业务逻辑。  
   • **绑定机制**：通过`setID()`与ALooper关联，每个AHandler必须绑定到一个ALooper实例。  
   • **消息处理**：ALooper从队列取出消息后，根据消息的`handler_id`调用对应AHandler的`onMessageReceived()`。

3. **AMessage**  
   • **功能**：消息载体，支持多种数据类型（如int32、string、Buffer等），并可指定目标AHandler。  
   • **关键方法**：  
     ◦ `post()`：异步发送普通消息  
     ◦ `postAndAwaitResponse()`：发送需回复的消息，等待响应  
   • **构造方式**：推荐通过`AMessage::obtain()`复用对象，避免频繁内存分配。

### 协作流程
1. **初始化**  
   • 创建ALooper实例并启动线程循环：  
     ```c++
     sp<ALooper> mLooper = new ALooper();
     mLooper->start(false, true, ANDROID_PRIORITY_VIDEO);
     ```  
   • 注册AHandler到ALooperRoster（全局管理器），建立ID映射。

2. **消息发送**  
   • 构造AMessage并指定目标AHandler：  
     ```c++
     sp<AMessage> msg = new AMessage(kWhatEvent, mHandler);
     msg->setInt32("value", 100);
     msg->post();
     ```  
   • 消息被插入ALooper的MessageQueue。

3. **消息处理**  
   • ALooper线程循环调用`loop()`，从队列取出AMessage，根据ID查找对应AHandler并触发`onMessageReceived()`。

### 与Java层Handler机制的区别
| 特性                | Native层（ALooper/AHandler）       | Java层（Handler/Looper）       |
|---------------------|-----------------------------------|--------------------------------|
| **线程模型**         | 需要显式注册Handler到Looper       | 主线程自动初始化Looper          |
| **性能**            | 更轻量，无GC开销                  | 依赖Java虚拟机                |
| **应用场景**         | 多媒体框架、高性能场景            | UI线程通信、常规异步任务       |
| **消息类型**         | 支持需回复的消息（同步等待）       | 仅异步消息                    |

### 实际应用场景
• **NuPlayer播放器**：通过继承AHandler实现媒体状态机，处理解码、渲染等异步事件。  
• **MediaCodec编解码**：使用AMessage传递缓冲区数据和控制命令（如FLUSH、STOP）。  

### 设计优势
1. **解耦线程与任务**：将耗时操作封装为AMessage，由ALooper线程异步执行，避免阻塞主线程。  
2. **高效内存管理**：通过引用计数（sp<>智能指针）和对象池（AMessage复用）减少资源消耗。  
3. **跨线程安全**：通过互斥锁（Mutex）保护消息队列操作，确保多线程环境下的数据一致性。  

---

### AMessage的重要接口解析

AMessage是Android Native层异步通信的核心载体，其接口设计兼顾了高性能与扩展性，以下从**数据存取**、**消息投递**、**生命周期管理**三个维度解析关键接口：

---

#### 一、数据存取接口（类型安全操作）
1. **基础类型存取**  
   ```c++
   // 设置数据
   void setInt32(const char *name, int32_t value);
   void setString(const char *name, const AString &value);
   void setBuffer(const char *name, const sp<ABuffer> &buffer);

   // 获取数据（返回bool表示是否存在）
   bool findInt32(const char *name, int32_t *value) const;
   bool findString(const char *name, AString *value) const;
   bool findBuffer(const char *name, sp<ABuffer> *buffer) const;
   ```
   • **特点**：通过键值对（`const char*`）实现强类型数据存取，支持`int32_t`、`AString`、`ABuffer`等常用类型。

2. **复杂对象传递**  
   ```c++
   void setObject(const char *name, const sp<RefBase> &obj);
   bool findObject(const char *name, sp<RefBase> *obj) const;
   ```
   • **应用场景**：传递继承自`RefBase`的智能指针对象（如跨线程共享的媒体缓冲区）。

3. **嵌套消息支持**  
   ```c++
   bool findMessage(const char *name, sp<AMessage> *msg) const;
   ```
   • **用途**：构建消息链式处理逻辑（如将子任务结果封装为嵌套消息）。

---

#### 二、消息投递与控制接口
1. **异步投递**  
   ```c++
   status_t post(int64_t delayUs = 0);  // 延迟投递（单位微秒）
   ```
   • **示例**：延迟100ms发送消息  
     ```c++
     msg->post(100000);  // 用于定时任务或流量控制
     ```

2. **同步通信机制**  
   ```c++
   status_t postAndAwaitResponse(sp<AMessage> *response);  // 发送并等待回复
   bool senderAwaitsResponse(sp<AReplyToken> *replyID);    // 判断是否需要回复
   status_t postReply(const sp<AReplyToken> &replyID);     // 返回响应消息
   ```
   • **协作流程**：  
     1. 发送端调用`postAndAwaitResponse`阻塞等待  
     2. 接收端通过`senderAwaitsResponse`检测是否需要回复  
     3. 接收端构造响应消息并调用`postReply`

---

#### 三、生命周期管理接口
1. **对象复用机制**  
   ```c++
   static sp<AMessage> obtain();  // 从对象池获取实例
   void recycle();                // 重置消息状态并回收至对象池
   ```
   • **优势**：减少内存分配开销（类似Java层`Message.obtain()`的设计）。

2. **深拷贝支持**  
   ```c++
   sp<AMessage> dup() const;  // 创建完全独立的副本
   ```
   • **使用场景**：多线程环境下需要发送相同消息至不同Handler时。

---

#### 四、辅助功能接口
1. **消息内容探查**  
   ```c++
   size_t countEntries() const;                          // 获取键值对总数
   const char* getEntryNameAt(size_t index, Type *type); // 遍历消息字段
   ```
   • **调试用途**：动态分析消息结构，适用于复杂状态机的日志输出。

2. **目标标识操作**  
   ```c++
   void setWhat(uint32_t what);          // 设置消息类型标识
   void setTarget(handler_id handlerID); // 指定接收Handler的ID
   ```
   • **设计思想**：解耦消息内容与处理逻辑（类似Android Java层的`Message.what`机制）。

---

## 示例代码

### Java层Handler示例代码

#### 场景：子线程发送消息到主线程更新UI
```java
// 主线程中初始化Handler（默认绑定主线程Looper）
Handler mainHandler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(Message msg) {
        if (msg.what == 1) {
            String result = (String) msg.obj;
            textView.setText(result); // 更新UI
        }
    }
};

// 子线程中发送消息
new Thread(() -> {
    // 模拟耗时操作
    SystemClock.sleep(3000);
    
    Message msg = Message.obtain();
    msg.what = 1;
    msg.obj = "任务完成";
    mainHandler.sendMessage(msg); // 发送消息到主线程
}).start();
```

#### 关键点说明：
1. **主线程Looper**：通过`Looper.getMainLooper()`获取主线程的Looper
2. **消息封装**：使用`Message.obtain()`复用消息对象，设置`what`标识和`obj`数据载体
3. **跨线程通信**：子线程通过持有主线程的Handler引用实现安全UI更新

---

### Native层（C++）Handler示例代码

#### 场景：自定义线程通过ALooper处理异步消息
```c++
// 继承AHandler实现消息处理器
class MyHandler : public AHandler {
public:
    enum MessageType { kWhatEvent = 1 };

protected:
    void onMessageReceived(const sp<AMessage> &msg) override {
        int32_t what;
        msg->findInt32("what", &what);
        if (what == kWhatEvent) {
            int32_t value;
            msg->findInt32("value", &value);
            ALOGD("收到Native消息，value=%d", value);
        }
    }
};

// 初始化ALooper和Handler
sp<ALooper> mLooper = new ALooper();
sp<MyHandler> mHandler = new MyHandler();

// 启动Looper线程并注册Handler
mLooper->setName("NativeLooper");
mLooper->start(false, true, ANDROID_PRIORITY_DEFAULT); 
mLooper->registerHandler(mHandler);

// 发送消息
sp<AMessage> msg = new AMessage(MyHandler::kWhatEvent, mHandler);
msg->setInt32("value", 100);
msg->post(); // 异步投递消息
```

#### 关键点说明：
1. **消息类型定义**：使用枚举值`kWhatEvent`作为消息标识
2. **对象生命周期管理**：通过`sp<>`智能指针自动管理对象引用
3. **消息构造技巧**：推荐使用`AMessage::setXXX`设置数据类型，支持int32/string/buffer等
4. **线程绑定**：通过`registerHandler()`建立Handler与Looper的绑定关系

---

### 对比说明

| 特性                | Java层实现                           | Native层实现             |
|---------------------|------------------------------------|-------------------------------|
| **核心类**          | Handler/Looper/Message            | AHandler/ALooper/AMessage      |
| **线程模型**        | 主线程自动初始化Looper              | 需显式调用start()启动线程循环   |
| **内存管理**        | 依赖JVM垃圾回收                     | 引用计数(sp<>)自动回收          |
| **性能特点**        | 适合常规异步任务                    | 无GC开销，适合多媒体高吞吐场景  |
| **跨进程支持**      | 不支持                             | 可通过Binder传递AMessage        |
| **同步通信**        | 仅支持异步                         | 支持postAndAwaitResponse同步等待 |

---

### 进阶技巧
1. **Native层同步通信**：

```c++
// 发送端
sp<AMessage> response;
msg->postAndAwaitResponse(&response); // 阻塞等待响应

// 接收端
sp<AReplyToken> replyID;
if (msg->senderAwaitsResponse(&replyID)) {
    sp<AMessage> response = new AMessage;
    response->postReply(replyID); // 返回响应
}
```

2. **Java与Native交互**：

```java
// 通过JNI获取NativeHandler指针
public native void sendNativeMessage(int value);

// Native实现
extern "C" JNIEXPORT void JNICALL
Java_com_example_NativeHandler_sendNativeMessage(JNIEnv* env, jobject, jint value) {
    sp<AMessage> msg = new AMessage(kWhatFromJava, mHandler);
    msg->setInt32("value", value);
    msg->post();
}
```

---
