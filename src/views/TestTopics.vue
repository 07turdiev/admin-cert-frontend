<template>
  <div class="test-topics-container">
    <Toast />

    <div class="page-header">
      <div>
        <h2 class="page-title">Test mavzulari</h2>
        <p class="page-subtitle">Choraklarga bog‘langan test mavzularini boshqaring.</p>
      </div>
      <Button label="Mavzu qo‘shish" icon="pi pi-plus" class="p-button-success" @click="openAdd" />
    </div>

    <DataTable :value="topics" :loading="loading" class="p-datatable-sm" responsiveLayout="scroll"
      paginator :rows="10" :rowsPerPageOptions="[10, 25, 50]">
      <template #empty>
        <div class="table-empty">Hali mavzu qo‘shilmagan.</div>
      </template>

      <Column field="test_topic_id" header="ID" style="width: 70px" />
      <Column field="topic_name" header="Mavzu nomi" />
      <Column header="Chorak" style="min-width: 200px">
        <template #body="{ data }">
          <span v-if="data.quarter" class="quarter-pill">
            {{ data.quarter.name }} — {{ formatDate(data.quarter.date) }}
          </span>
          <span v-else class="muted">—</span>
        </template>
      </Column>
      <Column header="Amallar" style="width: 130px">
        <template #body="{ data }">
          <Button icon="pi pi-pencil" class="p-button-rounded p-button-text p-button-info mr-1"
            @click="openEdit(data)" v-tooltip.top="'Tahrirlash'" />
          <Button icon="pi pi-trash" class="p-button-rounded p-button-text p-button-danger"
            @click="confirmDelete(data)" v-tooltip.top="'O‘chirish'" />
        </template>
      </Column>
    </DataTable>

    <!-- Add / Edit Dialog -->
    <Dialog v-model:visible="showDialog" :header="dialogMode === 'add' ? 'Mavzu qo‘shish' : 'Mavzuni tahrirlash'"
      :modal="true" :closable="true" :style="{ width: '420px' }">
      <div class="form">
        <div class="field">
          <label class="field-label">Mavzu nomi <span class="req">*</span></label>
          <InputText v-model="form.topic_name" placeholder="Mavzu nomi" class="w-full"
            :class="{ 'p-invalid': submitted && !form.topic_name }" />
          <small v-if="submitted && !form.topic_name" class="p-error">Mavzu nomini kiriting.</small>
        </div>
        <div class="field">
          <label class="field-label">Chorak</label>
          <Dropdown v-model="form.quarter_id" :options="quarters" optionLabel="label" optionValue="value"
            placeholder="Chorakni tanlang" class="w-full" showClear filter />
        </div>
      </div>
      <template #footer>
        <Button label="Bekor qilish" icon="pi pi-times" class="p-button-text" @click="showDialog = false" />
        <Button :label="dialogMode === 'add' ? 'Qo‘shish' : 'Saqlash'" icon="pi pi-check" class="p-button-success"
          :loading="saving" @click="save" />
      </template>
    </Dialog>

    <!-- Delete Confirm -->
    <Dialog v-model:visible="showDeleteDialog" header="O‘chirishni tasdiqlang" :modal="true" :closable="false"
      :style="{ width: '400px' }">
      <span>Ushbu mavzuni o‘chirishni istaysizmi? Unga bog‘langan savollar ham yo‘qolishi mumkin.</span>
      <template #footer>
        <Button label="Bekor qilish" icon="pi pi-times" class="p-button-text" @click="showDeleteDialog = false" />
        <Button label="O‘chirish" icon="pi pi-trash" class="p-button-danger" :loading="deleting" @click="deleteTopic" />
      </template>
    </Dialog>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import { useStore } from 'vuex'
import { useToast } from 'primevue/usetoast'

const emptyForm = () => ({ test_topic_id: null, topic_name: '', quarter_id: null })

export default {
  name: 'TestTopics',
  setup() {
    const store = useStore()
    const toast = useToast()

    const topics = ref([])
    const quarters = ref([])
    const loading = ref(false)

    const showDialog = ref(false)
    const dialogMode = ref('add')
    const form = ref(emptyForm())
    const submitted = ref(false)
    const saving = ref(false)

    const showDeleteDialog = ref(false)
    const deleting = ref(false)
    const deleteId = ref(null)

    const formatDate = (iso) => {
      if (!iso) return ''
      const d = new Date(iso)
      const pad = n => String(n).padStart(2, '0')
      return `${pad(d.getDate())}.${pad(d.getMonth() + 1)}.${d.getFullYear()}`
    }

    // --- Loading --------------------------------------------------------
    const fetchTopics = async () => {
      loading.value = true
      topics.value = await store.dispatch('fetchTestTopics')
      loading.value = false
    }
    const fetchQuarters = async () => {
      try {
        const result = await store.dispatch('fetchQuarters', { offset: 0, limit: 100 })
        const list = (result && result.data) ? result.data : []
        quarters.value = list.map(q => ({ label: q.name + ' — ' + formatDate(q.date), value: q.quarter_id }))
      } catch (e) {
        quarters.value = []
      }
    }

    // --- Add / Edit -----------------------------------------------------
    const openAdd = () => {
      form.value = emptyForm()
      dialogMode.value = 'add'
      submitted.value = false
      showDialog.value = true
    }
    const openEdit = (topic) => {
      form.value = { test_topic_id: topic.test_topic_id, topic_name: topic.topic_name, quarter_id: topic.quarter_id || null }
      dialogMode.value = 'edit'
      submitted.value = false
      showDialog.value = true
    }
    const save = async () => {
      submitted.value = true
      if (!form.value.topic_name.trim()) {
        toast.add({ severity: 'warn', summary: 'To‘ldirilmagan', detail: 'Mavzu nomini kiriting.', life: 3000 })
        return
      }
      saving.value = true
      let res
      if (dialogMode.value === 'add') {
        res = await store.dispatch('addTestTopic', form.value)
      } else {
        res = await store.dispatch('editTestTopic', { id: form.value.test_topic_id, topicData: form.value })
      }
      saving.value = false

      if (res) {
        toast.add({ severity: 'success', summary: 'Saqlandi', detail: dialogMode.value === 'add' ? 'Mavzu qo‘shildi.' : 'Mavzu yangilandi.', life: 2500 })
        showDialog.value = false
        await fetchTopics()
      } else {
        toast.add({ severity: 'error', summary: 'Xatolik', detail: 'Saqlashda xatolik yuz berdi.', life: 4000 })
      }
    }

    // --- Delete ---------------------------------------------------------
    const confirmDelete = (topic) => {
      deleteId.value = topic.test_topic_id
      showDeleteDialog.value = true
    }
    const deleteTopic = async () => {
      deleting.value = true
      const res = await store.dispatch('deleteTestTopic', deleteId.value)
      deleting.value = false
      showDeleteDialog.value = false
      if (res) {
        toast.add({ severity: 'success', summary: 'O‘chirildi', detail: 'Mavzu o‘chirildi.', life: 2500 })
        await fetchTopics()
      } else {
        toast.add({ severity: 'error', summary: 'Xatolik', detail: 'O‘chirishda xatolik yuz berdi.', life: 4000 })
      }
    }

    onMounted(() => {
      fetchTopics()
      fetchQuarters()
    })

    return {
      topics, quarters, loading,
      showDialog, dialogMode, form, submitted, saving,
      showDeleteDialog, deleting,
      formatDate, openAdd, openEdit, save, confirmDelete, deleteTopic,
    }
  },
}
</script>

<style scoped>
.test-topics-container {
  padding: 2rem;
}

.page-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}
.page-title {
  margin: 0;
  font-size: 1.6rem;
  font-weight: 700;
  color: #1e293b;
}
.page-subtitle {
  margin: 4px 0 0;
  color: #64748b;
  font-size: 0.9rem;
}

.table-empty {
  text-align: center;
  color: #64748b;
  padding: 16px;
}

.quarter-pill {
  display: inline-block;
  background: #eef2ff;
  color: #4338ca;
  border-radius: 999px;
  padding: 3px 12px;
  font-size: 0.85rem;
  font-weight: 600;
}
.muted {
  color: #94a3b8;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.field-label {
  font-weight: 600;
  font-size: 0.9rem;
  color: #334155;
}
.req {
  color: #e53935;
}
.w-full {
  width: 100%;
}
.mr-1 {
  margin-right: 4px;
}
</style>
