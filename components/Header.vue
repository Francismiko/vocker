<script setup lang="ts">
import { Search, Setting } from '@element-plus/icons-vue'

const activeIndex = ref<string>('')
const router = useRouter()
</script>

<template>
  <el-menu :default-active="activeIndex" class="el-menu-demo" mode="horizontal" :ellipsis="false">
    <el-menu-item @click="router.push('/')">
      <NuxtImg src="/docker-full.png" sizes="48" />
    </el-menu-item>

    <div class="flex-grow" />

    <div class="flex items-center w-108">
      <el-input class="mr-4" placeholder="搜索..." :prefix-icon="Search" />
    </div>

    <el-menu-item index="setting">
      <el-icon>
        <Setting />
      </el-icon>
    </el-menu-item>
  </el-menu>
</template>
