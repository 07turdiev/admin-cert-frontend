<template>
  <div class="test-questions-container">
    <Toast />

    <div class="page-header">
      <h2 class="page-title">Test savollari</h2>
      <p class="page-subtitle">Har bir mavzu bo‘yicha savollarni qo‘shing, tahrirlang yoki o‘chiring.</p>
    </div>

    <Accordion :multiple="true">
      <AccordionTab v-for="topic in topics" :key="topic.test_topic_id">
        <template #header>
          <div class="topic-header">
            <span class="topic-header-name">{{ topic.topic_name }}</span>
            <span class="topic-header-count">{{ (questionsByTopic[topic.test_topic_id] || []).length }} ta savol</span>
          </div>
        </template>

        <div class="topic-toolbar">
          <Button label="Savol qo‘shish" icon="pi pi-plus" class="p-button-success p-button-sm"
            @click="openAdd(topic.test_topic_id)" />
        </div>

        <DataTable :value="questionsByTopic[topic.test_topic_id]" :loading="loadingTopics[topic.test_topic_id]"
          class="p-datatable-sm" responsiveLayout="scroll" :rows="10" paginator
          :rowsPerPageOptions="[10, 25, 50]">
          <template #empty>
            <div class="table-empty">Bu mavzuda hali savol yo‘q.</div>
          </template>

          <Column field="test_question_id" header="ID" style="width: 70px" />
          <Column field="question" header="Savol" />
          <Column header="To‘g‘ri javob" style="min-width: 200px">
            <template #body="{ data }">
              <span class="correct-pill">
                <i class="pi pi-check"></i>
                {{ correctAnswerText(data) }}
              </span>
            </template>
          </Column>
          <Column header="Amallar" style="width: 130px">
            <template #body="{ data }">
              <Button icon="pi pi-pencil" class="p-button-rounded p-button-text p-button-info mr-1"
                @click="openEdit(data, topic.test_topic_id)" v-tooltip.top="'Tahrirlash'" />
              <Button icon="pi pi-trash" class="p-button-rounded p-button-text p-button-danger"
                @click="confirmDelete(data, topic.test_topic_id)" v-tooltip.top="'O‘chirish'" />
            </template>
          </Column>
        </DataTable>
      </AccordionTab>
    </Accordion>

    <p v-if="!topics.length && !loading" class="no-topics">
      Avval <b>Test mavzulari</b> bo‘limidan mavzu qo‘shing.
    </p>

    <!-- Add / Edit Dialog (yagona) -->
    <Dialog v-model:visible="showDialog" :header="dialogMode === 'add' ? 'Savol qo‘shish' : 'Savolni tahrirlash'"
      :modal="true" :closable="true" :style="{ width: '560px' }" class="question-dialog">
      <div class="form">
        <div class="field">
          <label class="field-label">Savol matni <span class="req">*</span></label>
          <Textarea v-model="form.question" rows="2" autoResize placeholder="Savolni kiriting" class="w-full"
            :class="{ 'p-invalid': submitted && !form.question }" />
        </div>

        <div class="field">
          <label class="field-label">Variantlar — to‘g‘risini belgilang <span class="req">*</span></label>
          <div v-for="opt in answerKeys" :key="opt.key" class="answer-row"
            :class="{ 'answer-row--correct': form.correct_answer === opt.key }">
            <label class="answer-radio" v-tooltip.top="'To‘g‘ri javob deb belgilash'">
              <input type="radio" name="correctAnswer" :value="opt.key" v-model="form.correct_answer" />
            </label>
            <span class="answer-letter">{{ opt.letter }}</span>
            <InputText v-model="form[opt.key]" :placeholder="'Variant ' + opt.letter" class="w-full"
              :class="{ 'p-invalid': submitted && !form[opt.key] }" />
          </div>
          <small v-if="submitted && !form.correct_answer" class="p-error">To‘g‘ri javobni belgilang.</small>
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
      :style="{ width: '380px' }">
      <span>Ushbu savolni o‘chirishni istaysizmi? Bu amalni qaytarib bo‘lmaydi.</span>
      <template #footer>
        <Button label="Bekor qilish" icon="pi pi-times" class="p-button-text" @click="showDeleteDialog = false" />
        <Button label="O‘chirish" icon="pi pi-trash" class="p-button-danger" :loading="deleting" @click="deleteQuestion" />
      </template>
    </Dialog>
  </div>
</template>

<script>
import { ref, reactive, onMounted } from 'vue'
import { useStore } from 'vuex'
import { useToast } from 'primevue/usetoast'
import Accordion from 'primevue/accordion'
import AccordionTab from 'primevue/accordiontab'

const ANSWER_KEYS = [
  { key: 'answer_1', letter: 'A' },
  { key: 'answer_2', letter: 'B' },
  { key: 'answer_3', letter: 'C' },
  { key: 'answer_4', letter: 'D' },
]

const emptyForm = (topicId = null) => ({
  test_question_id: null,
  question: '',
  answer_1: '',
  answer_2: '',
  answer_3: '',
  answer_4: '',
  correct_answer: '',
  test_topic_id: topicId,
})

export default {
  name: 'TestQuestions',
  components: { Accordion, AccordionTab },
  setup() {
    const store = useStore()
    const toast = useToast()

    const topics = ref([])
    const questionsByTopic = reactive({})
    const loadingTopics = reactive({})
    const loading = ref(false)

    const showDialog = ref(false)
    const dialogMode = ref('add') // 'add' | 'edit'
    const form = ref(emptyForm())
    const submitted = ref(false)
    const saving = ref(false)

    const showDeleteDialog = ref(false)
    const deleting = ref(false)
    const deleteTarget = reactive({ id: null, topicId: null })

    // --- Loading --------------------------------------------------------
    const fetchTopicsAndQuestions = async () => {
      loading.value = true
      topics.value = await store.dispatch('fetchTestTopics')
      for (const topic of topics.value) {
        await refreshQuestionsForTopic(topic.test_topic_id)
      }
      loading.value = false
    }
    const refreshQuestionsForTopic = async (topicId) => {
      loadingTopics[topicId] = true
      questionsByTopic[topicId] = await store.dispatch('fetchTestQuestionsByTopic', topicId)
      loadingTopics[topicId] = false
    }

    // --- Helpers --------------------------------------------------------
    const correctAnswerText = (q) => {
      const txt = q && q.correct_answer ? q[q.correct_answer] : ''
      return txt || '—'
    }
    const isValid = () => {
      const f = form.value
      return (
        f.question.trim() &&
        f.answer_1.trim() && f.answer_2.trim() && f.answer_3.trim() && f.answer_4.trim() &&
        f.correct_answer && f.test_topic_id
      )
    }

    // --- Add / Edit -----------------------------------------------------
    const openAdd = (topicId) => {
      form.value = emptyForm(topicId)
      dialogMode.value = 'add'
      submitted.value = false
      showDialog.value = true
    }
    const openEdit = (question, topicId) => {
      form.value = { ...emptyForm(topicId), ...question, test_topic_id: topicId }
      dialogMode.value = 'edit'
      submitted.value = false
      showDialog.value = true
    }
    const save = async () => {
      submitted.value = true
      if (!isValid()) {
        toast.add({ severity: 'warn', summary: 'To‘ldirilmagan', detail: 'Barcha maydonlarni to‘ldiring va to‘g‘ri javobni belgilang.', life: 3500 })
        return
      }
      saving.value = true
      const payload = { ...form.value }
      let res
      if (dialogMode.value === 'add') {
        res = await store.dispatch('addTestQuestion', payload)
      } else {
        res = await store.dispatch('editTestQuestion', { id: payload.test_question_id, questionData: payload })
      }
      saving.value = false

      if (res) {
        toast.add({ severity: 'success', summary: 'Saqlandi', detail: dialogMode.value === 'add' ? 'Savol qo‘shildi.' : 'Savol yangilandi.', life: 2500 })
        showDialog.value = false
        await refreshQuestionsForTopic(form.value.test_topic_id)
      } else {
        toast.add({ severity: 'error', summary: 'Xatolik', detail: 'Saqlashda xatolik yuz berdi. Qaytadan urinib ko‘ring.', life: 4000 })
      }
    }

    // --- Delete ---------------------------------------------------------
    const confirmDelete = (question, topicId) => {
      deleteTarget.id = question.test_question_id
      deleteTarget.topicId = topicId
      showDeleteDialog.value = true
    }
    const deleteQuestion = async () => {
      deleting.value = true
      const res = await store.dispatch('deleteTestQuestion', deleteTarget.id)
      deleting.value = false
      showDeleteDialog.value = false
      if (res) {
        toast.add({ severity: 'success', summary: 'O‘chirildi', detail: 'Savol o‘chirildi.', life: 2500 })
        await refreshQuestionsForTopic(deleteTarget.topicId)
      } else {
        toast.add({ severity: 'error', summary: 'Xatolik', detail: 'O‘chirishda xatolik yuz berdi.', life: 4000 })
      }
    }

    onMounted(fetchTopicsAndQuestions)

    return {
      topics, questionsByTopic, loadingTopics, loading,
      showDialog, dialogMode, form, submitted, saving,
      showDeleteDialog, deleting,
      answerKeys: ANSWER_KEYS,
      correctAnswerText,
      openAdd, openEdit, save,
      confirmDelete, deleteQuestion,
    }
  },
}
</script>

<style scoped>
.test-questions-container {
  padding: 2rem;
}

.page-header {
  margin-bottom: 1.5rem;
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

/* Accordion header */
.topic-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  gap: 12px;
}
.topic-header-name {
  font-weight: 600;
}
.topic-header-count {
  font-size: 0.8rem;
  color: #64748b;
  background: #eef2f7;
  border-radius: 999px;
  padding: 2px 10px;
}

.topic-toolbar {
  margin-bottom: 12px;
}

.table-empty,
.no-topics {
  text-align: center;
  color: #64748b;
  padding: 16px;
}

/* Correct answer pill in table */
.correct-pill {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #e6f4ea;
  color: #1e7e34;
  border-radius: 999px;
  padding: 3px 12px;
  font-weight: 600;
  font-size: 0.85rem;
}
.correct-pill .pi {
  font-size: 0.7rem;
}

/* Dialog form */
.form {
  display: flex;
  flex-direction: column;
  gap: 18px;
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

.answer-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 8px;
  border: 1.5px solid #e5e9f0;
  border-radius: 8px;
  margin-bottom: 8px;
  transition: border-color 0.15s, background 0.15s;
}
.answer-row--correct {
  border-color: #34c759;
  background: #f1fbf4;
}
.answer-radio {
  display: flex;
  align-items: center;
  cursor: pointer;
}
.answer-radio input {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #34c759;
}
.answer-letter {
  width: 22px;
  text-align: center;
  font-weight: 700;
  color: #475569;
}

.w-full {
  width: 100%;
}
.mr-1 {
  margin-right: 4px;
}
</style>
