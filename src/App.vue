<script setup>
import LivePusher from './components/LivePusher.vue'
</script>

<template>
  <div class="app">
    <header class="app-header">
      <span class="logo-icon">📡</span>
      <span class="title">腾讯云 CSS 直播推流</span>
      <span class="subtitle">WebRTC Push Demo</span>
    </header>
    <main class="app-main">
      <LivePusher />
    </main>
  </div>
</template>

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Microsoft YaHei', sans-serif;
  background: #f0f2f5;
  color: #333;
}
</style>

<style scoped>
.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.app-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 0 28px;
  height: 56px;
  background: #fff;
  border-bottom: 1px solid #e8e8e8;
  box-shadow: 0 1px 4px rgba(0,0,0,.06);
}

.logo-icon { font-size: 22px; }

.title {
  font-size: 16px;
  font-weight: 600;
  color: #1a1a1a;
}

.subtitle {
  font-size: 12px;
  color: #999;
  border-left: 1px solid #e0e0e0;
  padding-left: 10px;
  margin-left: 4px;
}

.app-main {
  flex: 1;
  padding: 24px;
}
</style>
