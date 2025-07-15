<!--
 * @Author: weisheng
 * @Date: 2025-02-22 14:49:56
 * @LastEditTime: 2025-07-15 19:49:03
 * @LastEditors: weisheng
 * @Description:
 * @FilePath: /trae-ai-demo/src/pages/index.vue
 * 记得注释
-->
<script setup lang="ts">
import { nextTick, ref } from 'vue'
import { marked } from 'marked'
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'

interface Message {
  role: 'user' | 'assistant'
  content: string
  typing?: boolean
}

// 配置marked选项
marked.setOptions({
  highlight: (code: string, lang: string) => {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return hljs.highlight(code, { language: lang }).value
      }
      catch (err) {
        console.warn('代码高亮失败:', err)
      }
    }
    return hljs.highlightAuto(code).value
  },
  gfm: true,
  breaks: true,
  headerIds: false,
  mangle: false,
})

function renderMarkdown(content: string) {
  try {
    const html = marked(content)
    // 处理代码块的样式类
    return html.replace(/<pre><code/g, '<pre><code class="hljs"')
  }
  catch (err) {
    console.error('Markdown渲染失败:', err)
    return content
  }
}

const messages = ref<Message[]>([
  {
    role: 'assistant',
    content: '你好，我是你的智能助手。有什么问题我可以帮你解答吗？',
  },
])

const inputMessage = ref('')
const loading = ref(false)
const scrollTop = ref(0)
const messageListRef = ref<HTMLElement | null>(null)

// 防抖函数
function debounce(fn: Function, delay: number) {
  let timer: number | null = null
  return function (this: any, ...args: any[]) {
    if (timer)
      clearTimeout(timer)
    timer = setTimeout(() => {
      fn.apply(this, args)
    }, delay)
  }
}

// 节流函数
function throttle(fn: Function, delay: number) {
  let lastTime = 0
  let timer: number | null = null
  return function (this: any, ...args: any[]) {
    const now = Date.now()
    const remaining = delay - (now - lastTime)
    if (remaining <= 0) {
      if (timer) {
        clearTimeout(timer)
        timer = null
      }
      fn.apply(this, args)
      lastTime = now
    }
    else if (!timer) {
      timer = setTimeout(() => {
        fn.apply(this, args)
        lastTime = Date.now()
        timer = null
      }, remaining)
    }
  }
}

// 滚动到底部的函数
const scrollToBottom = throttle(async () => {
  if (!messageListRef.value)
    return

  const lastMessage = messages.value[messages.value.length - 1]
  if (lastMessage?.typing)
    return // 打字过程中不触发滚动

  await nextTick()
  const query = uni.createSelectorQuery()
  query.select('.message-list').boundingClientRect((data) => {
    if (data) {
      scrollTop.value = data.height
    }
  }).exec()
}, 200) // 增加节流时间，减少滚动更新频率

// 监听消息列表变化，自动滚动到底部
watch(() => messages.value.length, () => {
  if (messages.value[messages.value.length - 1]?.typing)
    return
  scrollToBottom()
})

// 监听最后一条消息的内容变化（用于打字效果）
watch(
  () => messages.value[messages.value.length - 1]?.content,
  async () => {
    const lastMessage = messages.value[messages.value.length - 1]
    if (lastMessage?.typing) {
      // 在打字过程中，每次内容变化都尝试滚动
      await nextTick()
      const query = uni.createSelectorQuery()
      query.select('.message-list').boundingClientRect((data) => {
        if (data && data.height > scrollTop.value) {
          scrollTop.value = data.height
        }
      }).exec()
    }
  },
)

// 模拟打字效果的函数
async function typeMessage(fullContent: string) {
  try {
    const message: Message = {
      role: 'assistant',
      content: '',
      typing: true,
    }
    messages.value.push(message)

    // 确保滚动到底部
    await nextTick()
    await scrollToBottom()

    // 使用Array.from正确处理Unicode字符
    const chars = Array.from(fullContent)
    let currentContent = ''
    const batchSize = 1 // 每次只更新一个字符，使效果更自然

    for (let i = 0; i < chars.length; i += batchSize) {
      currentContent += chars.slice(i, i + batchSize).join('')
      messages.value[messages.value.length - 1].content = currentContent
      await new Promise(resolve => setTimeout(resolve, 10)) // 减少延迟时间，使打字更流畅
      await scrollToBottom() // 每次更新内容后都滚动到底部
    }

    // 确保最后的内容完整显示
    messages.value[messages.value.length - 1].content = fullContent
    messages.value[messages.value.length - 1].typing = false

    // 使用nextTick确保DOM更新后再次滚动到底部
    await nextTick()
    await scrollToBottom() // 最后再次确保滚动到底部
  }
  catch (error) {
    console.error('打字效果执行失败:', error)
    // 发生错误时直接显示完整内容
    const message: Message = {
      role: 'assistant',
      content: fullContent,
      typing: false,
    }
    messages.value.push(message)
    // 错误处理时也确保滚动到底部
    await nextTick()
    await scrollToBottom()
  }
}

async function sendMessage() {
  if (!inputMessage.value.trim())
    return

  // 添加用户消息
  messages.value.push({
    role: 'user',
    content: inputMessage.value,
  })

  loading.value = true
  try {
    // TODO: 调用AI API获取回复
    const aiResponse = `让我为你展示一些Markdown的基本用法：

### 文本格式
**粗体文本** 和 *斜体文本*
~~删除线文本~~ 和 \`行内代码\`

### 列表示例
- 无序列表项1
- 无序列表项2
  - 子列表项
  - 子列表项

1. 有序列表项1
2. 有序列表项2

### 引用示例
> 这是一段引用文本
> 支持多行引用
>> 还可以嵌套引用

### 代码示例
\`\`\`javascript
function greeting(name) {
    return \`Hello, ${name}!\`;
}

console.log(greeting("World"));
\`\`\`

### 表格示例
| 表头1 | 表头2 | 表头3 |
|-------|-------|-------|
| 内容1 | 内容2 | 内容3 |
| 示例1 | 示例2 | 示例3 |

### 任务列表
- [x] 已完成任务
- [ ] 未完成任务
- [x] Markdown支持
- [x] 代码高亮

### Python代码示例
\`\`\`python
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
\`\`\`

### 链接和图片
[Markdown指南](https://www.markdownguide.org)

### 功能支持
- [x] 支持Markdown
- [x] 支持代码高亮
- [x] 支持表格`

    // 清空输入框
    inputMessage.value = ''

    // 开始模拟打字效果
    await typeMessage(aiResponse)
  }
  catch (error) {
    console.error('发送消息失败:', error)
  }
  finally {
    loading.value = false
  }
}

function loadMoreMessages() {
  // TODO: 实现加载更多历史消息
  console.log('加载更多消息')
}
</script>

<template>
  <view class="chat-container">
    <scroll-view
      ref="messageListRef"
      scroll-y
      class="chat-messages"
      :scroll-top="scrollTop"
      :scroll-with-animation="false"
      :scroll-anchoring="true"
      :enhanced="true"
      :bounces="false"
      @scrolltoupper="loadMoreMessages"
    >
      <view class="message-list">
        <view
          v-for="(message, index) in messages"
          :key="index"
          class="message-item"
          :class="[message.role, { typing: message.typing }]"
        >
          <view class="message-avatar">
            <image
              :src="message.role === 'user' ? 'https://avatars.githubusercontent.com/u/26426873?v=4' : 'https://avatars.githubusercontent.com/u/148330874?s=48&v=4'"
              mode="aspectFill"
            />
          </view>
          <view class="message-content">
            <rich-text :nodes="renderMarkdown(message.content)" />
            <view v-if="message.typing" class="typing-indicator">
              <view class="dot" />
              <view class="dot" />
              <view class="dot" />
            </view>
          </view>
        </view>
      </view>
    </scroll-view>

    <view class="chat-input safe-area-bottom">
      <wd-input
        v-model="inputMessage"
        placeholder="请输入消息"
        class="message-textarea"
        :disabled="loading"
        @keypress.enter.prevent="sendMessage"
      />
      <wd-button type="primary" size="small" :loading="loading" :disabled="!inputMessage.trim()" @click="sendMessage">
        发送
      </wd-button>
    </view>
  </view>
</template>

<style lang="scss" scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100dvh;
  background-color: #f7f7f7;
  position: relative;
}

.chat-messages {
  flex: 1;
  box-sizing: border-box;
  padding: 20rpx;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}

.message-list {
  padding-bottom: 20rpx;
}

.message-item {
  display: flex;
  margin-bottom: 30rpx;
  opacity: 1;
  transform: translateY(0);
  transition: opacity 0.3s, transform 0.3s;
  padding: 0 20rpx;

  &.typing {
    .message-content {
      min-width: 120rpx;
    }
  }

  &.user {
    flex-direction: row-reverse;

    .message-content {
      background-color: #007aff;
      color: #fff;
      margin-left: 0;
      margin-right: 20rpx;
      border-radius: 20rpx 4rpx 20rpx 20rpx;

      :deep(pre), :deep(code) {
        background-color: rgba(255, 255, 255, 0.1);
        color: #fff;
      }
    }
  }

  &.assistant .message-content {
    background-color: #fff;
    border-radius: 4rpx 20rpx 20rpx 20rpx;
  }
}

.message-avatar {
  width: 80rpx;
  height: 80rpx;
  flex-shrink: 0;
  margin: 0;

  image {
    width: 100%;
    height: 100%;
    border-radius: 50%;
  }
}

.message-content {
  max-width: 60%;
  padding: 20rpx;
  margin: 0 20rpx;
  font-size: 28rpx;
  word-break: break-all;
  box-shadow: 0 2rpx 12rpx rgba(0, 0, 0, 0.1);

  :deep(pre) {
    background-color: #f6f8fa;
    padding: 16rpx;
    border-radius: 6rpx;
    overflow-x: auto;
    margin: 16rpx 0;
  }

  :deep(code) {
    font-family: Consolas, Monaco, 'Andale Mono', monospace;
    font-size: 24rpx;
    padding: 4rpx 8rpx;
    background-color: #f6f8fa;
    border-radius: 4rpx;
  }

  :deep(p) {
    margin: 16rpx 0;
  }

  :deep(ul), :deep(ol) {
    padding-left: 32rpx;
    margin: 16rpx 0;
  }

  :deep(table) {
    border-collapse: collapse;
    margin: 16rpx 0;
    width: 100%;
  }

  :deep(th), :deep(td) {
    border: 2rpx solid #dfe2e5;
    padding: 12rpx 16rpx;
  }

  :deep(th) {
    background-color: #f6f8fa;
  }

  :deep(a) {
    color: #0366d6;
    text-decoration: none;

    &:hover {
      text-decoration: underline;
    }
  }

  :deep(img) {
    max-width: 100%;
    height: auto;
  }

  :deep(blockquote) {
    margin: 16rpx 0;
    padding: 0 16rpx;
    color: #6a737d;
    border-left: 4rpx solid #dfe2e5;
  }
}

.chat-input {
  padding: 20rpx;
  background-color: #fff;
  border-top: 2rpx solid #eee;
  display: flex;
  align-items: flex-end;
  gap: 20rpx;
  position: sticky;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 100;

  .message-textarea {
    flex: 1;
  }
}

.typing-indicator {
  display: flex;
  align-items: center;
  gap: 8rpx;
  padding: 8rpx 0;

  .dot {
    width: 8rpx;
    height: 8rpx;
    background-color: #999;
    border-radius: 50%;
    animation: typing 1.4s infinite;

    &:nth-child(2) {
      animation-delay: 0.2s;
    }

    &:nth-child(3) {
      animation-delay: 0.4s;
    }
  }
}

@keyframes typing {
  0%, 60%, 100% {
    transform: translateY(0);
    opacity: 0.3;
  }
  30% {
    transform: translateY(-4rpx);
    opacity: 1;
  }
}
</style>
