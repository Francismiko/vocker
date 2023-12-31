<script setup lang="ts">
import type { ElTable } from 'element-plus';
import { Refresh, Delete, MoreFilled } from '@element-plus/icons-vue'
import type { DockerContainer } from '~/server/api/docker-container/query/list.get';
import type { Filters } from 'element-plus/es/components/table/src/table-column/defaults';
import type { DockerContainerMutations } from '~/server/api/docker-container/mutate/[mutate].post';

type ContainerState = "exited" | "running" | "paused";

const { data: dockerInfo, pending: dockerInfoPending, refresh: dockerInfoRefresh } = await useLazyFetch('/api/docker/query/info')
const { data: dockerContainerList, pending: dockerContainerListPending, refresh: dockerContainerListRefresh } = await useLazyFetch('/api/docker-container/query/list')

const search = ref<string>('')
const mutatingContainers = ref<string[]>([])
const multipleTableRef = ref<InstanceType<typeof ElTable>>()
const multipleSelection = ref<DockerContainer[]>([])

const stateType = (state: ContainerState) => {
  if (state === "exited") return "info"
  if (state === "running") return "success"
  if (state === "paused") return "warning"
};

const stateFilters: Filters = [
  { text: 'Running', value: 'running' },
  { text: 'Paused', value: 'paused' },
  { text: 'Exited', value: 'exited' }
]

const handleSelectionChange = (val: DockerContainer[]) => {
  multipleSelection.value = val
}

const handleRefresh = async () => {
  try {
    await dockerInfoRefresh();
    await dockerContainerListRefresh();
    ElMessage({
      message: '数据同步完成',
      type: 'success',
    });
  } catch (error) {
    ElMessage({
      message: '数据同步失败',
      type: 'error',
    });
  }
};


const handleContainerMutation = async (containerId: string, mutation: DockerContainerMutations) => {
  mutatingContainers.value.push(containerId);
  const res = await $fetch(`/api/docker-container/mutate/${mutation}`, {
    method: 'POST',
    body: {
      containerId,
    },
  });
  mutatingContainers.value.shift();

  if (res.success) {
    await dockerContainerListRefresh();
    ElMessage({
      message: '容器操作完成',
      type: 'success',
    })
  } else {
    ElMessage({
      message: '容器操作失败',
      type: 'error',
    })
  }
};

const filterState = (value: string, row: DockerContainer) => {
  return row.state === value
}
</script>

<template>
  <el-empty v-if="isEmpty(dockerInfo)" image="/warning.png" description="检测到Docker 引擎并未开启, 请启动相关服务."
    v-loading="dockerInfoPending">
    <el-button type="primary" :icon="Refresh" plain @click="handleRefresh"
      :loading="dockerContainerListPending || dockerInfoPending">
      刷新
    </el-button>
  </el-empty>
  <div v-else>
    <div class="flex w-full">
      <el-button type="primary" :icon="Refresh" plain @click="handleRefresh"
        :loading="dockerContainerListPending || dockerInfoPending">
        刷新
      </el-button>
      <el-button-group v-show="!isEmpty(multipleSelection)" class="ml-auto mr-4">
        <el-button type="primary">
          <Icon size="24px" name="material-symbols:play-arrow-rounded" />
        </el-button>
        <el-button type="primary">
          <Icon size="24px" name="material-symbols:pause" />
        </el-button>
        <el-button type="primary">
          <Icon size="24px" name="material-symbols:stop-rounded" />
        </el-button>
        <el-button type="danger">
          <Icon size="24px" name="material-symbols:delete" />
        </el-button>
      </el-button-group>
    </div>
    <el-tabs @tab-click="">
      <el-tab-pane label="所有容器" name="first" />
    </el-tabs>
    <el-table ref="multipleTableRef" :data="dockerContainerList!" height="509" style="width: 100%"
      @selection-change="handleSelectionChange" v-loading="dockerContainerListPending || dockerInfoPending">
      <el-table-column type="selection" width="55" />
      <el-table-column prop="name" label="名称" />
      <el-table-column prop="state" label="状态" :filters="stateFilters" :filter-method="filterState"
        filter-placement="right-start">
        <template #default="scope">
          <el-tag :type="stateType(scope.row.state)" v-loading="mutatingContainers.includes(scope.row.id)">
            {{ useStartCase(scope.row.state) }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="image" label="镜像源" />
      <el-table-column prop="platform" label="平台"
        :formatter="(row, column, cellValue, index) => useStartCase(cellValue)" />
      <el-table-column prop="started" label="上次运行" sortable
        :formatter="(row, column, cellValue, index) => $dayjs.unix(cellValue).fromNow()" />
      <el-table-column align="right">
        <template #header>
          <el-input v-model="search" size="default" placeholder="搜索容器..." />
        </template>
        <template #default="scope">
          <el-button v-if="scope.row.state === 'exited'" size="small" type="primary" plain circle
            @click="handleContainerMutation(scope.row.id, 'start')">
            <Icon size="16" name="material-symbols:play-arrow-rounded" />
          </el-button>
          <el-button v-else-if="scope.row.state === 'paused'" size="small" type="primary" plain circle
            @click="handleContainerMutation(scope.row.id, 'unpause')">
            <Icon size="16" name="material-symbols:play-arrow-rounded" />
          </el-button>
          <el-button v-else-if="scope.row.state === 'running'" size="small" type="primary" plain circle
            @click="handleContainerMutation(scope.row.id, 'stop')">
            <Icon size="16" name="material-symbols:stop-rounded" />
          </el-button>
          <el-dropdown trigger="click" class="ml-4 mr-8">
            <el-button size="small" type="primary" :icon="MoreFilled" plain circle />
            <template #dropdown>
              <el-dropdown-menu>
                <el-dropdown-item disabled>
                  <Icon size="24px" name="mingcute:terminal-box-fill" class="mr-2" />
                  命令行打开
                </el-dropdown-item>
                <el-dropdown-item disabled>
                  <Icon size="24px" name="mingcute:copy-2-fill" class="mr-2" />
                  创建克隆
                </el-dropdown-item>
                <el-dropdown-item :disabled="scope.row.state !== 'running'"
                  @click="handleContainerMutation(scope.row.id, 'pause')">
                  <Icon size="24px" name="mingcute:pause-fill" class="mr-2" />
                  暂停运行
                </el-dropdown-item>
                <el-dropdown-item>
                  <Icon size="24px" name="mingcute:refresh-anticlockwise-1-fill" class="mr-2" />
                  重新启动
                </el-dropdown-item>
                <el-dropdown-item disabled>
                  <Icon size="24px" name="mingcute:folder-open-fill" class="mr-2" />
                  查看文件
                </el-dropdown-item>
              </el-dropdown-menu>
            </template>
          </el-dropdown>
          <el-popconfirm title="确定要删除该容器吗?" confirm-button-type="danger" @confirm="() => {
            if (scope.row.state !== 'exited') return ElMessage({ message: '容器必须处于停止状态才能删除', type: 'error' })
            handleContainerMutation(scope.row.id, 'remove')
          }">
            <template #reference>
              <el-button size="small" type="danger" :icon="Delete" plain circle />
            </template>
          </el-popconfirm>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>