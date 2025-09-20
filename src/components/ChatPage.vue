<template>
  <div class="chat-container">
    <div class="header">
      <h1>平替推荐</h1>
    </div>
    <div class="messages" ref="messagesContainer">
      <div v-for="message in messages" :key="message.id" 
           :class="['message', message.type]">
        <div class="content">{{ message.content }}</div>
        <div v-if="message.showOptions" class="options">
          <button v-for="option in message.options" :key="option" 
                  @click="selectOption(option)" class="option-btn">
            {{ option }}
          </button>
        </div>
      </div>
    </div>
    
    <div class="input-area">
      <input v-model="inputMessage" 
             @keyup.enter="sendMessage"
             placeholder="输入消息..."
             :disabled="loading" />
      <button @click="sendMessage" :disabled="loading || !inputMessage.trim()">
        {{ loading ? '发送中...' : '发送' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue'

const messages = ref([])
const inputMessage = ref('')
const loading = ref(false)
const messagesContainer = ref(null)
const conversationId = ref('')
const userIP = ref('')

const API_KEY = 'app-cMQWNu2CVCV9pv9JshO37WH3'
const API_URL = 'https://api.dify.ai/v1/chat-messages'

const getUserIP = async () => {
  try {
    const response = await fetch('https://api.ipify.org?format=json')
    const data = await response.json()
    return data.ip
  } catch {
    return 'unknown'
  }
}

const checkDailyLimit = () => {
  const today = new Date().toDateString()
  const key = `usage_${userIP.value}_${today}`
  const count = parseInt(localStorage.getItem(key) || '0')
  return count < 100
}

const incrementUsage = () => {
  const today = new Date().toDateString()
  const key = `usage_${userIP.value}_${today}`
  const count = parseInt(localStorage.getItem(key) || '0')
  localStorage.setItem(key, (count + 1).toString())
}

const getRemainingCount = () => {
  const today = new Date().toDateString()
  const key = `usage_${userIP.value}_${today}`
  const count = parseInt(localStorage.getItem(key) || '0')
  return 100 - count
}

const addMessage = (content, type, options = null) => {
  messages.value.push({
    id: Date.now(),
    content,
    type,
    showOptions: !!options,
    options
  })
  nextTick(() => {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  })
}

const selectOption = (option) => {
  // 隐藏选项按钮
  // messages.value.forEach(msg => {
  //   if (msg.showOptions) msg.showOptions = false
  // })
  // 发送选择的选项
  addMessage(option, 'user')
  sendMessageToAPI(option)
}

onMounted(async () => {
  userIP.value = await getUserIP()
  addMessage('输入商品、品牌，输出平替', 'assistant', ['沙发', '电脑', '耳机', '星巴克'])
})

const sendMessageToAPI = async (message) => {
  if (!checkDailyLimit()) {
    addMessage('今日使用次数已达上限（100次），请明天再来', 'assistant')
    return
  }
  
  loading.value = true
  
  try {
    const response = await fetch(API_URL, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        inputs: {},
        query: message,
        response_mode: 'blocking',
        conversation_id: conversationId.value,
        user: userIP.value
      })
    })
    
    const data = await response.json()
    
    if (data.answer) {
      incrementUsage()
      addMessage(data.answer, 'assistant')
      conversationId.value = data.conversation_id
    } else {
      addMessage('抱歉，出现了错误', 'assistant')
    }
  } catch (error) {
    addMessage('网络错误，请重试', 'assistant')
  } finally {
    loading.value = false
  }
}

const sendMessage = async () => {
  if (!inputMessage.value.trim() || loading.value) return
  
  const userMessage = inputMessage.value
  addMessage(userMessage, 'user')
  inputMessage.value = ''
  
  // 获取输入框引用
  const inputElement = document.querySelector('.input-area input')
  await sendMessageToAPI(userMessage)
  // 请求完成后光标复位
  nextTick(() => {
    inputElement?.focus()
  })
}
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 90vh;
  width: 100%;
  box-sizing: border-box;
  border-radius: 16px;
  background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
}

.header {
  padding: 20px;
  background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
  color: white;
  text-align: center;
  border-radius: 16px 16px 0 0;
}

.header h1 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
  letter-spacing: 0.5px;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background: transparent;
}

.message {
  margin-bottom: 16px;
  display: flex;
  flex-direction: column;
}

.message.user {
  align-items: flex-end;
}

.message.assistant {
  align-items: flex-start;
}

.content {
  max-width: min(75%, 600px);
  padding: 12px 16px;
  border-radius: 18px;
  word-wrap: break-word;
  font-size: 15px;
  line-height: 1.5;
}

.user .content {
  background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
  color: white;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.assistant .content {
  background: white;
  color: #374151;
  border: 1px solid #e5e7eb;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.input-area {
  display: flex;
  padding: 20px;
  gap: 12px;
  background: white;
  border-radius: 0 0 16px 16px;
  border-top: 1px solid #e5e7eb;
}

.input-area input {
  flex: 1;
  padding: 12px 16px;
  border: 2px solid #e5e7eb;
  border-radius: 24px;
  outline: none;
  background: #f9fafb;
  font-size: 15px;
  transition: all 0.2s;
}

.input-area input:focus {
  border-color: #3b82f6;
  background: white;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.input-area button {
  padding: 12px 24px;
  background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
  color: white;
  border: none;
  border-radius: 24px;
  cursor: pointer;
  font-weight: 500;
  font-size: 15px;
  transition: all 0.2s;
  box-shadow: 0 2px 8px rgba(59, 130, 246, 0.3);
}

.input-area button:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.4);
}

.input-area button:disabled {
  background: #9ca3af;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.options {
  margin-top: 12px;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.option-btn {
  padding: 8px 16px;
  background: white;
  border: 2px solid #e5e7eb;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
  color: #374151;
  font-weight: 500;
}

.option-btn:hover {
  border-color: #3b82f6;
  background: #3b82f6;
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(59, 130, 246, 0.2);
}

@media (max-width: 480px) {
  .content {
    max-width: 85%;
  }
  
  .header {
    padding: 16px;
  }
  
  .header h1 {
    font-size: 20px;
  }
  
  .messages {
    padding: 16px;
  }
  
  .input-area {
    padding: 16px;
  }
}
</style>