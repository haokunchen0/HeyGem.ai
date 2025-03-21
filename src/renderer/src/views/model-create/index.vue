<template>
  <div class="model-create-page">
    <!-- 上传区域 -->
    <div class="upload-section">
      <div 
        class="upload-area" 
        @click="handleUploadClick"
        @dragover.prevent="handleDragOver"
        @dragleave.prevent="handleDragLeave"
        @drop.prevent="handleDrop"
        :class="{ 'drag-over': isDragOver }"
      >
        <div class="upload-content">
          <div class="upload-icon">
            <img src="../../assets/images/create-model/image-shot.png" alt="Upload" />
          </div>
          <p class="upload-text">{{ $t('common.modelCreateView.tipsText') || '点击上传拍摄好的原始视频' }}</p>
          <p class="upload-text-sub">{{ $t('common.modelCreateBatch.dragTips') || '也可以拖拽多个视频文件到此处' }}</p>
          <input
            type="file"
            ref="fileInput"
            multiple
            accept="video/*"
            style="display: none;"
            @change="handleFileChange"
          />
        </div>
      </div>
    </div>

    <!-- 模型列表区域 -->
    <div class="model-list-section">
      <h2 class="section-title">{{ $t('common.tab.myAvatarsText') || '我的数字形象' }}</h2>
      <div v-if="home.homeState.modelNum === 0" class="empty">
        <div class="empty-box">
          <img src="../../assets/images/home/myModelList.svg" />
          <div class="empty-text">{{ $t('common.myModelList.emptyText') || '暂无数字形象' }}</div>
          <div class="empty-text">
            {{ $t('common.myModelList.emptyRightText') || '创建您的第一个数字形象吧！' }}
          </div>
        </div>
      </div>
      <div class="table-list" v-else>
        <div v-for="(item, index) in state.modelList" :key="index" class="li">
          <!-- 视频上部分内容 -->
          <div class="img-video comme">
            <div class="img-video-content">
              <video class="model-video" :src="item.video_path"></video>
            </div>
          </div>
          <!-- 下载和预览 -->
          <div class="download-preview comme">
            <div class="download-preview-content">
              <div class="download-button" @click="editVideo(item)">
                <img src="../../assets/images/home/video.svg" />
                <span>{{ $t('common.myModelList.createVideoText') || '制作视频' }}</span>
              </div>
              <div class="delete-video" @click="delModel(item.id)">
                <t-icon name="delete" style="color: #fff; font-size: 12px" />
              </div>
              <div class="preview-box" @click="previewVideo(item.video_path)">
                <img src="../../assets/images/home/play.svg" />
                <span>{{ $t('common.myModelList.previewText') || '预览' }}</span>
              </div>
            </div>
          </div>
          <!-- 视频下部分内容 -->
          <div class="bottom-text">
            <div class="top">
              <div class="h1">{{ item.name }}</div>
            </div>
            <div class="text">{{ formatDate(item.created_at) }}</div>
          </div>
        </div>
      </div>
      
      <!-- 分页 -->
      <div v-if="home.homeState.modelNum > 0" class="pagination-box">
        <div class="pagination-content">
          <t-config-provider :global-config="locale === 'zh' ? globalZh : globalEn">
            <t-pagination
              v-model="state.current"
              v-model:pageSize="state.pageSize"
              :total="state.total"
              show-jumper
              class="pagination"
              @page-size-change="onPageSizeChange"
              @current-change="onCurrentChange"
            />
          </t-config-provider>
        </div>
      </div>
    </div>
    
    <!-- 模态框 -->
    <VideoDialog
      :showVideoDialog="state.showVideoDialog"
      :videoUrl="state.videoUrl"
      @cancel="cancelFun"
    />
    <DeleteDialog ref="deleteDialogRef" @ok="okDelete" />
  </div>
</template>

<script setup>
import { reactive, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import { formatDate } from '@renderer/utils/index.js'
import { modelPage, removeModel, addModel } from '@renderer/api/index.js'
import { useHomeStore } from '@renderer/stores/home.js'
import { MessagePlugin } from 'tdesign-vue-next'
import VideoDialog from '@renderer/views/home/components/videoDialog.vue'
import DeleteDialog from '@renderer/components/deleteDialog.vue'
import { Client } from '@renderer/client'
import { useI18n } from 'vue-i18n'
import enConfig from 'tdesign-vue-next/es/locale/en_US'
import zhConfig from 'tdesign-vue-next/es/locale/zh_CN'
import merge from 'lodash/merge'

const { t, locale } = useI18n()
const router = useRouter()
const home = useHomeStore()
const deleteDialogRef = ref(null)
const fileInput = ref(null)

const globalEn = merge(enConfig, {
  pagination: {}
})
const globalZh = merge(zhConfig, {
  pagination: {}
})

const state = reactive({
  current: 1,
  pageSize: 10,
  total: 0,
  modelList: [],
  showVideoDialog: false,
  videoUrl: '',
  isDragOver: false,
  delModelId: '',
})

// 生命周期
onMounted(() => {
  // 确保使用10条/页
  state.pageSize = 10
  modelPageAJax()
})

// 方法
const modelPageAJax = async () => {
  try {
    const result = await modelPage({
      page: state.current,
      pageSize: state.pageSize
    })
    state.modelList = result.list || []
    state.total = result.total || 0
    home.setModelNum(result.total || 0)
  } catch (error) {
    console.error('加载数字人列表失败', error)
    state.modelList = []
    state.total = 0
  }
}

const onPageSizeChange = () => {
  modelPageAJax()
}

const onCurrentChange = () => {
  modelPageAJax()
}

const previewVideo = (videoUrl) => {
  state.videoUrl = videoUrl
  state.showVideoDialog = true
}

const cancelFun = () => {
  state.showVideoDialog = false
}

const editVideo = (model) => {
  router.push({
    path: '/video/edit',
    query: {
      modelId: model.id
    }
  })
}

const delModel = (id) => {
  state.delModelId = id
  deleteDialogRef.value.show()
}

const okDelete = async () => {
  try {
    await removeModel(state.delModelId)
    MessagePlugin.success(t('common.message.deleteSuccess') || '删除成功')
    modelPageAJax()
  } catch (error) {
    console.error('删除失败', error)
    MessagePlugin.error(t('common.message.deleteFailed') || '删除失败')
  }
}

// 文件上传相关方法
const handleUploadClick = () => {
  fileInput.value.click()
}

const handleDragOver = (e) => {
  state.isDragOver = true
}

const handleDragLeave = (e) => {
  state.isDragOver = false
}

const handleDrop = (e) => {
  state.isDragOver = false
  const files = e.dataTransfer.files
  if (files.length > 0) {
    processFiles(files)
  }
}

const handleFileChange = (e) => {
  const files = e.target.files
  if (files.length > 0) {
    processFiles(files)
    // 重置input，使同一文件可以再次选择
    e.target.value = ''
  }
}

const processFiles = async (files) => {
  const videoFiles = Array.from(files).filter(file => file.type.startsWith('video/'))
  
  if (videoFiles.length === 0) {
    MessagePlugin.error(t('common.message.noVideoSelected') || '请选择视频文件')
    return
  }
  
  // 显示上传进度提示
  let loadingInstance = MessagePlugin.loading({
    content: `正在处理 ${videoFiles.length} 个视频文件...`,
    duration: 0,
  })
  
  try {
    // 处理每个文件并显示进度
    for (let i = 0; i < videoFiles.length; i++) {
      const file = videoFiles[i];
      const fileName = file.name.split('.')[0] // 使用文件名（不含扩展名）作为模型名
      
      // 关闭上一个加载提示并显示新的进度提示
      loadingInstance.close();
      loadingInstance = MessagePlugin.loading({
        content: `正在处理 ${i+1}/${videoFiles.length}: ${fileName}`,
        duration: 0,
      });
      
      // 检查文件是否已经有本地路径
      let filePath = file.path
      if (!filePath) {
        // 关闭上一个加载提示并显示文件保存进度
        loadingInstance.close();
        loadingInstance = MessagePlugin.loading({
          content: `正在保存文件 ${i+1}/${videoFiles.length}: ${fileName}`,
          duration: 0,
        });
        
        filePath = await saveFileLocally(file, file.name)
      }
      
      if (filePath) {
        // 关闭上一个加载提示并显示视频检查进度
        loadingInstance.close();
        loadingInstance = MessagePlugin.loading({
          content: `正在检查视频 ${i+1}/${videoFiles.length}: ${fileName}`,
          duration: 0,
        });
        
        const videoInfo = await Client.file.getVideoInfo(filePath)
        
        if (videoInfo.isOK && videoInfo.duration >= 8) {
          // 关闭上一个加载提示并显示添加模型进度
          loadingInstance.close();
          loadingInstance = MessagePlugin.loading({
            content: `正在添加模型 ${i+1}/${videoFiles.length}: ${fileName}`,
            duration: 0,
          });
          
          await addModel({
            name: fileName,
            videoPath: filePath
          })
          MessagePlugin.success(`模型 ${fileName} 添加成功，正在训练...`)
        } else {
          console.error(`文件 ${fileName} 处理失败:`, videoInfo.msg || '视频长度需大于8秒')
          MessagePlugin.error(`${fileName}: ${videoInfo.msg || '视频长度需大于8秒'}`)
        }
      } else {
        MessagePlugin.error(`无法保存文件: ${fileName}`)
      }
    }
    
    // 刷新模型列表
    await modelPageAJax()
    MessagePlugin.success(`成功添加数字人模型，请在列表中查看`)
  } catch (error) {
    console.error('批量处理失败:', error)
    MessagePlugin.error('处理文件时出错: ' + error.message)
  } finally {
    if (loadingInstance) {
      loadingInstance.close()
    }
  }
}

// 如果需要，这个函数用于保存拖放的文件到本地
// 通常Electron会提供path属性，但以防万一
const saveFileLocally = async (file, fileName) => {
  try {
    // 将File对象转换为可传输的格式
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = async (event) => {
        try {
          // 获取ArrayBuffer
          const arrayBuffer = event.target.result;
          // 将ArrayBuffer传递给主进程保存
          const filePath = await Client.file.saveTemporaryFile(arrayBuffer, fileName);
          console.log('文件保存成功:', filePath);
          resolve(filePath);
        } catch (err) {
          console.error('处理文件数据失败:', err);
          reject(err);
        }
      };
      reader.onerror = (error) => {
        console.error('读取文件失败:', error);
        reject(error);
      };
      // 读取文件为ArrayBuffer
      reader.readAsArrayBuffer(file);
    });
  } catch (error) {
    console.error('保存文件失败:', error);
    return null;
  }
}
</script>

<style lang="less" scoped>
.model-create-page {
  padding: 20px 20px 0 20px;
  background-color: #f4f4f6;
  min-height: calc(100vh - 60px);
  overflow: auto;
  
  .upload-section {
    margin-bottom: 20px;
    
    .upload-area {
      background-color: #fff;
      border-radius: 8px;
      padding: 30px;
      cursor: pointer;
      transition: all 0.3s ease;
      border: 2px dashed #e7e7e9;
      
      &:hover, &.drag-over {
        border-color: #434AF9;
        background-color: rgba(67, 74, 249, 0.05);
      }
      
      .upload-content {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        
        .upload-icon {
          margin-bottom: 15px;
          
          img {
            width: 64px;
            height: 64px;
          }
        }
        
        .upload-text {
          font-size: 16px;
          color: #333;
          margin-bottom: 5px;
        }
        
        .upload-text-sub {
          font-size: 14px;
          color: #999;
        }
      }
    }
  }
  
  .model-list-section {
    background-color: #fff;
    border-radius: 8px;
    padding: 20px;
    
    .section-title {
      font-size: 18px;
      margin-bottom: 20px;
      color: #333;
      font-weight: 500;
    }
    
    .empty {
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 150px;
      .empty-box {
        img {
          width: 160px;
          margin: 0 auto;
          display: block;
        }
        .empty-text {
          font-family: PingFang SC, PingFang SC;
          font-weight: 400;
          font-size: 12px;
          text-align: center;
          color: #999999;
          line-height: 16px;
        }
      }
    }
    
    .table-list {
      display: grid;
      grid-template-columns: repeat(4, 1fr);
      gap: 20px;
      padding-bottom: 10px;
      color: #ccc;
      
      .li:hover {
        transform: scale(1.01);
        box-shadow: 0px 0px 12px rgba(0, 0, 0, 0.12);
        .download-preview {
          z-index: 2;
          background: rgba(0, 0, 0, 0.6);
          display: flex;
          justify-content: center;
          align-items: center;
          position: relative;
          .download-preview-content {
            .download-button {
              padding: 0 5px;
              height: 30px;
              cursor: pointer;
              background: #434af9;
              border-radius: 4px;
              display: flex;
              align-items: center;
              justify-content: center;
              border: 1px solid #434af9;
              font-family: PingFang SC, PingFang SC;
              font-weight: 500;
              font-size: 12px;
              color: #ffffff;
              line-height: 18px;

              img {
                margin-right: 4px;
              }
            }
            .preview-box {
              padding: 0 5px;
              height: 18px;
              background: rgba(0, 0, 0, 0.6);
              border-radius: 100px;
              position: absolute;
              right: 6px;
              display: flex;
              align-items: center;
              justify-content: center;
              cursor: pointer;
              bottom: 6px;
              z-index: 1;
              font-family: HarmonyOS Sans SC, HarmonyOS Sans SC;
              font-weight: 400;
              font-size: 10px;
              color: #ffffff;
              line-height: 12px;
              
              img {
                margin-right: 4px;
              }
            }
            .delete-video {
              width: 20px;
              height: 20px;
              background: rgba(255, 255, 255, 0.2);
              border-radius: 6px;
              position: absolute;
              left: 10px;
              display: flex;
              align-items: center;
              justify-content: center;
              cursor: pointer;
              top: 10px;
              z-index: 1;
            }
          }
        }
      }

      .li {
        transition: all 0.3s ease;
        aspect-ratio: 1 / 0.83;
        border-radius: 8px;
        position: relative;
        cursor: pointer;
        border: 1px solid #f2f2f4;

        .download-preview {
          display: none;
        }
        .comme {
          position: absolute;
          top: 0;
          width: 100%;
          left: 0;
          border-radius: 8px 8px 0 0;
          height: calc(100% - 65px);
        }
        .img-video {
          z-index: 1;
          background: linear-gradient(180deg, #b8c2ce 0%, #e2e6f0 100%);
          .img-video-content {
            position: relative;
            height: 100%;
            .model-video {
              width: 100%;
              height: 100%;
              object-fit: contain;
            }
          }
        }
        .bottom-text {
          position: absolute;
          bottom: 0;
          padding: 4px 8px 8px 8px;
          left: 0;
          width: 100%;
          height: 65px;
          background: #ffffff;
          .top {
            display: flex;
            margin-top: 5px;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
            margin-bottom: 4px;
            .h1 {
              font-family: HarmonyOS Sans SC, HarmonyOS Sans SC;
              font-weight: 600;
              font-size: 14px;
              color: #252525;
              line-height: 23px;
              margin-left: 4px;
            }
          }
          .text {
            font-family: HarmonyOS Sans SC, HarmonyOS Sans SC;
            font-weight: 400;
            font-size: 12px;
            color: rgba(37, 37, 37, 0.5);
            white-space: nowrap;
            overflow: hidden;
            margin-top: 6px;
            text-overflow: ellipsis;
          }
        }
      }
    }
    
    .pagination-box {
      position: relative;
      z-index: 99;
      width: 100%;
      padding-top: 10px;
      background-color: #fff;
      .pagination-content {
        justify-content: center;
        display: flex;
        height: 40px;
      }
    }
  }
}
</style> 