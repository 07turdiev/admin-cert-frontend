<template>
  <div class="test-topics-container">
    <Button label="Add Topic" icon="pi pi-plus" class="p-button-success mb-3" @click="openAddDialog" />
    <DataTable :value="topics" :loading="loading" class="p-datatable-sm">
      <Column field="test_topic_id" header="ID" style="width: 80px" />
      <Column field="topic_name" header="Topic Name" />
      <Column header="Actions" style="width: 160px">
        <template #body="{ data }">
          <Button icon="pi pi-pencil" class="p-button-rounded p-button-info mr-2" @click="openEditDialog(data)" />
          <Button icon="pi pi-trash" class="p-button-rounded p-button-danger" @click="confirmDelete(data)" />
        </template>
      </Column>
    </DataTable>
    <!-- Add Dialog -->
    <Dialog v-model:visible="showAddDialog" header="Add Topic" :modal="true" :closable="true" :style="{ width: '400px' }">
      <div class="p-fluid">
        <InputText v-model="newTopic.topic_name" placeholder="Topic Name" />
      </div>
      <template #footer>
        <Button label="Cancel" icon="pi pi-times" class="p-button-text" @click="showAddDialog = false" />
        <Button label="Add" icon="pi pi-check" class="p-button-success" @click="addTopic" :disabled="!newTopic.topic_name" />
      </template>
    </Dialog>
    <!-- Edit Dialog -->
    <Dialog v-model:visible="showEditDialog" header="Edit Topic" :modal="true" :closable="true" :style="{ width: '400px' }">
      <div class="p-fluid">
        <InputText v-model="editTopic.topic_name" placeholder="Topic Name" />
      </div>
      <template #footer>
        <Button label="Cancel" icon="pi pi-times" class="p-button-text" @click="showEditDialog = false" />
        <Button label="Save" icon="pi pi-check" class="p-button-success" @click="updateTopic" :disabled="!editTopic.topic_name" />
      </template>
    </Dialog>
    <!-- Delete Confirm Dialog -->
    <Dialog v-model:visible="showDeleteDialog" header="Confirm Delete" :modal="true" :closable="false" :style="{ width: '350px' }">
      <span>Are you sure you want to delete this topic?</span>
      <template #footer>
        <Button label="Cancel" icon="pi pi-times" class="p-button-text" @click="showDeleteDialog = false" />
        <Button label="Delete" icon="pi pi-trash" class="p-button-danger" @click="deleteTopic" />
      </template>
    </Dialog>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import { useStore } from 'vuex'

export default {
  name: 'TestTopics',
  setup() {
    const store = useStore()
    const topics = ref([])
    const loading = ref(false)
    const showAddDialog = ref(false)
    const showEditDialog = ref(false)
    const showDeleteDialog = ref(false)
    const newTopic = ref({ topic_name: '' })
    const editTopic = ref({})
    const deleteTopicId = ref(null)

    const fetchTopics = async () => {
      loading.value = true
      topics.value = await store.dispatch('fetchTestTopics')
      loading.value = false
    }

    const openAddDialog = () => {
      newTopic.value = { topic_name: '' }
      showAddDialog.value = true
    }
    const addTopic = async () => {
      await store.dispatch('addTestTopic', newTopic.value)
      showAddDialog.value = false
      fetchTopics()
    }
    const openEditDialog = (topic) => {
      editTopic.value = { ...topic }
      showEditDialog.value = true
    }
    const updateTopic = async () => {
      await store.dispatch('editTestTopic', { id: editTopic.value.test_topic_id, topicData: editTopic.value })
      showEditDialog.value = false
      fetchTopics()
    }
    const confirmDelete = (topic) => {
      deleteTopicId.value = topic.test_topic_id
      showDeleteDialog.value = true
    }
    const deleteTopic = async () => {
      await store.dispatch('deleteTestTopic', deleteTopicId.value)
      showDeleteDialog.value = false
      fetchTopics()
    }

    onMounted(fetchTopics)

    return {
      topics,
      loading,
      showAddDialog,
      showEditDialog,
      showDeleteDialog,
      newTopic,
      editTopic,
      openAddDialog,
      addTopic,
      openEditDialog,
      updateTopic,
      confirmDelete,
      deleteTopic
    }
  }
}
</script>

<style scoped>
.test-topics-container {
  padding: 2rem;
}
.mb-3 {
  margin-bottom: 1.5rem;
}
</style> 