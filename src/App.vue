<script setup lang="ts">
import { ref, computed } from 'vue'
import FullCalendar from '@fullcalendar/vue3'
import dayGridPlugin from '@fullcalendar/daygrid'
import interactionPlugin from '@fullcalendar/interaction'
import locale from '@fullcalendar/core/locales/zh-cn'

// ── 类型定义 ──
type Category = 'work' | 'study' | 'life' | 'other'

interface Task {
  id: string
  title: string
  start: string
  end: string
  allDay: boolean
  location?: string
  description?: string
  category: Category
  recurring?: 'none' | 'daily' | 'weekly' | 'monthly'
  recurringEnd?: string
}

// ── 分类配置 ──
const categories: { value: Category; label: string; color: string; bg: string }[] = [
  { value: 'work', label: '工作', color: '#dc2626', bg: '#fee2e2' },
  { value: 'study', label: '学习', color: '#2563eb', bg: '#dbeafe' },
  { value: 'life', label: '生活', color: '#16a34a', bg: '#dcfce7' },
  { value: 'other', label: '其他', color: '#9333ea', bg: '#f3e8ff' },
]

function getCategoryConfig(cat: Category) {
  return categories.find((c) => c.value === cat) || categories[3]
}

// ── 状态 ──
const showModal = ref(false)
const editingTask = ref<Task | null>(null)
const form = ref({
  title: '',
  date: '',
  startTime: '09:00',
  endTime: '10:00',
  location: '',
  description: '',
  category: 'life' as Category,
  recurring: 'none' as 'none' | 'daily' | 'weekly' | 'monthly',
  recurringEnd: '',
})

// ── localStorage 读写 ──
function loadTasks(): Task[] {
  try {
    return JSON.parse(localStorage.getItem('calendar_tasks') || '[]')
  } catch {
    return []
  }
}

function saveTasks(tasks: Task[]) {
  localStorage.setItem('calendar_tasks', JSON.stringify(tasks))
}

// ── 工具函数 ──
const pad = (n: number) => String(n).padStart(2, '0')

function formatDate(d: Date) {
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}`
}

// ── 展开循环日程为具体事件 ──
function expandRecurringTasks(tasks: Task[]) {
  const events: any[] = []
  const today = new Date()
  const viewEnd = new Date()
  viewEnd.setMonth(viewEnd.getMonth() + 3)

  for (const task of tasks) {
    if (!task.recurring || task.recurring === 'none') {
      events.push(task)
      continue
    }

    const startDate = new Date(task.start)
    const endDate = task.end ? new Date(task.end) : null
    const recEnd = task.recurringEnd
      ? new Date(task.recurringEnd + 'T23:59:59')
      : viewEnd

    if (recEnd < today) continue

    let current = new Date(startDate)
    let count = 0
    const maxIterations = 500

    while (current <= recEnd && current <= viewEnd && count < maxIterations) {
      if (current >= today) {
        const newStart = new Date(current)
        newStart.setHours(startDate.getHours(), startDate.getMinutes(), startDate.getSeconds())

        const newEnd = endDate ? new Date(current) : null
        if (newEnd && endDate) {
          newEnd.setHours(endDate.getHours(), endDate.getMinutes(), endDate.getSeconds())
        }

        const cat = getCategoryConfig(task.category)
        events.push({
          ...task,
          id: `${task.id}_${formatDate(current)}`,
          start: newStart.toISOString(),
          end: newEnd ? newEnd.toISOString() : undefined,
          _parentId: task.id,
          backgroundColor: cat.color,
          borderColor: cat.color,
          textColor: '#fff',
        })
      }

      count++
      if (task.recurring === 'daily') {
        current.setDate(current.getDate() + 1)
      } else if (task.recurring === 'weekly') {
        current.setDate(current.getDate() + 7)
      } else if (task.recurring === 'monthly') {
        current.setMonth(current.getMonth() + 1)
      }
    }
  }

  return events
}

// ── 日历事件 ──
const rawTasks = ref(loadTasks())

function getCalendarEvents() {
  return expandRecurringTasks(rawTasks.value)
}

const calendarEvents = ref(getCalendarEvents())

// ── 日历配置 ──
const calendarOptions = computed(() => ({
  plugins: [dayGridPlugin, interactionPlugin],
  initialView: 'dayGridMonth',
  locale,
  headerToolbar: {
    left: 'prev,next today',
    center: 'title',
    right: 'dayGridMonth',
  },
  buttonText: {
    today: '今天',
  },
  editable: true,
  selectable: true,
  events: calendarEvents.value,
  dateClick: (info: any) => {
    resetForm()
    form.value.date = info.dateStr
    showModal.value = true
  },
  eventClick: (info: any) => {
    editTask(info.event)
  },
  eventDrop: (info: any) => {
    handleEventDrop(info)
  },
}))

function resetForm() {
  editingTask.value = null
  form.value = {
    title: '',
    date: new Date().toISOString().slice(0, 10),
    startTime: '09:00',
    endTime: '10:00',
    location: '',
    description: '',
    category: 'life',
    recurring: 'none',
    recurringEnd: '',
  }
}

function closeModal() {
  showModal.value = false
  editingTask.value = null
}

// ── 保存日程 ──
function saveTask() {
  if (!form.value.title.trim()) {
    alert('请输入标题')
    return
  }

  const start = `${form.value.date}T${form.value.startTime}:00+08:00`
  const end = `${form.value.date}T${form.value.endTime}:00+08:00`

  const tasks = loadTasks()

  if (editingTask.value) {
    const idx = tasks.findIndex((t) => t.id === editingTask.value!.id)
    if (idx !== -1) {
      tasks[idx] = {
        ...tasks[idx],
        title: form.value.title,
        start,
        end,
        location: form.value.location,
        description: form.value.description,
        category: form.value.category,
        recurring: form.value.recurring,
        recurringEnd: form.value.recurringEnd || undefined,
      }
    }
  } else {
    const newTask: Task = {
      id: 'task_' + Date.now(),
      title: form.value.title,
      start,
      end,
      allDay: false,
      location: form.value.location,
      description: form.value.description,
      category: form.value.category,
      recurring: form.value.recurring,
      recurringEnd: form.value.recurringEnd || undefined,
    }
    tasks.push(newTask)
  }

  saveTasks(tasks)
  rawTasks.value = tasks
  calendarEvents.value = getCalendarEvents()
  closeModal()
}

// ── 编辑日程 ──
function editTask(event: any) {
  let searchId = event.id

  // 如果是循环日程展开的事件，ID 格式为 "原始ID_日期"
  if (event._parentId) {
    searchId = event._parentId
  } else if (event.id.includes('_')) {
    // 去掉最后的 _日期 部分，取原始任务 ID
    const parts = event.id.split('_')
    // 最后一部分如果是日期格式（如 2026-10-05），则去掉
    if (parts.length >= 2 && /^\d{4}-\d{2}-\d{2}$/.test(parts[parts.length - 1])) {
      searchId = parts.slice(0, -1).join('_')
    }
  }

  const tasks = loadTasks()
  const task = tasks.find((t) => t.id === searchId)
  if (!task) {
    console.warn('找不到任务:', searchId, '原始event.id:', event.id)
    return
  }
  editingTask.value = task

  const startDate = new Date(task.start)
  const endDate = task.end ? new Date(task.end) : null

  form.value = {
    title: task.title,
    date: `${startDate.getFullYear()}-${pad(startDate.getMonth() + 1)}-${pad(startDate.getDate())}`,
    startTime: `${pad(startDate.getHours())}:${pad(startDate.getMinutes())}`,
    endTime: endDate ? `${pad(endDate.getHours())}:${pad(endDate.getMinutes())}` : '10:00',
    location: task.location || '',
    description: task.description || '',
    category: task.category,
    recurring: task.recurring || 'none',
    recurringEnd: task.recurringEnd || '',
  }
  showModal.value = true
}

// ── 删除日程 ──
function deleteTask() {
  if (!editingTask.value) {
    console.warn('没有正在编辑的任务')
    return
  }

  const targetId = editingTask.value.id
  const tasks = loadTasks()
  const filtered = tasks.filter((t) => t.id !== targetId)

  if (filtered.length === tasks.length) {
    console.warn('删除失败，未找到ID:', targetId)
    return
  }

  saveTasks(filtered)
  rawTasks.value = filtered
  calendarEvents.value = getCalendarEvents()
  closeModal()
}

// ── 拖拽后更新日期 ──
function handleEventDrop(info: any) {
  const tasks = loadTasks()
  const parentId = info.event._parentId
  const searchId = parentId || info.event.id
  const idx = tasks.findIndex((t) => t.id === searchId)
  if (idx === -1) return

  const newDate = info.event.startStr.slice(0, 10)
  const oldStart = new Date(tasks[idx].start)
  const oldEnd = tasks[idx].end ? new Date(tasks[idx].end) : null

  const startTime = `${pad(oldStart.getHours())}:${pad(oldStart.getMinutes())}`
  const endTime = oldEnd ? `${pad(oldEnd.getHours())}:${pad(oldEnd.getMinutes())}` : '10:00'

  tasks[idx].start = `${newDate}T${startTime}:00+08:00`
  tasks[idx].end = `${newDate}T${endTime}:00+08:00`

  saveTasks(tasks)
  rawTasks.value = tasks
  calendarEvents.value = getCalendarEvents()
}

// ── 导出到手机日历 ──
function exportToICS() {
  const tasks = loadTasks()
  if (tasks.length === 0) {
    alert('没有日程可导出')
    return
  }

  let ics = 'BEGIN:VCALENDAR\nVERSION:2.0\nPRODID:-//MyCalendar//CN\n'

  tasks.forEach((task) => {
    const start = task.start.replace(/[-:]/g, '').replace('+08:00', '')
    const end = task.end.replace(/[-:]/g, '').replace('+08:00', '')
    ics += 'BEGIN:VEVENT\n'
    ics += `SUMMARY:[${getCategoryConfig(task.category).label}] ${task.title}\n`
    ics += `DTSTART:${start}\n`
    ics += `DTEND:${end}\n`
    if (task.location) ics += `LOCATION:${task.location}\n`
    if (task.description) ics += `DESCRIPTION:${task.description}\n`
    if (task.recurring && task.recurring !== 'none') {
      let rrule = ''
      if (task.recurring === 'daily') rrule = 'FREQ=DAILY'
      if (task.recurring === 'weekly') rrule = 'FREQ=WEEKLY'
      if (task.recurring === 'monthly') rrule = 'FREQ=MONTHLY'
      if (task.recurringEnd) rrule += `;UNTIL=${task.recurringEnd.replace(/-/g, '')}T235959`
      ics += `RRULE:${rrule}\n`
    }
    ics += 'END:VEVENT\n'
  })

  ics += 'END:VCALENDAR'

  const blob = new Blob([ics], { type: 'text/calendar;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'my-calendar.ics'
  a.click()
  URL.revokeObjectURL(url)
}
</script>

<template>
  <div class="app">
    <header class="header">
      <h1>📅 我的日程</h1>
      <div class="actions">
        <button class="btn btn-primary" @click="resetForm(); showModal = true">+ 新建日程</button>
        <button class="btn btn-secondary" @click="exportToICS">📥 导出到手机日历</button>
      </div>
    </header>

    <!-- 分类图例 -->
    <div class="legend">
      <span v-for="cat in categories" :key="cat.value" class="legend-item">
        <span class="legend-dot" :style="{ background: cat.color }"></span>
        {{ cat.label }}
      </span>
    </div>

    <div class="calendar-wrapper">
      <FullCalendar :options="calendarOptions" />
    </div>

    <!-- 弹窗 -->
    <div v-if="showModal" class="modal-overlay" @click.self="closeModal">
      <div class="modal">
        <h2>{{ editingTask ? '编辑日程' : '新建日程' }}</h2>

        <!-- 分类选择 -->
        <div class="form-group">
          <label>分类</label>
          <div class="category-picker">
            <button
              v-for="cat in categories"
              :key="cat.value"
              type="button"
              class="category-btn"
              :class="{ active: form.category === cat.value }"
              :style="{
                borderColor: cat.color,
                background: form.category === cat.value ? cat.bg : 'transparent',
                color: cat.color,
              }"
              @click="form.category = cat.value"
            >
              {{ cat.label }}
            </button>
          </div>
        </div>

        <div class="form-group">
          <label>标题</label>
          <input v-model="form.title" placeholder="输入日程标题" />
        </div>
        <div class="form-row">
          <div class="form-group">
            <label>日期</label>
            <input type="date" v-model="form.date" />
          </div>
          <div class="form-group">
            <label>开始时间</label>
            <input type="time" v-model="form.startTime" />
          </div>
          <div class="form-group">
            <label>结束时间</label>
            <input type="time" v-model="form.endTime" />
          </div>
        </div>
        <div class="form-group">
          <label>地点</label>
          <input v-model="form.location" placeholder="可选" />
        </div>
        <div class="form-group">
          <label>描述</label>
          <textarea v-model="form.description" placeholder="可选" rows="3"></textarea>
        </div>

        <!-- 循环设置 -->
        <div class="form-group">
          <label>循环方式</label>
          <select v-model="form.recurring" class="form-select">
            <option value="none">不循环</option>
            <option value="daily">每天</option>
            <option value="weekly">每周</option>
            <option value="monthly">每月</option>
          </select>
        </div>
        <div v-if="form.recurring !== 'none'" class="form-group">
          <label>结束日期（留空则无限循环）</label>
          <input type="date" v-model="form.recurringEnd" />
        </div>

        <div class="modal-actions">
          <button v-if="editingTask" class="btn btn-danger" @click="deleteTask">删除</button>
          <button class="btn btn-secondary" @click="closeModal">取消</button>
          <button class="btn btn-primary" @click="saveTask">保存</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  flex-direction: column;
  padding: 16px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #f5f5f5;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  flex-shrink: 0;
}

.header h1 {
  margin: 0;
  font-size: 1.5rem;
}

.actions {
  display: flex;
  gap: 10px;
}

.btn {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
}

.btn-primary {
  background: #3b82f6;
  color: white;
}

.btn-secondary {
  background: #e5e7eb;
  color: #374151;
}

.btn-danger {
  background: #ef4444;
  color: white;
}

/* 分类图例 */
.legend {
  display: flex;
  gap: 16px;
  margin-bottom: 12px;
  flex-shrink: 0;
  flex-wrap: wrap;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.85rem;
  color: #6b7280;
}

.legend-dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

/* 日历 */
.calendar-wrapper {
  flex: 1;
  background: white;
  border-radius: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  padding: 16px;
  overflow: hidden;
  min-height: 0;
}

.calendar-wrapper :deep(.fc) {
  height: 100%;
}

.calendar-wrapper :deep(.fc-view-harness) {
  flex: 1;
}

/* 弹窗 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  border-radius: 12px;
  padding: 24px;
  width: 420px;
  max-width: 90vw;
  max-height: 90vh;
  overflow-y: auto;
}

.modal h2 {
  margin: 0 0 16px;
  font-size: 1.2rem;
}

.form-group {
  margin-bottom: 12px;
}

.form-group label {
  display: block;
  margin-bottom: 4px;
  font-size: 0.85rem;
  color: #6b7280;
}

.form-group input,
.form-group textarea,
.form-select {
  width: 100%;
  padding: 8px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.95rem;
  box-sizing: border-box;
}

.form-group input:focus,
.form-group textarea:focus,
.form-select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.1);
}

.form-row {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr;
  gap: 10px;
}

/* 分类选择器 */
.category-picker {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.category-btn {
  padding: 6px 14px;
  border: 2px solid;
  border-radius: 20px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s;
}

.category-btn.active {
  font-weight: 600;
}

/* 弹窗按钮 */
.modal-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}

/* 移动端适配 */
@media (max-width: 600px) {
  .header {
    flex-direction: column;
    gap: 10px;
  }

  .form-row {
    grid-template-columns: 1fr;
  }

  .modal {
    width: 95vw;
    padding: 16px;
  }
}
</style>