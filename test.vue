// composables/useWebSocketWithReconnect.js
import { ref, onUnmounted } from 'vue'

export function useWebSocketWithReconnect(url, options = {}) {
  const {
    reconnectAttempts = 5,
    reconnectInterval = 3000
  } = options
  
  const ws = ref(null)
  const message = ref('')
  const status = ref('closed')
  const reconnectCount = ref(0)
  let reconnectTimer = null
  
  const connect = () => {
    status.value = 'connecting'
    ws.value = new WebSocket(url)
    
    ws.value.onopen = () => {
      status.value = 'open'
      reconnectCount.value = 0
      console.log('WebSocket connected')
    }
    
    ws.value.onmessage = (event) => {
      message.value = event.data
    }
    
    ws.value.onerror = (error) => {
      status.value = 'error'
      console.error('WebSocket error:', error)
    }
    
    ws.value.onclose = (event) => {
      status.value = 'closed'
      
      // 非正常关闭时尝试重连
      if (event.code !== 1000 && reconnectCount.value < reconnectAttempts) {
        reconnectTimer = setTimeout(() => {
          reconnectCount.value++
          console.log(`尝试重连 (${reconnectCount.value}/${reconnectAttempts})`)
          connect()
        }, reconnectInterval)
      }
    }
  }
  
  const send = (data) => {
    if (ws.value && ws.value.readyState === WebSocket.OPEN) {
      ws.value.send(typeof data === 'string' ? data : JSON.stringify(data))
      return true
    }
    return false
  }
  
  const disconnect = () => {
    if (reconnectTimer) {
      clearTimeout(reconnectTimer)
    }
    if (ws.value) {
      ws.value.close(1000, '正常关闭')
    }
  }
  
  onUnmounted(() => {
    disconnect()
  })
  
  return {
    message,
    status,
    reconnectCount,
    connect,
    disconnect,
    send
  }
}