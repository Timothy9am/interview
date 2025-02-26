<template>
  <q-page class="row q-pt-xl">
    <div class="full-width q-px-xl">
      <div class="q-mb-xl">
        <q-input
          v-model="tempData.name"
          label="姓名"
          lazy-rules
          :rules="[(val) => !!val || '欄位不得空白']"
        />
        <q-input
          v-model="tempData.age"
          label="年齡"
          type="number"
          lazy-rules
          :rules="[
            (val) => !!val || '欄位不得空白',
            (val) => /^\d+$/.test(val) || '僅限輸入正整數',
          ]"
        />
        <q-btn color="primary" class="q-mt-md" @click="handleSubmit">
          {{ isEditing ? '更新' : '新增' }}
        </q-btn>
      </div>

      <q-table
        flat
        bordered
        ref="tableRef"
        :rows="blockData"
        :columns="(tableConfig as QTableProps['columns'])"
        row-key="id"
        hide-pagination
        separator="cell"
        :rows-per-page-options="[0]"
      >
        <template v-slot:header="props">
          <q-tr :props="props">
            <q-th v-for="col in props.cols" :key="col.name" :props="props">
              {{ col.label }}
            </q-th>
            <q-th></q-th>
          </q-tr>
        </template>

        <template v-slot:body="props">
          <q-tr :props="props">
            <q-td
              v-for="col in props.cols"
              :key="col.name"
              :props="props"
              style="min-width: 120px"
            >
              <div>{{ col.value }}</div>
            </q-td>
            <q-td class="text-right" auto-width v-if="tableButtons.length > 0">
              <q-btn
                @click="handleClickOption(btn, props.row)"
                v-for="(btn, index) in tableButtons"
                :key="index"
                size="sm"
                color="grey-6"
                round
                dense
                :icon="btn.icon"
                class="q-ml-md"
                padding="5px 5px"
              >
                <q-tooltip
                  transition-show="scale"
                  transition-hide="scale"
                  anchor="top middle"
                  self="bottom middle"
                  :offset="[10, 10]"
                >
                  {{ btn.label }}
                </q-tooltip>
              </q-btn>
            </q-td>
          </q-tr>
        </template>
        <template v-slot:no-data="{ icon }">
          <div
            class="full-width row flex-center items-center text-primary q-gutter-sm"
            style="font-size: 18px"
          >
            <q-icon size="2em" :name="icon" />
            <span> 無相關資料 </span>
          </div>
        </template>
      </q-table>
    </div>
    <q-dialog v-model="confirmDialogVisible">
      <q-card>
        <q-card-section class="row items-center">
          <q-icon name="warning" color="warning" />
          <span class="q-ml-sm">確定要刪除這筆資料嗎？</span>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="取消" color="primary" v-close-popup />
          <q-btn flat label="確定" color="primary" @click="confirmDelete" />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup lang="ts">
import axios from 'axios';
import { QTableProps } from 'quasar';
import { ref, onMounted } from 'vue';
import { nanoid } from 'nanoid';

interface btnType {
  label: string;
  icon: string;
  status: string;
}
interface DataType {
  id: string;
  name: string;
  age: number;
}

const blockData = ref<DataType[]>([]);
const tableConfig = ref([
  {
    label: '姓名',
    name: 'name',
    field: 'name',
    align: 'left',
  },
  {
    label: '年齡',
    name: 'age',
    field: 'age',
    align: 'left',
  },
]);
const tableButtons = ref([
  {
    label: '編輯',
    icon: 'edit',
    status: 'edit',
  },
  {
    label: '刪除',
    icon: 'delete',
    status: 'delete',
  },
]);
const tempData = ref({
  id: '',
  name: '',
  age: '',
});
const isEditing = ref(false);
const confirmDialogVisible = ref(false);
const deleteData = ref<DataType | null>(null);

onMounted(async () => {
  await fetchData();
});

async function fetchData() {
  try {
    const response = await axios.get(
      'https://dahua.metcfire.com.tw/api/CRUDTest/a'
    );
    blockData.value = response.data;
  } catch (error) {
    console.error('Error fetching data:', error);
  }
}

async function handleSubmit() {
  if (tempData.value.name && tempData.value.age) {
    try {
      if (isEditing.value) {
        const response = await axios.patch(
          `https://dahua.metcfire.com.tw/api/CRUDTest`,
          {
            id: tempData.value.id,
            name: tempData.value.name,
            age: tempData.value.age,
          }
        );
        const index = blockData.value.findIndex(
          (item) => item.id === tempData.value.id
        );
        if (index !== -1) {
          blockData.value[index] = response.data;
          fetchData();
        }
      } else {
        const response = await axios.post(
          'https://dahua.metcfire.com.tw/api/CRUDTest',
          {
            id: nanoid(),
            name: tempData.value.name,
            age: tempData.value.age,
          }
        );
        blockData.value.push(response.data);
        fetchData();
      }
      resetForm();
    } catch (error) {
      console.error('Error submitting data:', error);
    }
  }
}

function resetForm() {
  tempData.value.id = '';
  tempData.value.name = '';
  tempData.value.age = '';
  isEditing.value = false;
}

async function handleClickOption(btn: btnType, data: DataType) {
  if (btn.status === 'edit') {
    tempData.value.id = data.id;
    tempData.value.name = data.name;
    tempData.value.age = data.age.toString();
    isEditing.value = true;
  } else if (btn.status === 'delete') {
    deleteData.value = data;
    confirmDialogVisible.value = true;
  }
}

async function confirmDelete() {
  if (deleteData.value) {
    try {
      await axios.delete(
        `https://dahua.metcfire.com.tw/api/CRUDTest/${deleteData.value.id}`
      );
      const index = blockData.value.findIndex(
        (item) => deleteData.value && item.id === deleteData.value.id
      );
      if (index !== -1) {
        blockData.value.splice(index, 1);
      }
      deleteData.value = null;
      confirmDialogVisible.value = false;
    } catch (error) {
      console.error('Error deleting data:', error);
    }
  }
}
</script>

<style lang="scss" scoped>
.q-table th {
  font-size: 20px;
  font-weight: bold;
}

.q-table tbody td {
  font-size: 18px;
}
</style>
