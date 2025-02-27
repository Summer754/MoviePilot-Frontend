<script lang="ts" setup>
import { useToast } from 'vue-toast-notification'
import api from '@/api'
import type { Plugin } from '@/api/types'
import noImage from '@images/logos/plugin.png'
import { getDominantColor } from '@/@core/utils/image'

// 输入参数
const props = defineProps({
  plugin: Object as PropType<Plugin>,
  width: String,
  height: String,
})

// 定义触发的自定义事件
const emit = defineEmits(['install'])

// 背景颜色
const backgroundColor = ref('#28A9E1')

// 图片对象
const imageRef = ref<any>()

// 提示框
const $toast = useToast()

// 进度框
const progressDialog = ref(false)

// 进度框文本
const progressText = ref('正在安装插件...')

// 图片是否加载完成
const isImageLoaded = ref(false)

// 图片是否加载失败
const imageLoadError = ref(false)

// 图片加载完成
async function imageLoaded() {
  isImageLoaded.value = true
  const imageElement = imageRef.value?.$el.querySelector('img') as HTMLImageElement
  // 从图片中提取背景色
  backgroundColor.value = await getDominantColor(imageElement)
}

// 安装插件
async function installPlugin() {
  try {
    // 显示等待提示框
    progressDialog.value = true
    progressText.value = `正在安装 ${props.plugin?.plugin_name} ${props?.plugin?.plugin_version} 插件...`

    const result: { [key: string]: any } = await api.get(
      `plugin/install/${props.plugin?.id}`,
      {
        params: {
          repo_url: props.plugin?.repo_url,
          force: props.plugin?.has_update,
        },
      },
    )

    // 隐藏等待提示框
    progressDialog.value = false

    if (result.success) {
      $toast.success(`插件 ${props.plugin?.plugin_name} 安装成功！`)

      // 通知父组件刷新
      emit('install')
    }
    else {
      $toast.error(`插件 ${props.plugin?.plugin_name} 安装失败：${result.message}`)
    }
  }
  catch (error) {
    console.error(error)
  }
}

// 计算图标路径
const iconPath: Ref<string> = computed(() => {
  if (imageLoadError.value)
    return noImage
  // 如果是网络图片则使用代理后返回
  if (props.plugin?.plugin_icon?.startsWith('http'))
    return `${import.meta.env.VITE_API_BASE_URL}system/img/${encodeURIComponent(props.plugin?.plugin_icon)}/1`

  return `./plugin_icon/${props.plugin?.plugin_icon}`
})

// 访问插件页面
function visitPluginPage() {
  // 将raw.githubusercontent.com转换为项目地址
  let repoUrl = props.plugin?.repo_url
  if (repoUrl) {
    if (repoUrl.includes('raw.githubusercontent.com')) {
      if (!repoUrl.endsWith('/'))
        repoUrl += '/'

      if (repoUrl.split('/').length < 6)
        repoUrl = `${repoUrl}main/`

      try {
        const [user, repo] = repoUrl.split('/').slice(-4, -2)
        repoUrl = `https://github.com/${user}/${repo}`
      }
      catch (error) {
        return
      }
    }
  }
  else {
    repoUrl = props.plugin?.author_url
  }
  window.open(repoUrl, '_blank')
}

// 弹出菜单
const dropdownItems = ref([
  {
    title: '查看详情',
    value: 1,
    props: {
      prependIcon: 'mdi-information-outline',
      click: visitPluginPage,
    },
  },
])
</script>

<template>
  <VCard
    :width="props.width"
    :height="props.height"
    @click="installPlugin"
  >
    <div
      class="relative pa-4 text-center card-cover-blurred"
      :style="{ background: `${backgroundColor}` }"
    >
      <div class="me-n3 absolute top-0 right-3">
        <IconBtn>
          <VIcon icon="mdi-dots-vertical" class="text-white" />
          <VMenu
            activator="parent"
            close-on-content-click
          >
            <VList>
              <VListItem
                v-for="(item, i) in dropdownItems"
                :key="i"
                variant="plain"
                @click="item.props.click"
              >
                <template #prepend>
                  <VIcon :icon="item.props.prependIcon" />
                </template>
                <VListItemTitle v-text="item.title" />
              </VListItem>
            </VList>
          </VMenu>
        </IconBtn>
      </div>
      <div
        v-if="props.plugin?.has_update"
        class="me-n3 absolute top-0 left-1"
      >
        <VIcon
          icon="mdi-new-box"
          class="text-white"
        />
      </div>
      <VAvatar
        size="8rem"
      >
        <VImg
          ref="imageRef"
          :src="iconPath"
          aspect-ratio="4/3"
          cover
          :class="{ shadow: isImageLoaded }"
          @load="imageLoaded"
          @error="imageLoadError = true"
        />
      </VAvatar>
    </div>
    <VCardTitle>{{ props.plugin?.plugin_name }}</VCardTitle>

    <VCardText>
      {{ props.plugin?.plugin_desc }}
    </VCardText>
    <VCardText>
      作者：<a
        :href="props.plugin?.author_url"
        target="_blank"
        @click.stop
      >
        {{ props.plugin?.plugin_author }}
      </a><br>
      版本：{{ props.plugin?.plugin_version }}
    </VCardText>
  </VCard>
  <!-- 安装插件进度框 -->
  <VDialog
    v-model="progressDialog"
    :scrim="false"
    width="25rem"
  >
    <VCard
      color="primary"
    >
      <VCardText class="text-center">
        {{ progressText }}
        <VProgressLinear
          indeterminate
          color="white"
          class="mb-0 mt-1"
        />
      </VCardText>
    </VCard>
  </VDialog>
</template>

<style lang="scss" scoped>
.card-cover-blurred::before {
  position: absolute;
  /* stylelint-disable-next-line property-no-vendor-prefix */
  -webkit-backdrop-filter: blur(2px);
  backdrop-filter: blur(2px);
  background: rgba(29, 39, 59, 48%);
  content: "";
  inset: 0;
}
</style>
