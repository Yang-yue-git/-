<template>
  <div class="msg-page">
    <!-- 顶部标题栏 -->
    <el-card shadow="never" class="header-card">
      <div class="header-inner">
        <div>
          <h1 class="page-title"><el-icon class="title-icon"><ChatDotRound /></el-icon>客户留言系统</h1>
          <div class="sub-title">留言展示 · 在线留言 · 分类筛选 · 本地持久化</div>
        </div>
        <div class="header-actions">
          <el-button @click="exportMessages" :disabled="!messages.length"><el-icon><Download /></el-icon>导出留言</el-button>
          <el-button type="danger" plain :disabled="!messages.length" @click="clearAllMessages"><el-icon><Delete /></el-icon>清空全部</el-button>
        </div>
      </div>
    </el-card>

    <!-- 统计条 -->
    <div class="stat-bar">
      <span>共 <b class="num">{{ messages.length }}</b> 条留言</span>
      <span class="divider"></span>
      <span>已回复 <b class="num ok">{{ repliedCount }}</b> 条</span>
      <span class="divider"></span>
      <span>待回复 <b class="num warn">{{ pendingCount }}</b> 条</span>
      <span style="margin-left:auto;color:#9aa7ba">留言数据保存在本地，刷新不丢失</span>
    </div>

    <div class="msg-layout">
      <!-- 左侧：留言表单 -->
      <el-card shadow="never" class="form-card">
        <template #header>
          <span class="card-title"><el-icon><EditPen /></el-icon> 我要留言</span>
        </template>
        <el-form :model="form" label-position="top" class="form-body">
          <el-form-item label="客户姓名" required>
            <el-input v-model.trim="form.name" placeholder="请输入您的姓名" maxlength="20" clearable />
          </el-form-item>
          <el-form-item label="联系电话">
            <el-input v-model.trim="form.phone" placeholder="请输入手机号（选填）" maxlength="11" clearable />
          </el-form-item>
          <el-form-item label="留言分类" required>
            <el-select v-model="form.category" style="width:100%">
              <el-option v-for="c in categories" :key="c" :label="c" :value="c" />
            </el-select>
          </el-form-item>
          <el-form-item label="留言内容" required>
            <el-input
              v-model.trim="form.content"
              type="textarea"
              :rows="5"
              placeholder="请输入留言内容（最多500字）"
              maxlength="500"
              show-word-limit
            />
          </el-form-item>
          <el-button type="primary" class="submit-btn" :disabled="!canSubmit" @click="submitMessage">
            <el-icon><Promotion /></el-icon>提交留言
          </el-button>
        </el-form>
      </el-card>

      <!-- 右侧：留言列表 -->
      <div class="list-section">
        <!-- 筛选栏 -->
        <el-card shadow="never" class="filter-card">
          <div class="filter-bar">
            <el-input v-model.trim="filterKeyword" placeholder="搜索留言内容/姓名..." clearable style="flex:1;min-width:160px">
              <template #prefix><el-icon><Search /></el-icon></template>
            </el-input>
            <el-select v-model="filterCategory" placeholder="全部分类" clearable style="width:130px">
              <el-option v-for="c in categories" :key="c" :label="c" :value="c" />
            </el-select>
            <el-select v-model="filterStatus" placeholder="全部状态" clearable style="width:120px">
              <el-option label="待回复" value="pending" />
              <el-option label="已回复" value="replied" />
            </el-select>
            <el-select v-model="sortBy" style="width:120px">
              <el-option label="最新优先" value="newest" />
              <el-option label="最早优先" value="oldest" />
            </el-select>
          </div>
        </el-card>

        <!-- 留言卡片列表 -->
        <div class="msg-list" v-if="filteredList.length">
          <el-card
            v-for="msg in filteredList"
            :key="msg.id"
            shadow="hover"
            class="msg-item"
          >
            <div class="msg-head">
              <div class="msg-user">
                <el-avatar :size="38" class="avatar">{{ msg.name.charAt(0) }}</el-avatar>
                <div>
                  <div class="user-name">{{ msg.name }}</div>
                  <div class="user-meta">
                    <el-tag :type="catTagType(msg.category)" size="small" effect="light">{{ msg.category }}</el-tag>
                    <span v-if="msg.phone" class="phone"><el-icon><Phone /></el-icon>{{ msg.phone }}</span>
                    <span class="time">{{ formatTime(msg.createdAt) }}</span>
                  </div>
                </div>
              </div>
              <div class="msg-ops">
                <el-tag :type="msg.replied ? 'success' : 'warning'" size="small" effect="plain">
                  {{ msg.replied ? '已回复' : '待回复' }}
                </el-tag>
                <el-button
                  :type="replyingId === msg.id ? 'default' : 'primary'"
                  size="small"
                  circle
                  :icon="ChatLineRound"
                  :title="msg.replied ? '修改回复' : '回复'"
                  @click="toggleReply(msg)"
                />
                <el-button type="danger" size="small" circle :icon="Delete" title="删除" @click="deleteMessage(msg.id)" />
              </div>
            </div>
            <div class="msg-content">{{ msg.content }}</div>

            <!-- 回复输入区 -->
            <div v-if="replyingId === msg.id" class="reply-area">
              <el-input
                v-model.trim="replyText"
                type="textarea"
                :rows="3"
                placeholder="输入回复内容..."
                maxlength="500"
              />
              <div class="reply-ops">
                <el-button size="small" @click="cancelReply">取消</el-button>
                <el-button size="small" type="primary" :disabled="!replyText.trim()" @click="saveReply(msg)">保存回复</el-button>
              </div>
            </div>

            <!-- 已有回复 -->
            <div v-if="msg.replied && replyingId !== msg.id" class="reply-box">
              <div class="reply-label"><el-icon><ChatDotRound /></el-icon> 官方回复</div>
              <div class="reply-text">{{ msg.reply }}</div>
              <div class="reply-time">回复时间：{{ formatTime(msg.repliedAt) }}</div>
            </div>
          </el-card>
        </div>

        <!-- 空状态 -->
        <el-empty
          v-else
          :description="messages.length ? '没有符合筛选条件的留言' : '暂无留言，快来发表第一条留言吧'"
          :image-size="120"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  ChatDotRound, EditPen, Promotion, Delete, Search, Phone, ChatLineRound, Download
} from '@element-plus/icons-vue'

const STORAGE_KEY = 'vue3_customer_messages'
const categories = ['咨询', '建议', '投诉', '合作', '其他']

// ===== 表单 =====
const form = reactive({
  name: '',
  phone: '',
  category: '咨询',
  content: ''
})

// ===== 留言数据 =====
const messages = ref([])
const filterKeyword = ref('')
const filterCategory = ref('')
const filterStatus = ref('')
const sortBy = ref('newest')
const replyingId = ref(null)
const replyText = ref('')

// ===== 加载/保存数据 =====
function loadMessages() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw) {
      const data = JSON.parse(raw)
      if (Array.isArray(data)) messages.value = data
    }
  } catch (e) {
    console.warn('[留言] 读取失败:', e.message)
  }
}

function saveMessages() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(messages.value))
  } catch (e) {
    ElMessage.error('保存失败：' + e.message)
  }
}

// ===== 计算属性 =====
const canSubmit = computed(() => form.name.trim() && form.content.trim())
const repliedCount = computed(() => messages.value.filter(m => m.replied).length)
const pendingCount = computed(() => messages.value.filter(m => !m.replied).length)

const filteredList = computed(() => {
  let list = [...messages.value]
  if (filterKeyword.value) {
    const kw = filterKeyword.value.toLowerCase()
    list = list.filter(m =>
      m.name.toLowerCase().includes(kw) ||
      m.content.toLowerCase().includes(kw)
    )
  }
  if (filterCategory.value) list = list.filter(m => m.category === filterCategory.value)
  if (filterStatus.value === 'replied') list = list.filter(m => m.replied)
  else if (filterStatus.value === 'pending') list = list.filter(m => !m.replied)
  list.sort((a, b) => sortBy.value === 'newest' ? b.createdAt - a.createdAt : a.createdAt - b.createdAt)
  return list
})

// ===== 方法 =====
function catTagType(cat) {
  const map = { '咨询': 'success', '建议': 'primary', '投诉': 'danger', '合作': 'warning', '其他': 'info' }
  return map[cat] || 'info'
}

function submitMessage() {
  if (!form.name.trim()) { ElMessage.warning('请输入客户姓名'); return }
  if (!form.content.trim()) { ElMessage.warning('请输入留言内容'); return }
  const msg = {
    id: Date.now() + Math.random().toString(36).slice(2, 8),
    name: form.name.trim(),
    phone: form.phone.trim(),
    category: form.category,
    content: form.content.trim(),
    createdAt: Date.now(),
    replied: false,
    reply: '',
    repliedAt: null
  }
  messages.value.unshift(msg)
  saveMessages()
  form.name = ''
  form.phone = ''
  form.category = '咨询'
  form.content = ''
  ElMessage.success('留言提交成功！')
}

function deleteMessage(id) {
  ElMessageBox.confirm('确定要删除这条留言吗？', '删除确认', {
    confirmButtonText: '删除',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(() => {
    messages.value = messages.value.filter(m => m.id !== id)
    saveMessages()
    ElMessage.success('已删除留言')
  }).catch(() => {})
}

function toggleReply(msg) {
  if (replyingId.value === msg.id) {
    cancelReply()
  } else {
    replyingId.value = msg.id
    replyText.value = msg.reply || ''
  }
}

function cancelReply() {
  replyingId.value = null
  replyText.value = ''
}

function saveReply(msg) {
  if (!replyText.value.trim()) return
  msg.replied = true
  msg.reply = replyText.value.trim()
  msg.repliedAt = Date.now()
  saveMessages()
  cancelReply()
  ElMessage.success('回复已保存')
}

function clearAllMessages() {
  if (!messages.value.length) return
  ElMessageBox.confirm('确定要清空全部 ' + messages.value.length + ' 条留言吗？此操作不可撤销。', '清空确认', {
    confirmButtonText: '清空全部',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(() => {
    messages.value = []
    saveMessages()
    ElMessage.success('已清空全部留言')
  }).catch(() => {})
}

function exportMessages() {
  if (!messages.value.length) { ElMessage.warning('没有可导出的留言'); return }
  const header = '序号,客户姓名,联系电话,分类,留言内容,留言时间,是否回复,回复内容,回复时间'
  const rows = messages.value.map((m, i) => [
    i + 1, m.name, m.phone || '', m.category, m.content,
    formatTime(m.createdAt), m.replied ? '已回复' : '待回复',
    m.reply || '', m.repliedAt ? formatTime(m.repliedAt) : ''
  ].map(v => {
    let s = String(v == null ? '' : v)
    if (/[",\n]/.test(s)) s = '"' + s.replace(/"/g, '""') + '"'
    return s
  }).join(','))
  const csv = '\uFEFF' + header + '\r\n' + rows.join('\r\n')
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = '客户留言导出_' + new Date().toISOString().slice(0, 10) + '.csv'
  document.body.appendChild(a); a.click()
  document.body.removeChild(a); URL.revokeObjectURL(url)
  ElMessage.success('已导出 ' + messages.value.length + ' 条留言')
}

function formatTime(ts) {
  if (!ts) return ''
  const d = new Date(ts)
  const pad = n => String(n).padStart(2, '0')
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}`
}

onMounted(() => {
  loadMessages()
})
</script>

<style scoped>
.msg-page { padding: 20px 24px 40px; }

.header-card { border: 1px solid var(--border); border-radius: 12px; margin-bottom: 16px; }
.header-card :deep(.el-card__body) { padding: 14px 22px; }
.header-inner { display: flex; align-items: center; justify-content: space-between; gap: 16px; flex-wrap: wrap; }
.page-title { font-size: 20px; font-weight: 600; display: flex; align-items: center; gap: 10px; }
.title-icon { color: var(--primary); font-size: 22px; }
.sub-title { color: var(--text-2); font-size: 12px; margin-top: 4px; }
.header-actions { display: flex; gap: 10px; flex-wrap: wrap; }

.stat-bar { display: flex; align-items: center; gap: 10px; padding: 0 2px 14px; color: var(--text-2); font-size: 13px; flex-wrap: wrap; }
.stat-bar .num { color: var(--primary); font-weight: 700; font-size: 15px; }
.stat-bar .num.ok { color: var(--ok); }
.stat-bar .num.warn { color: #d97706; }
.divider { width: 1px; height: 14px; background: var(--border); }

.msg-layout { display: grid; grid-template-columns: 380px 1fr; gap: 16px; align-items: start; }

/* 表单卡片 */
.form-card { border: 1px solid var(--border); border-radius: 12px; }
.form-card :deep(.el-card__header) { padding: 14px 20px; background: #f8fafd; border-bottom: 1px solid var(--border); }
.form-title { font-size: 15px; font-weight: 600; display: flex; align-items: center; gap: 8px; }
.form-body { padding-top: 4px; }

.submit-btn { width: 100%; padding: 12px 0; font-size: 14px; }

/* 列表区 */
.list-section { min-width: 0; }
.filter-card { border: 1px solid var(--border); border-radius: 12px; margin-bottom: 14px; }
.filter-card :deep(.el-card__body) { padding: 14px 16px; }
.filter-bar { display: flex; gap: 10px; flex-wrap: wrap; }

.msg-list { display: flex; flex-direction: column; gap: 12px; }
.msg-item { border: 1px solid var(--border); border-radius: 12px; }
.msg-item :deep(.el-card__body) { padding: 16px 20px; }

.msg-head { display: flex; align-items: flex-start; justify-content: space-between; gap: 12px; margin-bottom: 10px; }
.msg-user { display: flex; gap: 10px; align-items: flex-start; }
.avatar { background: linear-gradient(135deg,#2f6fed,#5aa0ff); color: #fff; font-weight: 600; flex-shrink: 0; }
.user-name { font-size: 14px; font-weight: 600; color: var(--text); }
.user-meta { display: flex; align-items: center; gap: 8px; margin-top: 4px; flex-wrap: wrap; }
.phone { font-size: 12px; color: var(--text-2); display: inline-flex; align-items: center; gap: 3px; }
.time { font-size: 12px; color: var(--text-2); }

.msg-ops { display: flex; align-items: center; gap: 8px; flex-shrink: 0; }

.msg-content { font-size: 14px; line-height: 1.8; color: var(--text); padding: 8px 0; border-top: 1px dashed #eef1f6; white-space: pre-wrap; word-break: break-all; }

.reply-area { margin-top: 12px; padding: 12px; background: #f7f9fc; border-radius: 8px; border: 1px solid var(--border); }
.reply-ops { display: flex; justify-content: flex-end; gap: 8px; margin-top: 8px; }

.reply-box { margin-top: 10px; padding: 10px 14px; background: #f0f6ff; border-radius: 8px; border-left: 3px solid var(--primary); }
.reply-label { font-size: 12px; color: var(--primary); font-weight: 600; margin-bottom: 4px; display: flex; align-items: center; gap: 4px; }
.reply-text { font-size: 13px; line-height: 1.7; color: var(--text); white-space: pre-wrap; word-break: break-all; }
.reply-time { font-size: 11px; color: var(--text-2); margin-top: 6px; }

@media (max-width: 900px) {
  .msg-layout { grid-template-columns: 1fr; }
  .header-inner { flex-direction: column; align-items: flex-start; }
}
</style>