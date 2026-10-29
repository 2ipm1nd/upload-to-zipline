<script setup lang="ts">
import { ref } from 'vue';

const emit = defineEmits<{
  (e: 'uploadSuccess', url: string): void;
  (e: 'uploadError', err: string): void;
  (e: 'uploadStart'): void;
}>();

const isCapturingVisible = ref(false);
const isReadingClipboard = ref(false);

async function onAreaSnip() {
  try {
    const [tab] = await browser.tabs.query({ active: true, currentWindow: true });
    if (!tab?.id) {
      emit('uploadError', '無法在當前分頁執行區域截圖。');
      return;
    }
    // Inform background to start area snip
    await browser.runtime.sendMessage({ action: 'startAreaSnip' });
    // Close popup so user can drag on the page
    window.close();
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : String(err);
    emit('uploadError', `啟動區域截圖失敗: ${msg}`);
  }
}

async function onCaptureVisible() {
  if (isCapturingVisible.value) return;
  isCapturingVisible.value = true;
  emit('uploadStart');
  try {
    const res = (await browser.runtime.sendMessage({ action: 'captureVisible' })) as {
      success: boolean;
      url?: string;
      error?: string;
    };
    if (res?.success && res.url) {
      emit('uploadSuccess', res.url);
    } else {
      emit('uploadError', res?.error || '截取可見畫面失敗。');
    }
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : String(err);
    emit('uploadError', `可見畫面截圖失敗: ${msg}`);
  } finally {
    isCapturingVisible.value = false;
  }
}

async function onUploadClipboard() {
  if (isReadingClipboard.value) return;
  isReadingClipboard.value = true;
  emit('uploadStart');

  try {
    let imageBlob: Blob | null = null;
    let filename = `clipboard-${Date.now()}.png`;

    // Try reading via navigator.clipboard in popup first
    if (navigator.clipboard?.read) {
      try {
        const items = await navigator.clipboard.read();
        for (const item of items) {
          for (const type of item.types) {
            if (type.startsWith('image/')) {
              imageBlob = await item.getType(type);
              filename = `clipboard-${Date.now()}.${type.split('/')[1] || 'png'}`;
              break;
            }
          }
          if (imageBlob) break;
        }
      } catch {
        // Fallback or permission prompt
      }
    }

    if (!imageBlob) {
      emit('uploadError', '剪貼簿中未偵測到圖片，請先截圖或複製圖片！');
      isReadingClipboard.value = false;
      return;
    }

    // Convert blob to dataUrl to send to background
    const dataUrl = await new Promise<string>((resolve, reject) => {
      const reader = new FileReader();
      reader.onloadend = () => resolve(reader.result as string);
      reader.onerror = reject;
      reader.readAsDataURL(imageBlob);
    });

    const res = (await browser.runtime.sendMessage({
      action: 'uploadBlob',
      dataUrl,
      filename,
    })) as { success: boolean; url?: string; error?: string };

    if (res?.success && res.url) {
      emit('uploadSuccess', res.url);
    } else {
      emit('uploadError', res?.error || '剪貼簿圖片上傳失敗。');
    }
  } catch (err: unknown) {
    const msg = err instanceof Error ? err.message : String(err);
    emit('uploadError', `上傳剪貼簿失敗: ${msg}`);
  } finally {
    isReadingClipboard.value = false;
  }
}
</script>

<template>
  <div class="grid grid-cols-3 gap-2">
    <button
      type="button"
      class="flex flex-col items-center justify-center gap-1.5 p-2.5 rounded-card bg-panel border border-border
             hover:border-accent hover:bg-[#202025] active:translate-y-px transition cursor-pointer group"
      title="框選網頁任意區域並上傳 (快捷鍵: Alt+Shift+S)"
      @click="onAreaSnip"
    >
      <span class="text-lg group-hover:scale-110 transition-transform">✂️</span>
      <span class="text-xs font-medium text-text group-hover:text-accent">區域截圖</span>
    </button>

    <button
      type="button"
      :disabled="isCapturingVisible"
      class="flex flex-col items-center justify-center gap-1.5 p-2.5 rounded-card bg-panel border border-border
             hover:border-accent hover:bg-[#202025] active:translate-y-px transition cursor-pointer group disabled:opacity-50"
      title="一鍵擷取目前可見視窗並上傳 (快捷鍵: Alt+Shift+V)"
      @click="onCaptureVisible"
    >
      <span class="text-lg group-hover:scale-110 transition-transform">
        {{ isCapturingVisible ? '⏳' : '🖥️' }}
      </span>
      <span class="text-xs font-medium text-text group-hover:text-accent">
        {{ isCapturingVisible ? '截圖中...' : '可見畫面' }}
      </span>
    </button>

    <button
      type="button"
      :disabled="isReadingClipboard"
      class="flex flex-col items-center justify-center gap-1.5 p-2.5 rounded-card bg-panel border border-border
             hover:border-accent hover:bg-[#202025] active:translate-y-px transition cursor-pointer group disabled:opacity-50"
      title="上傳剪貼簿中的截圖 (支援 Ctrl+V / 快捷鍵: Alt+Shift+U)"
      @click="onUploadClipboard"
    >
      <span class="text-lg group-hover:scale-110 transition-transform">
        {{ isReadingClipboard ? '⏳' : '📋' }}
      </span>
      <span class="text-xs font-medium text-text group-hover:text-accent">
        {{ isReadingClipboard ? '讀取中...' : '貼上剪貼簿' }}
      </span>
    </button>
  </div>
</template>

