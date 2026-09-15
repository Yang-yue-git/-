<template>
  <div class="page">
    <!-- 顶部标题栏 -->
    <el-card shadow="never" class="header-card">
      <div class="header-inner">
        <div>
          <h1 class="page-title"><el-icon class="title-icon"><Document /></el-icon>签约项目管理系统</h1>
          <div class="sub-title">Vue3 · Element Plus 工程化版 · 多条件检索 · 灵活导出 · 数据导入</div>
        </div>
        <div class="header-actions">
          <el-button @click="downloadTemplate"><el-icon><Download /></el-icon>下载模板</el-button>
          <el-button @click="openImport"><el-icon><Upload /></el-icon>导入Excel</el-button>
          <el-button @click="triggerRestoreBackup"><el-icon><FolderOpened /></el-icon>恢复备份</el-button>
          <el-button :disabled="!statImported" @click="exportBackup"><el-icon><Wallet /></el-icon>导出备份</el-button>
          <el-button type="danger" plain :disabled="!statImported" @click="clearImported"><el-icon><Delete /></el-icon>清除导入</el-button>
          <el-button type="primary" @click="openExport"><el-icon><Download /></el-icon>导出表格</el-button>
          <input ref="backupFileInput" type="file" accept=".json" style="display:none" @change="restoreBackup">
        </div>
      </div>
    </el-card>

    <!-- 搜索区 -->
    <el-card shadow="never" class="section-card">
      <el-form class="search-form" :class="{ expanded: searchExpanded }">
        <el-form-item label="项目名称" class="grid-c1">
          <el-input v-model="search.projectName" placeholder="请输入项目名称关键字" clearable @keyup.enter="doSearch" />
        </el-form-item>
        <el-form-item label="项目编码" class="grid-c2">
          <el-input v-model="search.projectCode" placeholder="如 XM-2026-001" clearable @keyup.enter="doSearch" />
        </el-form-item>
        <el-form-item label="商机编码" class="grid-c3">
          <el-input v-model="search.opportunityCode" placeholder="如 SJ-2026-0101" clearable @keyup.enter="doSearch" />
        </el-form-item>
        <el-form-item label="签约时间" class="js-collapse grid-r2c1">
          <el-date-picker
            v-model="search.dateRange"
            type="daterange"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            value-format="YYYY-MM-DD"
            style="width:100%"
          />
        </el-form-item>
        <el-form-item label="项目经理" class="js-collapse grid-r2c2">
          <el-select v-model="search.manager" placeholder="全部" clearable style="width:100%">
            <el-option label="郭泽鹏" value="郭泽鹏" />
            <el-option label="鹿梦康" value="鹿梦康" />
            <el-option label="王雷" value="王雷" />
          </el-select>
        </el-form-item>
        <el-form-item class="search-btns">
          <el-button type="primary" @click="doSearch"><el-icon><Search /></el-icon>查询</el-button>
          <el-button @click="doReset"><el-icon><Refresh /></el-icon>重置</el-button>
          <el-button text type="primary" class="btn-toggle" @click="searchExpanded = !searchExpanded">
            {{ searchExpanded ? '收起' : '更多筛选' }}
            <el-icon class="toggle-arrow" :class="{ expanded: searchExpanded }"><ArrowDown /></el-icon>
          </el-button>
        </el-form-item>
      </el-form>
    </el-card>

    <!-- 统计条 -->
    <div class="stat-bar">
      <span>共 <b class="num">{{ allData.length }}</b> 条商机</span>
      <span class="divider"></span>
      <span>当前显示 <b class="num">{{ sortedData.length }}</b> 条</span>
      <span class="divider"></span>
      <span>其中导入数据 <b class="num">{{ statImported }}</b> 条</span>
    </div>

    <!-- 表格 -->
    <el-card shadow="never" class="section-card table-card">
      <el-table
        :data="pagedData"
        stripe
        border
        height="calc(100vh - 420px)"
        style="width:100%"
        @sort-change="onSortChange"
        empty-text="暂无数据，请调整搜索条件或导入数据"
      >
        <el-table-column prop="projectName" label="项目名称" min-width="220" show-overflow-tooltip />
        <el-table-column prop="projectCode" label="项目编码" width="130" />
        <el-table-column prop="opportunityCode" label="商机编码" width="130" />
        <el-table-column prop="customerName" label="客户名称" min-width="150" show-overflow-tooltip />
        <el-table-column prop="projectManager" label="项目经理" width="100" />
        <el-table-column prop="signDate" label="签约时间" width="115" />
        <el-table-column prop="amount" label="合同金额(万元)" width="140" align="right" sortable="custom" :formatter="fmtAmount" />
        <el-table-column prop="status" label="项目状态" width="105">
          <template #default="{ row }">
            <el-tag :type="statusTagType(row.status)" size="small">{{ row.status }}</el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="source" label="数据来源" width="95">
          <template #default="{ row }">
            <el-tag v-if="row.source === '导入'" type="warning" size="small" effect="light">导入</el-tag>
            <span v-else>系统</span>
          </template>
        </el-table-column>
      </el-table>

      <!-- 分页 -->
      <div class="pager-bar">
        <el-pagination
          :current-page="safePage"
          :page-size="pageSize"
          :page-sizes="[5, 10, 30]"
          :total="sortedData.length"
          layout="total, sizes, prev, pager, next, jumper"
          @size-change="onSizeChange"
          @current-change="goToPage"
        />
      </div>
    </el-card>

    <!-- 导出弹窗 -->
    <el-dialog v-model="exportMaskShow" title="导出表格 - 选择要导出的列" width="480px">
      <div class="check-grid">
        <el-checkbox v-for="col in COLUMNS" :key="col.key" v-model="exportChecks[col.key]">
          {{ col.label }}
        </el-checkbox>
      </div>
      <div class="tip-text">提示：默认勾选全部列，取消勾选后导出文件将只包含所选列。导出格式为 CSV（UTF-8），可用 Excel 直接打开。</div>
      <template #footer>
        <el-button @click="exportMaskShow = false">取消</el-button>
        <el-button type="primary" @click="doExport">确认导出</el-button>
      </template>
    </el-dialog>

    <!-- 导入弹窗 -->
    <el-dialog v-model="importMaskShow" title="导入数据" width="500px">
      <p class="import-desc">请选择通过【下载模板】填写好的 Excel 文件。导入后数据将<strong>追加</strong>到当前列表，来源标记为「导入」，可参与搜索与导出。</p>
      <div class="file-drop-area" @click="clickFileInput">
        <div class="upload-icon"><el-icon :size="32"><UploadFilled /></el-icon></div>
        <div class="upload-text"><strong>点击选择文件</strong>，支持 .xlsx / .xls 格式</div>
        <div class="file-name" v-if="importFileName">已选择：{{ importFileName }}</div>
        <input ref="fileInput" type="file" accept=".xlsx,.xls" style="display:none" @change="handleFile">
      </div>
      <el-alert
        v-if="importResult"
        :title="importResult"
        :type="importStatus === 'success' ? 'success' : 'error'"
        :closable="false"
        show-icon
        class="import-result"
      />
      <template #footer>
        <el-button :type="importStatus === 'success' ? 'primary' : 'default'" @click="importMaskShow = false">完成</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  Document, Download, Upload, UploadFilled, FolderOpened, Wallet,
  Delete, Search, Refresh, ArrowDown
} from '@element-plus/icons-vue'
import * as XLSX from 'xlsx'
import { seedData, COLUMNS, STATUS_MAP } from '../data.js'

// ===== 数据持久化常量 =====
const STORAGE_KEY_IMPORTED = 'vue3_imported_projects'
const STORAGE_KEY_PREFS = 'vue3_projects_prefs'
const STORAGE_VERSION = 2

// ===== 数据 =====
const allData = reactive((() => {
  const base = JSON.parse(JSON.stringify(seedData))
  try {
    const raw = localStorage.getItem(STORAGE_KEY_IMPORTED)
    if (!raw) return base
    const wrapper = JSON.parse(raw)
    let list = Array.isArray(wrapper) ? wrapper : (wrapper && wrapper.v === STORAGE_VERSION && Array.isArray(wrapper.d) ? wrapper.d : null)
    if (list && list.length) return base.concat(list)
  } catch (e) {
    console.warn('[持久化] 读取导入数据失败:', e.message)
  }
  return base
})())

const prefs = (() => {
  try { return JSON.parse(localStorage.getItem(STORAGE_KEY_PREFS) || '{}') } catch (e) { return {} }
})()
const search = reactive({ projectName: '', projectCode: '', opportunityCode: '', dateRange: null, manager: '' })
const searchExpanded = ref(false)
const sortKey = ref(null)
const sortDir = ref('ascending')
const pageSize = ref(prefs.pageSize || 10)
const currentPage = ref(1)
const exportMaskShow = ref(false)
const exportChecks = reactive({})
const importMaskShow = ref(false)
const importResult = ref('')
const importStatus = ref('')
const importFileName = ref('')
const fileInput = ref(null)
const backupFileInput = ref(null)

// ===== 计算属性 =====
const filteredData = computed(() => {
  const s = search
  return allData.filter(r => {
    if (s.projectName && !r.projectName.toLowerCase().includes(s.projectName.trim().toLowerCase())) return false
    if (s.projectCode && !r.projectCode.toLowerCase().includes(s.projectCode.trim().toLowerCase())) return false
    if (s.opportunityCode && !r.opportunityCode.toLowerCase().includes(s.opportunityCode.trim().toLowerCase())) return false
    if (s.dateRange && s.dateRange.length === 2) {
      if (r.signDate < s.dateRange[0]) return false
      if (r.signDate > s.dateRange[1]) return false
    }
    if (s.manager && r.projectManager !== s.manager) return false
    return true
  })
})

const sortedData = computed(() => {
  if (!sortKey.value || !sortDir.value) return filteredData.value
  const k = sortKey.value, d = sortDir.value === 'ascending' ? 1 : -1
  return [...filteredData.value].sort((a, b) => {
    let va = a[k], vb = b[k]
    if (k === 'amount') { va = Number(va); vb = Number(vb) }
    return va < vb ? -1 * d : va > vb ? 1 * d : 0
  })
})

const totalPages = computed(() => Math.max(1, Math.ceil(sortedData.value.length / pageSize.value)))
const safePage = computed(() => Math.min(Math.max(1, currentPage.value), totalPages.value))
const pagedData = computed(() => {
  const start = (safePage.value - 1) * pageSize.value
  return sortedData.value.slice(start, start + pageSize.value)
})

const statImported = computed(() => allData.filter(d => d.source === '导入').length)

watch(pageSize, () => { currentPage.value = 1; savePrefs() })

// ===== 方法 =====
function statusTagType(s) { return STATUS_MAP[s] || 'success' }
function fmtAmount(row, col, val) { return Number(val).toLocaleString() }

function onSortChange({ prop, order }) {
  sortKey.value = prop
  sortDir.value = order || null
}

function doSearch() { currentPage.value = 1; ElMessage.success('查询完成') }
function doReset() {
  Object.assign(search, { projectName: '', projectCode: '', opportunityCode: '', dateRange: null, manager: '' })
  sortKey.value = null; sortDir.value = null; currentPage.value = 1
}

function onSizeChange(size) { pageSize.value = size }
function goToPage(p) { if (p >= 1 && p <= totalPages.value) currentPage.value = p }

function savePrefs() {
  try {
    const payload = { pageSize: pageSize.value }
    localStorage.setItem(STORAGE_KEY_PREFS, JSON.stringify(payload))
  } catch (e) {
    console.warn('[持久化] 保存偏好失败:', e.message)
  }
}

function openExport() {
  COLUMNS.forEach(c => exportChecks[c.key] = true)
  exportMaskShow.value = true
}

function doExport() {
  const keys = COLUMNS.filter(c => exportChecks[c.key]).map(c => c.key)
  if (!keys.length) { ElMessage.warning('请至少勾选一列'); return }
  if (!sortedData.value.length) { ElMessage.warning('当前没有可导出的数据'); return }
  const cols = COLUMNS.filter(c => keys.includes(c.key))
  const head = cols.map(c => c.label)
  const rows = sortedData.value.map(r => cols.map(c => {
    let v = r[c.key] == null ? '' : String(r[c.key])
    if (/[",\n]/.test(v)) v = '"' + v.replace(/"/g, '""') + '"'
    return v
  }))
  const csv = '\uFEFF' + [head, ...rows].map(row => row.join(',')).join('\r\n')
  downloadBlob('签约项目导出_' + Date.now() + '.csv', csv, 'text/csv;charset=utf-8')
  exportMaskShow.value = false
  ElMessage.success('已导出 ' + sortedData.value.length + ' 条数据')
}

function downloadTemplate() {
  const head = COLUMNS.map(c => c.label)
  const example = COLUMNS.map(c => c.key === 'projectName' ? '示例：银行智能合约平台' :
    c.key === 'projectCode' ? 'XM-2026-100' :
    c.key === 'opportunityCode' ? 'SJ-2026-0200' :
    c.key === 'customerName' ? '示例客户有限公司' :
    c.key === 'projectManager' ? '郭泽鹏' :
    c.key === 'signDate' ? '2026-09-30' :
    c.key === 'amount' ? '500' :
    c.key === 'status' ? '已签约' : '模拟')
  const ws = XLSX.utils.aoa_to_sheet([head, example])
  ws['!cols'] = COLUMNS.map(c => ({ wch: Math.max(c.label.length * 2 + 6, 14) }))
  const wb = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(wb, ws, '签约项目模板')
  XLSX.writeFile(wb, '签约项目导入模板.xlsx')
  ElMessage.success('模板已下载（Excel），请按表头填入数据后导入')
}

function openImport() { importResult.value = ''; importStatus.value = ''; importFileName.value = ''; importMaskShow.value = true }
function clickFileInput() { fileInput.value && fileInput.value.click() }

function saveImported() {
  try {
    const imported = allData.filter(d => d.source === '导入')
    const payload = JSON.stringify({ v: STORAGE_VERSION, d: imported, t: Date.now() })
    localStorage.setItem(STORAGE_KEY_IMPORTED, payload)
  } catch (e) {
    console.warn('[持久化] 保存导入数据失败:', e.message)
  }
}

function handleFile(e) {
  const file = e.target.files[0]
  if (!file) return
  importFileName.value = file.name
  importStatus.value = ''
  importResult.value = ''
  const reader = new FileReader()
  reader.onload = (ev) => {
    try {
      const wb = XLSX.read(new Uint8Array(ev.target.result), { type: 'array', cellDates: true })
      const ws = wb.Sheets[wb.SheetNames[0]]
      const rows = XLSX.utils.sheet_to_json(ws, { header: 1, defval: '' })
      if (!rows.length || rows.length < 2) throw new Error('文件内容为空或没有数据行')
      const headMap = {}
      COLUMNS.forEach(c => headMap[c.label] = c.key)
      const header = rows[0].map(h => headMap[h] != null ? headMap[h] : null)
      if (header.every(h => h === null)) throw new Error('表头与模板不匹配，请使用下载的模板格式')

      const existingKeys = new Set(allData.map(d => (d.projectCode || '') + '|' + (d.opportunityCode || '')))
      let added = 0, skipped = 0
      for (let i = 1; i < rows.length; i++) {
        const r = rows[i]
        if (!r.join('')) continue
        const obj = {}
        header.forEach((key, idx) => {
          if (!key) return
          let v = r[idx]
          if (v instanceof Date) v = v.getFullYear() + '-' + String(v.getMonth() + 1).padStart(2, '0') + '-' + String(v.getDate()).padStart(2, '0')
          obj[key] = String(v == null ? '' : v).trim()
        })
        if (!obj.projectName) continue
        const dupKey = (obj.projectCode || '') + '|' + (obj.opportunityCode || '')
        if (dupKey !== '|' && existingKeys.has(dupKey)) { skipped++; continue }
        obj.amount = obj.amount || '0'
        obj.status = obj.status || '已签约'
        obj.source = '导入'
        allData.push(obj)
        existingKeys.add(dupKey)
        added++
      }
      if (!added) {
        if (skipped) throw new Error('全部 ' + skipped + ' 条数据已存在于系统中，未新增任何数据')
        throw new Error('没有解析到有效数据行')
      }
      currentPage.value = 999999
      saveImported()
      let html = '成功导入 <strong>' + added + '</strong> 条数据'
      if (skipped) html += '，跳过重复 <strong>' + skipped + '</strong> 条'
      html += '。数据已自动保存到本地，刷新页面不会丢失。'
      importStatus.value = 'success'
      importResult.value = html
      ElMessage.success('成功导入 ' + added + ' 条' + (skipped ? '，跳过 ' + skipped + ' 条重复' : ''))
    } catch (err) {
      importStatus.value = 'error'
      importResult.value = '导入失败：' + err.message
      ElMessage.error('导入失败')
    }
    e.target.value = ''
  }
  reader.readAsArrayBuffer(file)
}

function downloadBlob(name, content, type) {
  const blob = new Blob([content], { type })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url; a.download = name
  document.body.appendChild(a); a.click()
  document.body.removeChild(a); URL.revokeObjectURL(url)
}

// ===== 备份 / 恢复 / 清除 =====
function exportBackup() {
  const imported = allData.filter(d => d.source === '导入')
  if (!imported.length) { ElMessage.warning('当前没有导入数据可备份'); return }
  const payload = JSON.stringify({ v: STORAGE_VERSION, exportedAt: new Date().toISOString(), count: imported.length, data: imported }, null, 2)
  downloadBlob('签约项目导入数据备份_' + new Date().toISOString().slice(0, 10) + '.json', payload, 'application/json')
  ElMessage.success('已导出备份（' + imported.length + ' 条）')
}

function triggerRestoreBackup() { backupFileInput.value && backupFileInput.value.click() }
function restoreBackup(e) {
  const file = e.target.files[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    try {
      const wrapper = JSON.parse(ev.target.result)
      const list = Array.isArray(wrapper) ? wrapper : wrapper.data
      if (!Array.isArray(list) || !list.length) throw new Error('备份文件为空或格式不正确')
      const existingKeys = new Set(allData.map(d => (d.projectCode || '') + '|' + (d.opportunityCode || '')))
      let added = 0, skipped = 0
      for (const item of list) {
        if (!item || !item.projectName) continue
        const dupKey = (item.projectCode || '') + '|' + (item.opportunityCode || '')
        if (dupKey !== '|' && existingKeys.has(dupKey)) { skipped++; continue }
        item.source = '导入'
        allData.push(item)
        existingKeys.add(dupKey)
        added++
      }
      if (!added) throw new Error('备份中 ' + skipped + ' 条数据均已存在，未新增')
      saveImported()
      ElMessage.success('恢复成功：新增 ' + added + ' 条' + (skipped ? '，跳过 ' + skipped + ' 条重复' : ''))
    } catch (err) {
      ElMessage.error('恢复失败：' + err.message)
    }
    e.target.value = ''
  }
  reader.readAsText(file, 'utf-8')
}

function clearImported() {
  const cnt = statImported.value
  if (!cnt) { ElMessage.warning('当前没有导入数据'); return }
  ElMessageBox.confirm('确定要清除全部 ' + cnt + ' 条导入数据吗？此操作不可撤销（建议先导出备份）。', '清除确认', {
    confirmButtonText: '确定清除',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(() => {
    for (let i = allData.length - 1; i >= 0; i--) {
      if (allData[i].source === '导入') allData.splice(i, 1)
    }
    saveImported()
    currentPage.value = 1
    ElMessage.success('已清除 ' + cnt + ' 条导入数据')
  }).catch(() => {})
}

onMounted(() => {
  const imported = allData.filter(d => d.source === '导入')
  if (imported.length) {
    const savedRaw = localStorage.getItem(STORAGE_KEY_IMPORTED)
    if (savedRaw) {
      try {
        const w = JSON.parse(savedRaw)
        const t = w.t ? new Date(w.t) : null
        const timeStr = t ? t.toLocaleString() : ''
        setTimeout(() => ElMessage.success('已从本地恢复 ' + imported.length + ' 条导入数据' + (timeStr ? '（上次保存：' + timeStr + '）' : '')), 400)
      } catch (e) {
        setTimeout(() => ElMessage.success('已恢复 ' + imported.length + ' 条导入数据'), 400)
      }
    }
  }
})
</script>

<style scoped>
.page { padding: 20px 24px 40px; }

.header-card { border: 1px solid var(--border); border-radius: 12px; margin-bottom: 16px; }
.header-card :deep(.el-card__body) { padding: 14px 22px; }
.header-inner { display: flex; align-items: center; justify-content: space-between; gap: 16px; flex-wrap: wrap; }
.page-title { font-size: 20px; font-weight: 600; display: flex; align-items: center; gap: 10px; }
.title-icon { color: var(--primary); font-size: 22px; }
.sub-title { color: var(--text-2); font-size: 12px; margin-top: 4px; }
.header-actions { display: flex; gap: 10px; flex-wrap: wrap; }

.section-card { border: 1px solid var(--border); border-radius: 12px; margin-bottom: 16px; }
/* 始终用 grid 布局，三列等宽 + 第四列放按钮，展开/收起第一行不动 */
.search-form { display: grid; grid-template-columns: 1fr 1fr 1fr auto; gap: 16px 44px; align-items: end; }
.search-form :deep(.el-form-item) { margin-bottom: 0; margin-right: 0; }
.search-form :deep(.el-form-item__content) { width: 100%; }
.search-form :deep(.el-input), .search-form :deep(.el-select), .search-form :deep(.el-date-editor) { width: 100% !important; }
/* 第一行固定位置 */
.search-form :deep(.grid-c1) { grid-column: 1; grid-row: 1; }
.search-form :deep(.grid-c2) { grid-column: 2; grid-row: 1; }
.search-form :deep(.grid-c3) { grid-column: 3; grid-row: 1; }
/* 第二行：签约时间、项目经理 */
.search-form :deep(.grid-r2c1) { grid-column: 1; grid-row: 2; }
.search-form :deep(.grid-r2c2) { grid-column: 2; grid-row: 2; }
/* 按钮：始终固定在第一行第四列，展开/收起时第一行不动 */
.search-form :deep(.search-btns) { grid-column: 4; grid-row: 1; justify-self: end; }
/* 收起时隐藏第二行 */
.search-form:not(.expanded) .js-collapse { display: none; }
.btn-toggle { font-size: 13px; padding: 0 4px; }
.btn-toggle :deep(.el-icon) { margin-left: 2px; }
.toggle-arrow { transition: transform 0.2s; }
.toggle-arrow.expanded { transform: rotate(180deg); }

.stat-bar { display: flex; align-items: center; gap: 10px; padding: 0 2px 14px; color: var(--text-2); font-size: 13px; flex-wrap: wrap; }
.stat-num, .stat-bar b { color: var(--primary); font-weight: 700; font-size: 15px; }
.divider { width: 1px; height: 14px; background: var(--border); }

.table-card :deep(.el-card__body) { padding: 0; }
.table-card { overflow: hidden; }
.pager-bar { display: flex; justify-content: flex-end; padding: 14px 16px; border-top: 1px solid var(--border); background: #fafbfe; }

.check-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
.tip-text { font-size: 12px; color: var(--text-2); line-height: 1.7; margin-top: 12px; background: #f7f9fc; border-radius: 8px; padding: 10px 12px; }

.import-desc { font-size: 13px; line-height: 1.8; color: #374151; margin-bottom: 4px; }
.file-drop-area { border: 2px dashed #c9d4e2; border-radius: 10px; padding: 26px 16px; text-align: center; cursor: pointer; transition: all .15s; background: #fafbfe; }
.file-drop-area:hover { border-color: var(--primary); background: var(--primary-light); }
.upload-icon { font-size: 32px; color: var(--primary); margin-bottom: 8px; display: flex; justify-content: center; }
.upload-text { font-size: 13px; color: var(--text-2); }
.upload-text strong { color: var(--primary); }
.file-name { font-size: 12px; color: var(--primary); margin-top: 8px; font-weight: 500; }
.import-result { margin-top: 14px; }
</style>