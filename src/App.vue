<template>
  <div id="app-container">
    <div class="container">
      <h1>AI 视频换脸工具</h1>
      <p>上传一张源图片和目标视频，即可体验换脸效果。</p>

      <div class="form-group">
        <label for="swap-image">源图片 (包含人脸)</label>
        <input type="file" id="swap-image" @change="handleSourceImageUpload" accept="image/png, image/jpeg">
      </div>

      <div class="form-group">
        <label for="target-video">目标视频</label>
        <input type="file" id="target-video" @change="handleTargetVideoUpload" accept="video/mp4">
      </div>

      <button @click="startFaceSwap" :disabled="isLoading">
        {{ isLoading ? '处理中...' : '开始换脸' }}
      </button>

      <div id="status">{{ statusMessage }}</div>
      <div class="loader" v-if="isLoading"></div>

      <div id="result" v-if="resultUrl">
        <video controls :src="resultUrl"></video>
        <a :href="resultUrl" download="result.mp4">下载视频</a>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';

// --- 响应式状态定义 ---
const sourceImageFile = ref(null);
const targetVideoFile = ref(null);
const isLoading = ref(false);
const statusMessage = ref('');
const resultUrl = ref('');

// --- API 配置 ---
const API_TOKEN = 'f56ec386ae935002f1a9643893ebd6bc44a4d75b';
const CORS_PROXY = "https://proxy.cors.sh/";
const REPLICATE_API_URL = CORS_PROXY + "https://api.replicate.com/v1/predictions";
const REPLICATE_FILES_URL = CORS_PROXY + "https://api.replicate.com/v1/files";


// --- 事件处理函数 ---
const handleSourceImageUpload = (event) => {
  sourceImageFile.value = event.target.files[0];
};

const handleTargetVideoUpload = (event) => {
  targetVideoFile.value = event.target.files[0];
};

// --- 核心业务逻辑方法 ---
const startFaceSwap = async () => {
  if (!sourceImageFile.value || !targetVideoFile.value) {
    statusMessage.value = '错误：请同时上传源图片和目标视频。';
    return;
  }

  // 重置状态
  isLoading.value = true;
  statusMessage.value = '准备开始...';
  resultUrl.value = '';

  try {
    // 1. 上传文件
    statusMessage.value = '正在上传源图片...';
    const swapImageUrl = await uploadFile(sourceImageFile.value);

    statusMessage.value = '源图片上传成功！正在上传目标视频...';
    const targetVideoUrl = await uploadFile(targetVideoFile.value);

    // 2. 创建预测任务
    statusMessage.value = '文件上传完成！正在创建换脸任务...';
    const prediction = await createPrediction(swapImageUrl, targetVideoUrl);

    // 3. 轮询任务状态
    statusMessage.value = '任务已创建，正在处理中... 请耐心等待。';
    await pollPredictionStatus(prediction);

  } catch (error) {
    console.error(error);
    statusMessage.value = `发生错误: ${error.message}`;
  } finally {
    isLoading.value = false; // 无论成功失败，都结束加载状态
  }
};

const uploadFile = async (file) => {
  const uploadUrlResponse = await fetch(REPLICATE_FILES_URL, {
    method: 'POST',
    headers: {
      'Authorization': `Token ${API_TOKEN}`,
      'Content-Type': 'application/json'
    },
    // ###################################
    // ###     这是本次修正的关键     ###
    // ###################################
    body: JSON.stringify({
      'file_name': file.name, // 必须是 'file_name'
      'content_type': file.type
    })
  });

  const uploadUrlData = await uploadUrlResponse.json();
  if (uploadUrlResponse.status > 299) {
    throw new Error(`获取上传 URL 失败: ${uploadUrlData.detail}`);
  }

  const { serving_url, upload_url } = uploadUrlData;

  await fetch(upload_url, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file
  });

  return serving_url;
};

const createPrediction = async (swapImageUrl, targetVideoUrl) => {
  const response = await fetch(REPLICATE_API_URL, {
    method: 'POST',
    headers: {
      'Authorization': `Token ${API_TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      version: "116bbf0f4e14d808f655e87e5448233cceff10a45f659d71539cafb7163b2e84",
      input: {
        swap_image: swapImageUrl,
        target_video: targetVideoUrl
      }
    })
  });

  const prediction = await response.json();
  if (response.status !== 201) {
    throw new Error(prediction.detail || '创建任务失败。');
  }
  return prediction;
};

const pollPredictionStatus = async (prediction) => {
  const getUrl = CORS_PROXY + `https://api.replicate.com/v1/predictions/${prediction.id}`;

  while (prediction.status !== 'succeeded' && prediction.status !== 'failed') {
    await new Promise(resolve => setTimeout(resolve, 4000)); // 等待 4 秒

    const response = await fetch(getUrl, {
      headers: { 'Authorization': `Token ${API_TOKEN}` }
    });

    prediction = await response.json();
    if (response.status > 299) {
      throw new Error(prediction.detail || '查询任务状态失败。');
    }
    statusMessage.value = `任务状态: ${prediction.status}...`;
  }

  if (prediction.status === 'succeeded') {
    statusMessage.value = '换脸成功！';
    resultUrl.value = prediction.output; // 更新结果 URL 以显示视频
  } else {
    throw new Error(`任务失败: ${prediction.error}`);
  }
};
</script>

<style>
/* 样式部分没有变化 */
#app-container {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    background-color: #f4f7f6;
    color: #333;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    margin: 0;
    padding: 20px;
}
.container {
    background-color: #ffffff;
    padding: 30px 40px;
    border-radius: 12px;
    box-shadow: 0 8px 30px rgba(0,0,0,0.1);
    width: 100%;
    max-width: 600px;
}
h1 {
    text-align: center;
    color: #1a1a1a;
    margin-bottom: 10px;
}
p {
    text-align: center;
    color: #666;
    margin-bottom: 30px;
}
.form-group {
    margin-bottom: 25px;
}
label {
    display: block;
    margin-bottom: 8px;
    font-weight: 600;
    color: #555;
}
input[type="text"], input[type="file"] {
    width: 100%;
    padding: 12px;
    border: 1px solid #ccc;
    border-radius: 6px;
    box-sizing: border-box;
    font-size: 16px;
}
input[type="file"] {
    padding: 10px;
}
button {
    width: 100%;
    padding: 15px;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 6px;
    font-size: 18px;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.3s ease;
}
button:hover {
    background-color: #0056b3;
}
button:disabled {
    background-color: #aaa;
    cursor: not-allowed;
}
#status {
    text-align: center;
    margin-top: 25px;
    font-size: 16px;
    font-weight: 500;
    color: #333;
    min-height: 25px;
}
.loader {
    border: 4px solid #f3f3f3;
    border-top: 4px solid #007bff;
    border-radius: 50%;
    width: 25px;
    height: 25px;
    animation: spin 1s linear infinite;
    margin: 20px auto;
}
@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}
#result {
    margin-top: 30px;
    text-align: center;
}
#result video {
    max-width: 100%;
    border-radius: 8px;
    border: 1px solid #ddd;
}
#result a {
    display: inline-block;
    margin-top: 15px;
    padding: 10px 20px;
    background-color: #28a745;
    color: white;
    text-decoration: none;
    border-radius: 6px;
    font-weight: 500;
}
#result a:hover {
    background-color: #218838;
}
</style>
