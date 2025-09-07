// axios 配置文件中
import axios from 'axios'

// 验证图片 URL 的函数
const validateImgUrl = (url) => {
  if (!url) return null
  // 这里添加你的验证逻辑
  const pattern = /^https?:\/\/.+\/.+\.(jpg|jpeg|png|gif|webp)$/i
  return pattern.test(url) ? url : null
}

// 响应拦截器
axios.interceptors.response.use(
  (response) => {
    if (response.data && typeof response.data === 'object') {
      // 递归处理对象中的所有值
      const processValue = (value) => {
        if (typeof value === 'string') {
          // 如果是字符串，尝试验证是否为图片 URL
          return validateImgUrl(value)
        } else if (Array.isArray(value)) {
          // 如果是数组，递归处理每个元素
          return value.map(processValue)
        } else if (typeof value === 'object' && value !== null) {
          // 如果是对象，递归处理每个属性
          const processedObj = {}
          Object.keys(value).forEach(key => {
            processedObj[key] = processValue(value[key])
          })
          return processedObj
        }
        return value
      }
      
      response.data = processValue(response.data)
    }
    return response
  },
  (error) => {
    return Promise.reject(error)
  }
)

export default axios