<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { fetchEventSource, EventStreamContentType } from '@microsoft/fetch-event-source';

// ===== 联调配置：按实际环境修改这三项 =====
// 开发环境使用 Vite 代理，保持为空可避免浏览器 CORS 预检。
// 如果前端和后端已经部署在同一个域名下，也保持为空。
const API_BASE_URL = '';
const API_PREFIX = '/kczyclYfgl';
const token = ref(sessionStorage.getItem('yfgl-demo-token') || '');
const tokenInput = ref(token.value);

const xmbh = ref(localStorage.getItem('yfgl-demo-xmbh') || '');
const progress = ref(null);
const results = ref([]);
const busy = ref(false);
const loadingResults = ref(false);
const errorMessage = ref('');
const connectionState = ref('disconnected');
const streamController = ref(null);
const tokenReady = computed(() => token.value.trim().length > 0);

const apiUrl = (path) => `${API_BASE_URL}${API_PREFIX}${path}`;
const isRunning = computed(() => ['queued', 'running', 'calculating'].includes(progress.value?.status));
const isFinished = computed(() => ['done', 'fail', 'cancelled'].includes(progress.value?.status));
const statusText = computed(() => ({
  queued: '排队中',
  running: '等待内网结果回传',
  calculating: '正在计算压覆面积',
  done: '分析完成',
  fail: '分析失败',
  cancelled: '已取消',
  unknown: '未找到任务'
}[progress.value?.status] || '未启动'));
const connectionText = computed(() => ({
  connected: 'SSE 已连接',
  reconnecting: 'SSE 断线，正在重连',
  disconnected: 'SSE 未连接'
}[connectionState.value]));

function authHeaders() {
  const rawToken = token.value.trim();
  const authorization = /^Bearer\s+/i.test(rawToken) ? rawToken : `Bearer ${rawToken}`;
  return {
    Authorization: authorization
  };
}

function saveToken() {
  const value = tokenInput.value.trim();
  if (!value) {
    errorMessage.value = '请输入 Authorization token';
    return;
  }
  token.value = value;
  sessionStorage.setItem('yfgl-demo-token', value);
  errorMessage.value = '';
}

function clearToken() {
  stopStream();
  token.value = '';
  tokenInput.value = '';
  sessionStorage.removeItem('yfgl-demo-token');
}

async function readJson(response) {
  const body = await response.json();
  if (!response.ok || body.code >= 500) {
    throw new Error(body.message || `请求失败：${response.status}`);
  }
  return body;
}

async function startAnalysis() {
  if (!xmbh.value.trim()) {
    errorMessage.value = '请输入项目编号';
    return;
  }

  busy.value = true;
  errorMessage.value = '';
  localStorage.setItem('yfgl-demo-xmbh', xmbh.value.trim());
  try {
    const response = await fetch(apiUrl(`/analysisNew?xmbh=${encodeURIComponent(xmbh.value.trim())}`), {
      method: 'POST',
      headers: authHeaders()
    });
    const body = await readJson(response);
    if (body.data !== true) {
      throw new Error(body.message || '分析启动失败');
    }
    await loadProgress();
    // SSE 是持续连接，不能 await，否则 busy 会一直保持 true，取消按钮无法点击。
    connectStream();
  } catch (error) {
    errorMessage.value = error.message || '分析启动失败';
  } finally {
    busy.value = false;
  }
}

async function loadProgress() {
  if (!xmbh.value.trim()) return;
  const response = await fetch(apiUrl(`/analysis/progress?xmbh=${encodeURIComponent(xmbh.value.trim())}`), {
    headers: authHeaders()
  });
  const body = await readJson(response);
  progress.value = body.data || null;
}

async function connectStream() {
  if (!xmbh.value.trim()) return;
  if (streamController.value) {
    streamController.value.abort();
  }

  const controller = new AbortController();
  streamController.value = controller;
  connectionState.value = 'reconnecting';

  try {
    await fetchEventSource(apiUrl(`/analysis/progress/stream?xmbh=${encodeURIComponent(xmbh.value.trim())}`), {
      method: 'GET',
      headers: {
        ...authHeaders(),
        Accept: 'text/event-stream'
      },
      signal: controller.signal,
      openWhenHidden: true,
      retryDelay: 2000,

      async onopen(response) {
        const contentType = response.headers.get('content-type') || '';
        if (!response.ok || !contentType.includes(EventStreamContentType)) {
          throw new Error(`SSE 连接失败：${response.status}`);
        }
        connectionState.value = 'connected';
        await loadProgress();
      },

      onmessage(event) {
        if (!event.data) return;
        try {
          progress.value = JSON.parse(event.data);
          if (['done', 'fail', 'cancelled'].includes(progress.value.status)) {
            controller.abort();
            connectionState.value = 'disconnected';
          }
        } catch {
          errorMessage.value = '收到无法解析的进度消息';
        }
      },

      onclose() {
        if (!['done', 'fail', 'cancelled'].includes(progress.value?.status)) {
          connectionState.value = 'reconnecting';
          throw new Error('SSE 连接被提前关闭');
        }
      },

      onerror(error) {
        connectionState.value = 'reconnecting';
        if (controller.signal.aborted) return;
        console.warn('SSE 暂时断开，将自动重连', error);
        throw error;
      }
    });
  } catch (error) {
    if (!controller.signal.aborted) {
      connectionState.value = 'reconnecting';
      errorMessage.value = error.message || 'SSE 连接失败';
    }
  }
}

async function cancelAnalysis() {
  if (!xmbh.value.trim() || !isRunning.value) return;
  busy.value = true;
  errorMessage.value = '';
  try {
    const response = await fetch(apiUrl(`/analysis/cancel?xmbh=${encodeURIComponent(xmbh.value.trim())}`), {
      method: 'POST',
      headers: authHeaders()
    });
    await readJson(response);
    await loadProgress();
    await loadResults();
  } catch (error) {
    errorMessage.value = error.message || '取消失败';
  } finally {
    busy.value = false;
  }
}

async function loadResults() {
  if (!xmbh.value.trim()) return;
  loadingResults.value = true;
  try {
    const response = await fetch(apiUrl(`/analysis/results?xmbh=${encodeURIComponent(xmbh.value.trim())}`), {
      headers: authHeaders()
    });
    const body = await readJson(response);
    results.value = body.data || [];
  } catch (error) {
    errorMessage.value = error.message || '结果查询失败';
  } finally {
    loadingResults.value = false;
  }
}

function stopStream() {
  streamController.value?.abort();
  streamController.value = null;
  connectionState.value = 'disconnected';
}

onMounted(async () => {
  if (!tokenReady.value) return;
  if (!xmbh.value.trim()) return;
  try {
    await loadProgress();
    if (isRunning.value) {
      await connectStream();
    }
  } catch (error) {
    errorMessage.value = error.message || '恢复任务失败';
  }
});

onBeforeUnmount(stopStream);
</script>

<template>
  <main class="page-shell">
    <div v-if="!tokenReady" class="token-mask">
      <section class="token-dialog">
        <p class="eyebrow dark-eyebrow">Authorization</p>
        <h2>输入访问 Token</h2>
        <p class="hint">Demo 不调用登录接口，输入 token 即可，程序会自动添加 <code>Bearer </code> 前缀。</p>
        <input v-model="tokenInput" type="password" placeholder="请输入 token，也可输入 Bearer token" @keyup.enter="saveToken" />
        <button class="primary token-button" @click="saveToken">进入 Demo</button>
        <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
      </section>
    </div>

    <section class="card hero-card">
      <div>
        <p class="eyebrow">zrzy-ahlk</p>
        <h1>压覆分析进度 Demo</h1>
        <p class="subline">启动后台分析，实时查看内网分批回传的图层结果。</p>
      </div>
      <span class="connection" :class="connectionState">{{ connectionText }}</span>
    </section>

    <section class="card controls-card">
      <label for="xmbh">项目编号</label>
      <div class="control-row">
        <input id="xmbh" v-model="xmbh" placeholder="请输入 xmbh，例如 A001" :disabled="isRunning" />
        <button class="primary" :disabled="busy || isRunning" @click="startAnalysis">开始分析</button>
        <button class="danger" :disabled="busy || !isRunning" @click="cancelAnalysis">取消分析</button>
      </div>
      <p class="hint">Authorization token 由页面启动时输入，不调用登录接口。</p>
      <div class="token-actions">
        <span class="hint">当前 token 已保存在本次浏览器会话中。</span>
        <button class="secondary" @click="clearToken">更换 Token</button>
      </div>
      <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    </section>

    <section class="card progress-card">
      <div class="section-title">
        <h2>分析进度</h2>
        <span v-if="progress" class="status" :data-status="progress.status">{{ statusText }}</span>
      </div>

      <template v-if="progress">
        <div class="progress-number">{{ progress.percent }}<small>%</small></div>
        <div class="progress-track"><div class="progress-fill" :style="{ width: `${progress.percent}%` }"></div></div>
        <div class="progress-meta"><span>{{ progress.done }} / {{ progress.total }} 个图层</span><span>{{ progress.message }}</span></div>

        <div class="layer-grid">
          <div>
            <h3>已完成图层</h3>
            <ul v-if="progress.completedLayers?.length" class="layer-list done-list">
              <li v-for="layer in progress.completedLayers" :key="layer">✓ {{ layer }}</li>
            </ul>
            <p v-else class="empty">暂无</p>
          </div>
          <div>
            <h3>待回传图层</h3>
            <ul v-if="progress.pendingLayers?.length" class="layer-list pending-list">
              <li v-for="layer in progress.pendingLayers" :key="layer">… {{ layer }}</li>
            </ul>
            <p v-else class="empty">暂无</p>
          </div>
        </div>
        <p class="last-progress">最近完成：{{ progress.lastCompletedLayer || '暂无' }}</p>
      </template>
      <p v-else class="empty large-empty">还没有启动分析。</p>
    </section>

    <section class="card results-card">
      <div class="section-title">
        <h2>已同步结果</h2>
        <button class="secondary" :disabled="loadingResults || !xmbh" @click="loadResults">刷新结果</button>
      </div>
      <p class="hint">取消分析后，这里仍会保留已经回传并同步成功的图层结果。</p>
      <div v-if="results.length" class="table-wrap">
        <table>
          <thead><tr><th>图层</th><th>压覆面积（原始值）</th><th>是否有效</th></tr></thead>
          <tbody><tr v-for="item in results" :key="`${item.projid}-${item.layername}`"><td>{{ item.layername }}</td><td>{{ item.invalidarea }}</td><td>{{ item.isvalid }}</td></tr></tbody>
        </table>
      </div>
      <p v-else class="empty">暂无已同步结果。</p>
    </section>
  </main>
</template>
