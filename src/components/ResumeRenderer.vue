<template>
  <div class="resume-renderer">
    <div class="resume-content" v-html="renderedContent"></div>
  </div>
</template>

<script setup>
import { computed, ref, watch, nextTick, onUnmounted } from 'vue'
import Cherry from 'cherry-markdown'

const props = defineProps({
  content: {
    type: String,
    default: ''
  }
})

// 本地状态，用于防抖更新
const localContent = ref(props.content)
let renderTimer = null

// 监听content变化，使用防抖更新本地状态
watch(() => props.content, (newContent) => {
  if (renderTimer) {
    clearTimeout(renderTimer)
  }
  renderTimer = setTimeout(() => {
    localContent.value = newContent
  }, 300) // 300ms延迟，比编辑器防抖时间短
}, { immediate: true })

// 组件卸载时清理定时器
onUnmounted(() => {
  if (renderTimer) {
    clearTimeout(renderTimer)
    renderTimer = null
  }
})

const renderedContent = computed(() => {
  if (!localContent.value || localContent.value.trim() === '') {
    return '<div class="empty-content"><p>请开始编辑您的简历内容...</p></div>'
  }
  return renderMarkdown(localContent.value)
})

const renderMarkdown = (markdown) => {
  if (!markdown) return ''
  
  try {
    // 使用Cherry Markdown的渲染功能
    const cherry = new Cherry({
      id: 'temp-renderer',
      value: markdown,
      editor: {
        defaultModel: 'preview'
      },
      preview: {
        enable: true,
        hljs: {
          enable: true,
          style: 'github'
        }
      }
    })
    
    // 获取渲染后的HTML
    const html = cherry.getHtml()
    
    // 清理临时元素
    const tempElement = document.getElementById('temp-renderer')
    if (tempElement) {
      tempElement.remove()
    }
    
    return html
  } catch (error) {
    console.error('Cherry Markdown渲染失败:', error)
    // 降级到简单的Markdown渲染
    return fallbackRender(markdown)
  }
}

const fallbackRender = (markdown) => {
  if (!markdown) return ''
  
  const lines = markdown.split('\n')
  let html = ''
  let inProfile = false
  let inExperience = false
  let inList = false
  
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim()
    
    if (!line) {
      if (inList) {
        html += '</ul>'
        inList = false
      }
      continue
    }
    
    // 处理一级标题（姓名）
    if (line.startsWith('# ') && i === 0) {
      const name = line.substring(2)
      html += `<div class="profile"><h1 class="name">${name}</h1>`
      inProfile = true
      continue
    }
    
    // 处理个人基本信息（列表项）
    if (inProfile && line.startsWith('- ')) {
      if (!inList) {
        html += '<ul class="profile-info">'
        inList = true
      }
      const content = line.substring(2)
      html += `<li>${content}</li>`
      continue
    }
    
    // 如果遇到二级标题，结束profile部分
    if (line.startsWith('## ')) {
      if (inList) {
        html += '</ul>'
        inList = false
      }
      if (inProfile) {
        html += '</div>'
        inProfile = false
      }
      if (inExperience) {
        html += '</div>'
        inExperience = false
      }
      
      const title = line.substring(3)
      html += `<h2 class="section-title">${title}</h2>`
      continue
    }
    
    // 处理三级标题（经历标题）
    if (line.startsWith('### ')) {
      if (inList) {
        html += '</ul>'
        inList = false
      }
      if (inExperience) {
        html += '</div>'
      }
      
      const titleText = line.substring(4)
      const timeMatch = titleText.match(/(.+) @ (.+)/)
      
      if (timeMatch) {
        const title = timeMatch[1]
        const time = timeMatch[2]
        html += `<div class="experience"><h3 class="experience-title">${title}</h3><p class="experience-time">${time}</p>`
        inExperience = true
      } else {
        html += `<div class="experience"><h3 class="experience-title">${titleText}</h3>`
        inExperience = true
      }
      continue
    }
    
    // 处理列表项（不在profile中的）
    if (line.startsWith('- ') && !inProfile) {
      if (!inList) {
        html += '<ul class="content-list">'
        inList = true
      }
      const content = line.substring(2)
      html += `<li>${content}</li>`
      continue
    }
    
    // 处理正文内容
    if (inExperience) {
      html += `<p class="experience-content">${line}</p>`
      continue
    }
    
    // 处理普通段落
    if (line && !line.startsWith('#')) {
      html += `<p class="content">${line}</p>`
    }
  }
  
  // 关闭未闭合的标签
  if (inList) {
    html += '</ul>'
  }
  if (inProfile) {
    html += '</div>'
  }
  if (inExperience) {
    html += '</div>'
  }
  
  return html
}
</script>

<style scoped>
.resume-renderer {
  font-family: 'PingFang SC', 'Microsoft YaHei', 'Helvetica Neue', Arial, sans-serif;
  line-height: 1.6;
  color: #333;
  max-width: 800px;
  margin: 0 auto;
  background: #fff;
  padding: 40px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
}

.resume-content {
  font-size: 14px;
}

/* 姓名样式 */
.resume-content h1.name {
  font-size: 32px;
  font-weight: 700;
  color: #0052d9;
  margin: 0 0 16px 0;
  text-align: center;
  border-bottom: 3px solid #0052d9;
  padding-bottom: 16px;
}

/* 个人信息样式 */
.resume-content .profile {
  margin-bottom: 32px;
}

.resume-content .profile-info {
  list-style: none;
  padding: 0;
  margin: 16px 0;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 24px;
}

.resume-content .profile-info li {
  font-size: 14px;
  color: #666;
  margin: 0;
}

/* 章节标题样式 */
.resume-content h2.section-title {
  font-size: 20px;
  font-weight: 600;
  color: #0052d9;
  margin: 32px 0 16px 0;
  padding-bottom: 8px;
  border-bottom: 2px solid #e7e7e7;
  position: relative;
}

.resume-content h2.section-title::after {
  content: '';
  position: absolute;
  bottom: -2px;
  left: 0;
  width: 60px;
  height: 2px;
  background: #0052d9;
}

/* 经历样式 */
.resume-content .experience {
  margin-bottom: 24px;
  padding: 16px;
  background: #f8f9fa;
  border-radius: 6px;
  border-left: 4px solid #0052d9;
}

.resume-content .experience-title {
  font-size: 16px;
  font-weight: 600;
  color: #0052d9;
  margin: 0 0 4px 0;
}

.resume-content .experience-time {
  font-size: 13px;
  color: #666;
  margin: 0 0 12px 0;
  font-style: italic;
}

.resume-content .experience-content {
  font-size: 14px;
  color: #333;
  margin: 8px 0;
  line-height: 1.6;
}

/* 列表样式 */
.resume-content .content-list {
  list-style: none;
  padding-left: 0;
  margin: 12px 0;
}

.resume-content .content-list li {
  font-size: 14px;
  color: #333;
  margin: 6px 0;
  line-height: 1.6;
  position: relative;
  padding-left: 20px;
}

.resume-content .content-list li::before {
  content: "•";
  color: #0052d9;
  font-weight: bold;
  position: absolute;
  left: 0;
  top: 0;
}

/* 段落样式 */
.resume-content .content {
  font-size: 14px;
  color: #333;
  margin: 12px 0;
  line-height: 1.6;
}

/* Cherry Markdown渲染内容的样式覆盖 */
.resume-content :deep(h1) {
  font-size: 32px;
  font-weight: 700;
  color: #0052d9;
  margin: 0 0 16px 0;
  text-align: center;
  border-bottom: 3px solid #0052d9;
  padding-bottom: 16px;
}

.resume-content :deep(h2) {
  font-size: 20px;
  font-weight: 600;
  color: #0052d9;
  margin: 32px 0 16px 0;
  padding-bottom: 8px;
  border-bottom: 2px solid #e7e7e7;
  position: relative;
}

.resume-content :deep(h2)::after {
  content: '';
  position: absolute;
  bottom: -2px;
  left: 0;
  width: 60px;
  height: 2px;
  background: #0052d9;
}

.resume-content :deep(h3) {
  font-size: 16px;
  font-weight: 600;
  color: #0052d9;
  margin: 24px 0 8px 0;
}

.resume-content :deep(p) {
  font-size: 14px;
  color: #333;
  margin: 12px 0;
  line-height: 1.6;
}

.resume-content :deep(ul) {
  list-style: none;
  padding-left: 0;
  margin: 12px 0;
}

.resume-content :deep(li) {
  font-size: 14px;
  color: #333;
  margin: 6px 0;
  line-height: 1.6;
  position: relative;
  padding-left: 20px;
}

.resume-content :deep(li)::before {
  content: "•";
  color: #0052d9;
  font-weight: bold;
  position: absolute;
  left: 0;
  top: 0;
}

.resume-content :deep(strong) {
  color: #0052d9;
  font-weight: 600;
}

.resume-content :deep(code) {
  background: #f1f3f4;
  padding: 2px 4px;
  border-radius: 3px;
  font-family: 'Courier New', monospace;
  font-size: 13px;
}

.resume-content :deep(blockquote) {
  border-left: 4px solid #0052d9;
  padding-left: 16px;
  margin: 16px 0;
  color: #666;
  font-style: italic;
}

.resume-content :deep(.empty-content) {
  text-align: center;
  padding: 60px 20px;
  color: #999;
  font-size: 16px;
}

.resume-content :deep(.empty-content p) {
  margin: 0;
  font-style: italic;
}
</style> 