<template>
  <el-container class="app-shell">
    <!-- 左侧导航栏 -->
    <el-aside width="232px" class="sidebar">
      <div class="brand">
        <div class="brand-logo">管</div>
        <div class="brand-text">
          <div class="brand-name">项目管理系统</div>
          <div class="brand-sub">Vue3 · Element Plus</div>
        </div>
      </div>
      <el-menu
        class="nav-menu"
        :default-active="activeTab"
        @select="onSelect"
      >
        <el-menu-item v-for="tab in tabs" :key="tab.key" :index="tab.key">
          <el-icon><component :is="tab.icon" /></el-icon>
          <span>{{ tab.label }}</span>
        </el-menu-item>
      </el-menu>
      <div class="sidebar-footer">v2.0 · 2026</div>
    </el-aside>

    <!-- 右侧内容区 -->
    <el-main class="main-content">
      <component :is="currentComponent" />
    </el-main>
  </el-container>
</template>

<script setup>
import { ref, computed } from 'vue'
import { Document, ChatDotRound } from '@element-plus/icons-vue'
import SignProject from './components/SignProject.vue'
import MessageBoard from './components/MessageBoard.vue'

const activeTab = ref('sign')

const tabs = [
  { key: 'sign', label: '签约项目管理', icon: Document },
  { key: 'message', label: '客户留言系统', icon: ChatDotRound }
]

const componentMap = {
  sign: SignProject,
  message: MessageBoard
}

const currentComponent = computed(() => componentMap[activeTab.value])

function onSelect(key) {
  activeTab.value = key
}
</script>

<style>
:root {
  --primary: #2f6fed; --primary-dark: #1e56cc; --primary-light: #eaf1ff;
  --bg: #f4f7fc; --card-bg: #fff; --border: #e2e8f2;
  --text: #1f2937; --text-2: #6b7280; --danger: #e5484d; --ok: #16a34a;
  --sidebar-bg: #1e2a42;
}
* { margin: 0; padding: 0; box-sizing: border-box; }
html, body, #app { height: 100%; }
body {
  font-family: "Microsoft YaHei","PingFang SC","Segoe UI",sans-serif;
  background: var(--bg); color: var(--text); font-size: 14px;
}
body.resizing { cursor: col-resize; user-select: none; }
::-webkit-scrollbar { width: 9px; height: 9px; }
::-webkit-scrollbar-thumb { background: #c9d4e2; border-radius: 6px; }
::-webkit-scrollbar-thumb:hover { background: #a9b8cc; }
::-webkit-scrollbar-track { background: transparent; }
</style>

<style scoped>
.app-shell { height: 100vh; }

/* 侧边栏 */
.sidebar {
  background: var(--sidebar-bg);
  display: flex; flex-direction: column;
  overflow: hidden;
}
.brand {
  display: flex; align-items: center; gap: 12px;
  padding: 22px 20px;
  border-bottom: 1px solid rgba(255,255,255,.08);
  flex-shrink: 0;
}
.brand-logo {
  width: 40px; height: 40px; border-radius: 10px;
  background: linear-gradient(135deg,#2f6fed,#5aa0ff);
  color: #fff;
  display: inline-flex; align-items: center; justify-content: center;
  font-size: 20px; font-weight: 700; flex-shrink: 0;
}
.brand-name { font-size: 16px; font-weight: 600; color: #fff; }
.brand-sub { font-size: 12px; color: rgba(255,255,255,.5); margin-top: 2px; }

.nav-menu {
  flex: 1;
  border-right: none;
  background: transparent;
  padding: 14px 12px;
}
.nav-menu :deep(.el-menu-item) {
  color: rgba(255,255,255,.65);
  border-radius: 10px;
  margin-bottom: 6px;
  height: 48px;
  transition: all .18s;
}
.nav-menu :deep(.el-menu-item:hover) {
  background: rgba(255,255,255,.06);
  color: #fff;
}
.nav-menu :deep(.el-menu-item.is-active) {
  background: #2f6fed;
  color: #fff;
  box-shadow: 0 4px 14px rgba(47,111,237,.35);
}
.nav-menu :deep(.el-menu-item .el-icon) {
  font-size: 18px;
  margin-right: 10px;
}

.sidebar-footer {
  padding: 16px 20px;
  border-top: 1px solid rgba(255,255,255,.08);
  font-size: 12px;
  color: rgba(255,255,255,.35);
  flex-shrink: 0;
}

/* 主内容区 */
.main-content {
  padding: 0;
  background: var(--bg);
  overflow-y: auto;
}
</style>