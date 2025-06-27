<template>
  <div class="home">
    <t-layout>
      <t-header class="header">
        <div class="header-content">
          <h1>简历制作器</h1>
          <div class="header-actions">
            <t-button @click="exportPDF" theme="primary">
              <template #icon>
                <t-icon name="download" />
              </template>
              导出PDF
            </t-button>
          </div>
        </div>
      </t-header>
      
      <t-layout>
        <t-aside class="editor-panel">
          <div class="editor-container">
            <div class="editor-header">
              <h3>编辑区域</h3>
              <div class="editor-actions">
                <t-button @click="insertTemplate" size="small" theme="default">
                  <template #icon>
                    <t-icon name="file-add" />
                  </template>
                  插入模板
                </t-button>
                <t-button @click="clearContent" size="small" theme="default">
                  <template #icon>
                    <t-icon name="delete" />
                  </template>
                  清空内容
                </t-button>
              </div>
            </div>
            <div id="editor" class="markdown-editor"></div>
          </div>
        </t-aside>
        
        <t-main class="preview-panel">
          <div class="preview-container">
            <div class="preview-header">
              <h3>预览效果</h3>
              <div class="preview-tools">
                <t-button size="small" @click="refreshPreview">刷新预览</t-button>
              </div>
            </div>
            <div class="resume-preview" ref="previewRef">
              <ResumeRenderer :content="resumeStore.markdownContent" />
            </div>
          </div>
        </t-main>
      </t-layout>
    </t-layout>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, onUnmounted } from 'vue'
import { useResumeStore } from '../stores/resume'
import ResumeRenderer from '../components/ResumeRenderer.vue'
import html2pdf from 'html2pdf.js'
import Cherry from 'cherry-markdown'
import 'cherry-markdown/dist/cherry-markdown.css'

const resumeStore = useResumeStore()
const previewRef = ref(null)
let cherryEditor = null
let updateTimer = null

// 防抖更新函数
const debouncedUpdate = (content) => {
  if (updateTimer) {
    clearTimeout(updateTimer)
  }
  updateTimer = setTimeout(() => {
    resumeStore.updateContent(content)
  }, 500) // 500ms延迟
}

// 组件卸载时清理定时器
onUnmounted(() => {
  if (updateTimer) {
    clearTimeout(updateTimer)
    updateTimer = null
  }
})

onMounted(async () => {
  await nextTick()
  await initEditor()
})

const initEditor = async () => {
  try {
    createEditor(Cherry)
  } catch (error) {
    console.error('Cherry Markdown初始化失败:', error)
    fallbackEditor()
  }
}

const createEditor = (Cherry) => {
  try {
    cherryEditor = new Cherry({
      id: 'editor',
      value: resumeStore.markdownContent,
      editor: {
        theme: 'default',
        height: '100%',
        defaultModel: 'edit&preview',
        fontSize: '14px',
        fontFamily: 'PingFang SC, Microsoft YaHei, sans-serif'
      },
      callback: {
        afterChange: (markdown) => {
          // 使用防抖更新而不是立即更新
          debouncedUpdate(markdown)
        },
        afterInit: () => {
          console.log('Cherry Markdown编辑器初始化完成')
        },
        afterToolbarClick: (type, value) => {
          console.log('工具栏点击:', type, value)
        }
      },
      toolbars: {
        theme: 'light',
        toolbar: [
          'bold', 'italic', 'strikethrough', 'sub', 'sup', '|',
          'header', 'list', 'ordered-list', 'quote', '|',
          'code', 'code-block', '|',
          'link', 'image', 'table', '|',
          'undo', 'redo', '|',
          'fullscreen', 'preview'
        ],
        bubble: [
          'bold', 'italic', 'strikethrough', 'sub', 'sup', 'quote', '|',
          'header', 'list', 'ordered-list', '|',
          'link', 'image', 'code', 'code-block'
        ],
        sidebar: [
          'mobilePreview', 'copy', 'theme'
        ]
      },
      keydown: {
        bindKey: {
          'Ctrl-B': 'bold',
          'Ctrl-I': 'italic',
          'Ctrl-K': 'link',
          'Ctrl-Q': 'quote',
          'Ctrl-L': 'list',
          'Ctrl-O': 'ordered-list',
          'Ctrl-H': 'header'
        }
      },
      preview: {
        enable: true,
        hljs: {
          enable: true,
          style: 'github'
        }
      }
    })
  } catch (error) {
    console.error('Cherry编辑器创建失败:', error)
    fallbackEditor()
  }
}

const fallbackEditor = () => {
  const editorElement = document.getElementById('editor')
  if (editorElement) {
    editorElement.innerHTML = `
      <textarea 
        style="width: 100%; height: 100%; border: none; outline: none; padding: 16px; font-family: 'Courier New', monospace; font-size: 14px; resize: none; background: #fff;"
        placeholder="请输入Markdown内容..."
      >${resumeStore.markdownContent}</textarea>
    `
    
    const textarea = editorElement.querySelector('textarea')
    textarea.addEventListener('input', (e) => {
      // 使用防抖更新而不是立即更新
      debouncedUpdate(e.target.value)
    })
    
    // 为fallback编辑器添加清空方法
    cherryEditor = {
      setValue: (value) => {
        textarea.value = value
        // 清空时立即更新，不使用防抖
        resumeStore.updateContent(value)
      }
    }
  }
}

const insertTemplate = () => {
  const template = `# 张三

- 📧 zhangsan@email.com
- 📱 138-0000-0000
- 📍 北京市朝阳区
- 💼 求职意向：前端开发工程师

## 教育背景

### 计算机科学与技术 @ 北京大学
2020年9月 - 2024年6月

- 主修课程：数据结构、算法设计、计算机网络、操作系统
- GPA：3.8/4.0
- 获得校级奖学金

## 工作经验

### 前端开发工程师 @ 腾讯科技
2024年7月 - 至今

- 负责公司核心产品的前端开发工作
- 使用Vue.js、React等框架开发用户界面
- 优化网站性能，提升用户体验
- 参与技术方案设计和代码审查

### 前端实习生 @ 阿里巴巴
2023年6月 - 2023年9月

- 参与电商平台的前端开发
- 学习并应用最新的前端技术栈
- 完成多个功能模块的开发

## 技能专长

- **编程语言**：JavaScript、TypeScript、HTML、CSS
- **前端框架**：Vue.js、React、Angular
- **构建工具**：Webpack、Vite、Rollup
- **版本控制**：Git、SVN
- **其他技能**：Node.js、Python、Docker

## 项目经验

### 企业管理系统
- 使用Vue.js + Element UI开发的企业级管理系统
- 实现了用户管理、权限控制、数据统计等功能
- 项目获得公司年度最佳项目奖

### 电商平台
- 基于React + Ant Design开发的电商平台
- 支持商品展示、购物车、订单管理等功能
- 日均访问量超过10万用户

## 证书荣誉

- 计算机技术与软件专业技术资格（中级）
- 英语六级证书（CET-6）
- 校级优秀毕业生
- 全国大学生程序设计竞赛三等奖`
  
  // 清除之前的防抖定时器
  if (updateTimer) {
    clearTimeout(updateTimer)
    updateTimer = null
  }
  
  if (cherryEditor) {
    cherryEditor.setValue(template)
  } else {
    resumeStore.updateContent(template)
  }
}

const clearContent = () => {
  try {
    console.log('开始清空内容...')
    console.log('当前编辑器状态:', cherryEditor ? '已初始化' : '未初始化')
    
    // 清除之前的防抖定时器
    if (updateTimer) {
      clearTimeout(updateTimer)
      updateTimer = null
    }
    
    // 直接更新store，这会触发编辑器更新
    resumeStore.updateContent('')
    console.log('Store已更新')
    
    // 如果Cherry编辑器存在，也直接清空
    if (cherryEditor && cherryEditor.setValue) {
      cherryEditor.setValue('')
      console.log('Cherry编辑器已清空')
    }
    
    // 强制触发一次更新
    nextTick(() => {
      if (cherryEditor && cherryEditor.setValue) {
        cherryEditor.setValue('')
      }
    })
    
    console.log('内容清空完成')
  } catch (error) {
    console.error('清空内容时出错:', error)
    // 确保至少更新store
    resumeStore.updateContent('')
  }
}

const refreshPreview = () => {
  // 触发重新渲染
  resumeStore.updateContent(resumeStore.markdownContent)
}

const exportPDF = () => {
  if (!previewRef.value) return
  
  const element = previewRef.value
  const opt = {
    margin: 10,
    filename: '我的简历.pdf',
    image: { type: 'jpeg', quality: 0.98 },
    html2canvas: { scale: 2 },
    jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' }
  }
  
  html2pdf().set(opt).from(element).save()
}
</script>

<style scoped>
.home {
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.header {
  background: #fff;
  border-bottom: 1px solid #e7e7e7;
  padding: 0;
  flex-shrink: 0;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 24px;
  height: 100%;
}

.header-content h1 {
  margin: 0;
  color: #0052d9;
  font-size: 20px;
}

.t-layout {
  flex: 1;
  overflow: hidden;
}

.editor-panel {
  width: 50%;
  background: #fff;
  border-right: 1px solid #e7e7e7;
  overflow: hidden;
}

.preview-panel {
  background: #f5f5f5;
  padding: 0;
  overflow: hidden;
  flex: 1;
  min-width: 0;
}

.editor-container,
.preview-container {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.editor-header,
.preview-header {
  padding: 16px 24px;
  border-bottom: 1px solid #e7e7e7;
  background: #fafafa;
  flex-shrink: 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.editor-header h3,
.preview-header h3 {
  margin: 0;
  color: #333;
  font-size: 16px;
}

.editor-actions {
  display: flex;
  gap: 8px;
}

.markdown-editor {
  flex: 1;
  overflow: hidden;
  position: relative;
}

.resume-preview {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  background: #fff;
  margin: 16px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  min-height: 400px;
}

/* Cherry Markdown编辑器样式覆盖 */
:deep(.cherry-editor) {
  height: 100% !important;
  border: none !important;
}

:deep(.cherry-editor-content) {
  height: calc(100% - 50px) !important;
  overflow-y: auto !important;
}

:deep(.cherry-editor-toolbar) {
  background: #fafafa !important;
  border-bottom: 1px solid #e7e7e7 !important;
  padding: 8px 16px !important;
}

:deep(.cherry-editor-content-editor) {
  padding: 16px !important;
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif !important;
  font-size: 14px !important;
  line-height: 1.6 !important;
}

:deep(.cherry-editor-content-preview) {
  padding: 16px !important;
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif !important;
  font-size: 14px !important;
  line-height: 1.6 !important;
}

:deep(.cherry-editor-toolbar-button) {
  border-radius: 4px !important;
  transition: all 0.2s ease !important;
}

:deep(.cherry-editor-toolbar-button:hover) {
  background-color: #e6f3ff !important;
  color: #0052d9 !important;
}

:deep(.cherry-editor-toolbar-button.active) {
  background-color: #0052d9 !important;
  color: #fff !important;
}
</style> 