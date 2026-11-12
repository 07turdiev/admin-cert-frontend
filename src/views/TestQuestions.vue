<template>
  <div class="test-questions-container">
    <Toast />

    <div class="page-header">
      <div>
        <h2 class="page-title">Test savollari</h2>
        <p class="page-subtitle">Barcha savollar bitta umumiy bazada saqlanadi — choraklarga bo‘linmaydi.
          Har bir foydalanuvchiga butun bazadan tasodifiy {{ questionsPerTest }} ta savol tushadi
          (o‘tish bali — {{ passScore }} ta to‘g‘ri javob).</p>
      </div>
      <Button label="Savol qo‘shish" icon="pi pi-plus" class="p-button-success" @click="openAdd" />
    </div>

    <div class="pool-summary" :class="{ 'pool-summary--warn': poolTooSmall }">
      <i class="pi" :class="poolTooSmall ? 'pi-exclamation-triangle' : 'pi-database'"></i>
      <div>
        <div class="pool-summary-count">
          Bazada jami: <b>{{ loading ? '…' : questions.length }}</b> ta savol
          <span v-if="!loading && duplicateCount" class="dup-note">
            · {{ duplicateCount }} tasi takroriy (testda bir marta hisoblanadi)
          </span>
        </div>
        <div v-if="poolTooSmall" class="pool-summary-hint">
          Kamida {{ questionsPerTest }} ta turli savol kiriting — hozircha foydalanuvchilarga faqat {{ uniqueCount }} ta savol tushadi.
        </div>
        <div v-else class="pool-summary-hint">Savollar qancha ko‘p bo‘lsa, har bir foydalanuvchining testi shuncha farqli bo‘ladi.</div>
      </div>
    </div>

    <div class="toolbar">
      <div class="search">
        <i class="pi pi-search search-icon"></i>
        <InputText v-model="search" placeholder="Savol yoki javob bo‘yicha qidirish" class="w-full search-input" />
      </div>
      <label v-if="duplicateCount" class="dup-filter">
        <input v-model="onlyDuplicates" type="checkbox" />
        Faqat takroriy savollar
      </label>
    </div>

    <DataTable :value="filteredQuestions" :loading="loading" class="p-datatable-sm" responsiveLayout="scroll"
      paginator :rows="20" :rowsPerPageOptions="[20, 50, 100]" dataKey="test_question_id">
      <template #empty>
        <div class="table-empty">{{ search || onlyDuplicates ? 'Hech narsa topilmadi.' : 'Hali savol qo‘shilmagan.' }}</div>
      </template>

      <Column field="test_question_id" header="ID" style="width: 70px" />
      <Column header="Savol">
        <template #body="{ data }">
          <div class="question-cell">
            <span>{{ data.question }}</span>
            <span v-if="isDuplicate(data)" class="dup-pill" v-tooltip.top="'Shu matnli savol bazada bir necha marta bor'">takroriy</span>
          </div>
        </template>
      </Column>
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
          <Button icon="pi pi-pencil" class="p-button-rounded p-button-text action-btn action-edit"
            @click="openEdit(data)" v-tooltip.top="'Tahrirlash'" />
          <Button icon="pi pi-trash" class="p-button-rounded p-button-text action-btn action-delete"
            @click="confirmDelete(data)" v-tooltip.top="'O‘chirish'" />
        </template>
      </Column>
    </DataTable>

    <!-- Qo'shish / tahrirlash -->
    <Dialog v-model:visible="showDialog" :header="dialogMode === 'add' ? 'Savol qo‘shish' : 'Savolni tahrirlash'"
      :modal="true" :closable="true" :style="{ width: '560px' }" class="question-dialog">
      <div class="form">
        <div class="field">
          <label class="field-label">Savol matni <span class="req">*</span></label>
          <Textarea v-model="form.question" rows="2" autoResize placeholder="Savolni kiriting" class="w-full"
            :class="{ 'p-invalid': submitted && !form.question }" />
          <small v-if="formDuplicate" class="dup-warning">
            <i class="pi pi-exclamation-triangle"></i> Bunday matnli savol bazada allaqachon bor.
          </small>
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
        <Button v-if="dialogMode === 'add'" label="Saqlash va yana qo‘shish" icon="pi pi-plus" class="p-button-outlined"
          :loading="saving" @click="save(true)" />
        <Button :label="dialogMode === 'add' ? 'Qo‘shish' : 'Saqlash'" icon="pi pi-check" class="p-button-success"
          :loading="saving" @click="save(false)" />
      </template>
    </Dialog>

    <!-- O'chirishni tasdiqlash -->
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
import { ref, computed, onMounted } from 'vue'
import { useStore } from 'vuex'
import { useToast } from 'primevue/usetoast'

// Backend bilan bir xil: certificate_backend/src/helpers/test.js
const QUESTIONS_PER_TEST = 20
const PASS_SCORE = 15

const ANSWER_KEYS = [
  { key: 'answer_1', letter: 'A' },
  { key: 'answer_2', letter: 'B' },
  { key: 'answer_3', letter: 'C' },
  { key: 'answer_4', letter: 'D' },
]

const emptyForm = () => ({
  test_question_id: null,
  question: '',
  answer_1: '',
  answer_2: '',
  answer_3: '',
  answer_4: '',
  correct_answer: '',
})

// Takroriylarni aniqlash uchun matnni bir xil ko'rinishga keltirish
const normalize = (text) => String(text || '').toLowerCase().replace(/\s+/g, ' ').trim()

export default {
  name: 'TestQuestions',
  setup() {
    const store = useStore()
    const toast = useToast()

    const questions = ref([])
    const loading = ref(false)
    const search = ref('')
    const onlyDuplicates = ref(false)

    const showDialog = ref(false)
    const dialogMode = ref('add')
    const form = ref(emptyForm())
    const submitted = ref(false)
    const saving = ref(false)

    const showDeleteDialog = ref(false)
    const deleting = ref(false)
    const deleteId = ref(null)

    // --- Hisob-kitoblar -------------------------------------------------
    const textCounts = computed(() => {
      const counts = {}
      questions.value.forEach(q => {
        const key = normalize(q.question)
        counts[key] = (counts[key] || 0) + 1
      })
      return counts
    })
    const isDuplicate = (q) => textCounts.value[normalize(q.question)] > 1
    const uniqueCount = computed(() => Object.keys(textCounts.value).length)
    const duplicateCount = computed(() => questions.value.length - uniqueCount.value)
    const poolTooSmall = computed(() => !loading.value && uniqueCount.value < QUESTIONS_PER_TEST)

    const filteredQuestions = computed(() => {
      const term = normalize(search.value)
      return questions.value.filter(q => {
        if (onlyDuplicates.value && !isDuplicate(q)) return false
        if (!term) return true
        return [q.question, q.answer_1, q.answer_2, q.answer_3, q.answer_4].some(t => normalize(t).includes(term))
      })
    })

    const formDuplicate = computed(() => {
      const key = normalize(form.value.question)
      if (!key) return false
      return questions.value.some(q => normalize(q.question) === key && q.test_question_id !== form.value.test_question_id)
    })

    const correctAnswerText = (q) => {
      const txt = q && q.correct_answer ? q[q.correct_answer] : ''
      return txt || '—'
    }

    // --- Yuklash --------------------------------------------------------
    const fetchQuestions = async () => {
      loading.value = true
      questions.value = await store.dispatch('fetchTestQuestions')
      loading.value = false
    }

    // --- Qo'shish / tahrirlash -----------------------------------------
    const isValid = () => {
      const f = form.value
      return (
        f.question.trim() &&
        f.answer_1.trim() && f.answer_2.trim() && f.answer_3.trim() && f.answer_4.trim() &&
        f.correct_answer
      )
    }
    const openAdd = () => {
      form.value = emptyForm()
      dialogMode.value = 'add'
      submitted.value = false
      showDialog.value = true
    }
    const openEdit = (question) => {
      form.value = { ...emptyForm(), ...question }
      dialogMode.value = 'edit'
      submitted.value = false
      showDialog.value = true
    }
    const save = async (addAnother) => {
      submitted.value = true
      if (!isValid()) {
        toast.add({ severity: 'warn', summary: 'To‘ldirilmagan', detail: 'Barcha maydonlarni to‘ldiring va to‘g‘ri javobni belgilang.', life: 3500 })
        return
      }
      saving.value = true
      const { test_question_id, question, answer_1, answer_2, answer_3, answer_4, correct_answer } = form.value
      const payload = { question, answer_1, answer_2, answer_3, answer_4, correct_answer }
      const res = dialogMode.value === 'add'
        ? await store.dispatch('addTestQuestion', payload)
        : await store.dispatch('editTestQuestion', { id: test_question_id, questionData: payload })
      saving.value = false

      if (!res) {
        toast.add({ severity: 'error', summary: 'Xatolik', detail: 'Saqlashda xatolik yuz berdi. Qaytadan urinib ko‘ring.', life: 4000 })
        return
      }
      toast.add({ severity: 'success', summary: 'Saqlandi', detail: dialogMode.value === 'add' ? 'Savol qo‘shildi.' : 'Savol yangilandi.', life: 2500 })
      await fetchQuestions()
      if (addAnother) {
        form.value = emptyForm()
        submitted.value = false
      } else {
        showDialog.value = false
      }
    }

    // --- O'chirish ------------------------------------------------------
    const confirmDelete = (question) => {
      deleteId.value = question.test_question_id
      showDeleteDialog.value = true
    }
    const deleteQuestion = async () => {
      deleting.value = true
      const res = await store.dispatch('deleteTestQuestion', deleteId.value)
      deleting.value = false
      showDeleteDialog.value = false
      if (res) {
        toast.add({ severity: 'success', summary: 'O‘chirildi', detail: 'Savol o‘chirildi.', life: 2500 })
        await fetchQuestions()
      } else {
        toast.add({ severity: 'error', summary: 'Xatolik', detail: 'O‘chirishda xatolik yuz berdi.', life: 4000 })
      }
    }

    onMounted(fetchQuestions)

    return {
      questions, loading, search, onlyDuplicates, filteredQuestions,
      uniqueCount, duplicateCount, poolTooSmall, isDuplicate, formDuplicate,
      questionsPerTest: QUESTIONS_PER_TEST, passScore: PASS_SCORE,
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
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 1.25rem;
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
  max-width: 760px;
}

/* Savollar bazasi xulosasi */
.pool-summary {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px 18px;
  margin-bottom: 1rem;
  border-radius: 12px;
  background: #eef4fd;
  border: 1px solid #d3e1f7;
  color: #1e3a6e;
}
.pool-summary .pi {
  font-size: 1.4rem;
  color: #216ad7;
}
.pool-summary--warn {
  background: #fff6e5;
  border-color: #f5d49a;
  color: #7a4b00;
}
.pool-summary--warn .pi {
  color: #d97706;
}
.pool-summary-count {
  font-size: 1rem;
}
.pool-summary-hint {
  font-size: 0.85rem;
  opacity: 0.85;
  margin-top: 2px;
}
.dup-note {
  color: #a15c07;
  font-size: 0.9rem;
}

.toolbar {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 1rem;
}
.search {
  position: relative;
  flex: 1;
  min-width: 260px;
  max-width: 520px;
}
.search-icon {
  position: absolute;
  left: 12px;
  top: 50%;
  transform: translateY(-50%);
  color: #94a3b8;
  pointer-events: none;
}
.search-input {
  padding-left: 2.25rem;
}
.dup-filter {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  color: #475569;
  cursor: pointer;
}

.table-empty {
  text-align: center;
  color: #64748b;
  padding: 16px;
}

.question-cell {
  display: flex;
  align-items: flex-start;
  gap: 8px;
}
.dup-pill {
  flex-shrink: 0;
  background: #fff6e5;
  color: #a15c07;
  border-radius: 999px;
  padding: 1px 8px;
  font-size: 0.75rem;
  font-weight: 600;
}
.dup-warning {
  color: #a15c07;
}

/* Amal tugmalari (App.vue dagi umumiy p-button-info/danger fonini chetlab o'tadi) */
.action-btn {
  width: 2.25rem;
  height: 2.25rem;
  margin-right: 4px;
  background: transparent !important;
  border-color: transparent !important;
}
.action-edit {
  color: #2563eb !important;
}
.action-edit:hover {
  background: #eff6ff !important;
}
.action-delete {
  color: #dc2626 !important;
}
.action-delete:hover {
  background: #fef2f2 !important;
}

/* To'g'ri javob */
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

/* Forma */
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
