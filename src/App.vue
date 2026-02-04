<script setup>
import { ref, onMounted, computed, watch } from 'vue'
import Papa from 'papaparse'
import { 
  Search, 
  Filter, 
  Download, 
  ChevronLeft, 
  ChevronRight,
  Clock,
  User,
  Tag,
  Layers,
  Calendar,
  Copy,
  Check,
  Upload,
  FileText
} from 'lucide-vue-next'

const tasks = ref([])
const loading = ref(false)
const error = ref(null)
const fileName = ref('No file selected')
const searchQuery = ref('')
const selectedSections = ref([])
const selectedAssignee = ref('')
const filterMonth = ref('')
const filterDate = ref('')

const currentPage = ref(1)
const itemsPerPage = 20

// State for filters
const sections = computed(() => {
  const s = new Set(tasks.value.map(t => t['Section/Column']).filter(Boolean))
  return Array.from(s).sort()
})

const toggleSection = (section) => {
  const index = selectedSections.value.indexOf(section)
  if (index > -1) {
    selectedSections.value.splice(index, 1)
  } else {
    selectedSections.value.push(section)
  }
}

const assignees = computed(() => {
  const a = new Set(tasks.value.map(t => t['Assignee']).filter(Boolean))
  return Array.from(a).sort()
})

const completedMonths = computed(() => {
  const months = new Set()
  tasks.value.forEach(t => {
    const dateStr = t['Completed At']
    if (dateStr) {
      const date = new Date(dateStr)
      if (!isNaN(date.getTime())) {
        const month = date.toLocaleString('en-US', { month: 'long', year: 'numeric' })
        months.add(month)
      }
    }
  })
  return Array.from(months).sort((a, b) => new Date(b) - new Date(a))
})

const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  loading.value = true
  fileName.value = file.name
  
  Papa.parse(file, {
    header: true,
    skipEmptyLines: true,
    complete: (results) => {
      tasks.value = results.data
      loading.value = false
      currentPage.value = 1
    },
    error: (err) => {
      error.value = "Failed to parse CSV: " + err.message
      loading.value = false
    }
  })
}

onMounted(() => {
  loading.value = false
})

const filteredTasks = computed(() => {
  let filtered = tasks.value

  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase()
    filtered = filtered.filter(t => 
      (t.Name && t.Name.toLowerCase().includes(q)) ||
      (t['Task ID'] && t['Task ID'].toLowerCase().includes(q)) ||
      (t.Notes && t.Notes.toLowerCase().includes(q))
    )
  }

  if (selectedSections.value.length > 0) {
    filtered = filtered.filter(t => selectedSections.value.includes(t['Section/Column']))
  }

  if (selectedAssignee.value) {
    filtered = filtered.filter(t => t['Assignee'] === selectedAssignee.value)
  }

  if (filterMonth.value) {
    filtered = filtered.filter(t => {
      const dateStr = t['Completed At']
      if (!dateStr) return false
      const date = new Date(dateStr)
      return date.toLocaleString('en-US', { month: 'long', year: 'numeric' }) === filterMonth.value
    })
  }

  if (filterDate.value) {
    filtered = filtered.filter(t => {
      const dateStr = t['Completed At']
      if (!dateStr) return false
      // Match YYYY-MM-DD
      return dateStr.startsWith(filterDate.value)
    })
  }

  return filtered
})

const paginatedTasks = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage
  const end = start + itemsPerPage
  return filteredTasks.value.slice(start, end)
})

const totalPages = computed(() => {
  return Math.ceil(filteredTasks.value.length / itemsPerPage)
})

const visibleColumns = [
  { key: 'Task ID', label: 'ID' },
  { key: 'Name', label: 'Task Name' },
  { key: 'Section/Column', label: 'Section' },
  { key: 'Assignee', label: 'Assignee' },
  { key: 'Notes', label: 'Description' },
  { key: 'Created At', label: 'Created' },
  { key: 'Completed At', label: 'Completed' },
  { key: 'Tags', label: 'Tags' }
]

const copySuccess = ref(false)

const copyToClipboard = () => {
  // 1. Generate Plain Text (TSV)
  const headersText = visibleColumns.map(col => col.label).join('\t')
  const rowsText = filteredTasks.value.map(task => {
    return visibleColumns.map(col => {
      let val = task[col.key] || ''
      return val.toString().replace(/\n/g, ' ')
    }).join('\t')
  }).join('\n')
  const plainText = headersText + '\n' + rowsText

  // 2. Generate HTML Table
  const headersHtml = visibleColumns.map(col => `<th style="border: 1px solid #ddd; padding: 8px; background-color: #f2f2f2;">${col.label}</th>`).join('')
  const rowsHtml = filteredTasks.value.map(task => {
    const cells = visibleColumns.map(col => {
      let val = task[col.key] || ''
      return `<td style="border: 1px solid #ddd; padding: 8px;">${val.toString().replace(/\n/g, '<br>')}</td>`
    }).join('')
    return `<tr>${cells}</tr>`
  }).join('')
  
  const htmlTable = `
    <table style="border-collapse: collapse; width: 100%; font-family: sans-serif;">
      <thead><tr>${headersHtml}</tr></thead>
      <tbody>${rowsHtml}</tbody>
    </table>
  `

  // 3. Write to Clipboard using ClipboardItem for multi-format support
  const textBlob = new Blob([plainText], { type: 'text/plain' })
  const htmlBlob = new Blob([htmlTable], { type: 'text/html' })

  try {
    const data = [new ClipboardItem({
      'text/plain': textBlob,
      'text/html': htmlBlob
    })]
    
    navigator.clipboard.write(data).then(() => {
      copySuccess.value = true
      setTimeout(() => copySuccess.value = false, 2000)
    }).catch(err => {
      error.value = "Failed to copy: " + err
      console.error(err)
    })
  } catch (err) {
    // Fallback for browsers that don't support ClipboardItem fully
    navigator.clipboard.writeText(plainText).then(() => {
      copySuccess.value = true
      setTimeout(() => copySuccess.value = false, 2000)
    }).catch(e => {
      error.value = "Failed to copy: " + e
    })
  }
}

watch([searchQuery, selectedSections, selectedAssignee, filterMonth, filterDate], () => {
  currentPage.value = 1
}, { deep: true })

const formatDate = (dateStr) => {
  if (!dateStr) return '-'
  return new Date(dateStr).toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric'
  })
}

const getStatusColor = (section) => {
  if (!section) return 'gray'
  const s = section.toLowerCase()
  if (s.includes('done') || s.includes('complete')) return '#10b981'
  if (s.includes('progress')) return '#3b82f6'
  if (s.includes('hold') || s.includes('parking')) return '#f59e0b'
  return '#6366f1'
}

</script>

<template>
  <div class="container">
    <header class="header">
      <div class="title-section">
        <div class="logo-box">
          <Layers class="logo-icon" />
        </div>
        <div>
          <h1>Sprint Explorer</h1>
          <p class="subtitle">Real-time CSV Data Visualization</p>
        </div>
      </div>
      
      <div class="header-actions">
        <div class="file-info glass">
          <FileText :size="16" class="text-dim" />
          <span class="file-name">{{ fileName }}</span>
          <label class="upload-btn">
            <Upload :size="18" />
            Upload CSV
            <input type="file" accept=".csv" @change="handleFileUpload" hidden />
          </label>
        </div>

        <div class="stats glass">
          <div class="stat-item">
            <span class="stat-value">{{ filteredTasks.length.toLocaleString() }}</span>
            <span class="stat-label">Tasks</span>
          </div>
          <div class="stat-divider"></div>
          <button @click="copyToClipboard" class="export-btn" :class="{ 'btn-success': copySuccess }">
            <component :is="copySuccess ? Check : Copy" :size="18" />
            {{ copySuccess ? 'Copied!' : 'Copy' }}
          </button>
        </div>
      </div>
    </header>

    <main>
      <template v-if="tasks.length === 0 && !loading">
        <div class="landing-state glass">
          <div class="landing-content">
            <div class="landing-icon-box">
              <Upload :size="48" class="landing-icon" />
            </div>
            <h2>Ready to explore?</h2>
            <p>Upload your Asana CSV export to get started. All processing happens in your browser - your data stays private.</p>
            <label class="primary-upload-btn">
              <Upload :size="20" />
              Select Asana CSV File
              <input type="file" accept=".csv" @change="handleFileUpload" hidden />
            </label>
          </div>
        </div>
      </template>

      <template v-else>
        <section class="controls glass p-6">
          <div class="search-wrapper">
            <Search class="search-icon" :size="20" />
            <input 
              v-model="searchQuery" 
              type="text" 
              placeholder="Search tasks, IDs, or notes..."
              class="search-input"
            />
          </div>
          
          <div class="filters">
            <div class="filter-group section-filter">
              <label><Filter :size="14" /> Sections (Multi-select)</label>
              <div class="chip-container">
                <button 
                  v-for="s in sections" 
                  :key="s" 
                  class="chip"
                  :class="{ active: selectedSections.includes(s) }"
                  @click="toggleSection(s)"
                >
                  {{ s }}
                </button>
              </div>
            </div>
            
            <div class="filter-controls">
              <div class="filter-group">
                <label><User :size="14" /> Assignee</label>
                <select v-model="selectedAssignee">
                  <option value="">All Assignees</option>
                  <option v-for="a in assignees" :key="a" :value="a">{{ a }}</option>
                </select>
              </div>

              <div class="filter-group">
                <label><Clock :size="14" /> Completed Month</label>
                <select v-model="filterMonth">
                  <option value="">All Time</option>
                  <option v-for="m in completedMonths" :key="m" :value="m">{{ m }}</option>
                </select>
              </div>

              <div class="filter-group">
                <label><Calendar :size="14" /> Specific Date</label>
                <input type="date" v-model="filterDate" class="date-input" />
              </div>
            </div>
          </div>
        </section>

        <div v-if="loading" class="loading-state">
          <div class="spinner"></div>
          <p>Processing data file...</p>
        </div>

        <div v-else-if="error" class="error-state glass">
          <p>{{ error }}</p>
        </div>

        <div v-else class="results-container">
          <div class="table-wrapper glass">
            <table class="data-table">
              <thead>
                <tr>
                  <th v-for="col in visibleColumns" :key="col.key">
                    {{ col.label }}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="task in paginatedTasks" :key="task['Task ID']">
                  <td class="id-col">
                    <a 
                      v-if="task['Task ID']"
                      :href="'https://app.asana.com/0/0/' + task['Task ID']" 
                      target="_blank" 
                      class="asana-link"
                    >
                      #{{ task['Task ID'].slice(-6) }}
                    </a>
                    <span v-else>N/A</span>
                  </td>
                  <td class="name-col">
                    <div class="name-text">{{ task.Name || 'Untitled Task' }}</div>
                  </td>
                  <td>
                    <span 
                      class="badge" 
                      :style="{ backgroundColor: getStatusColor(task['Section/Column']) + '20', color: getStatusColor(task['Section/Column']) }"
                    >
                      {{ task['Section/Column'] || 'Unassigned' }}
                    </span>
                  </td>
                  <td class="assignee-col">
                    <div class="flex-center gap-2">
                      <User :size="14" class="text-dim" />
                      {{ task.Assignee || '-' }}
                    </div>
                  </td>
                  <td class="notes-col">
                    <div class="notes-text" v-if="task.Notes">
                      {{ task.Notes.length > 100 ? task.Notes.slice(0, 100) + '...' : task.Notes }}
                    </div>
                    <span v-else class="text-dim">-</span>
                  </td>
                  <td>{{ formatDate(task['Created At']) }}</td>
                  <td>{{ formatDate(task['Completed At']) }}</td>
                  <td class="tags-col">
                    <div v-if="task.Tags" class="tag-pill">
                      {{ task.Tags.split(',')[0] }}
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div v-if="paginatedTasks.length === 0" class="empty-state glass">
            <p>No tasks match your filters</p>
            <button @click="searchQuery = ''; selectedSections = []; selectedAssignee = ''; filterMonth = ''; filterDate = ''">Clear All Filters</button>
          </div>

          <nav class="pagination-nav" v-if="totalPages > 1">
            <button 
              @click="currentPage--" 
              :disabled="currentPage === 1"
              class="nav-btn"
            >
              <ChevronLeft :size="20" />
            </button>
            
            <div class="page-info">
              Page <span>{{ currentPage }}</span> of {{ totalPages }}
            </div>
            
            <button 
              @click="currentPage++" 
              :disabled="currentPage === totalPages"
              class="nav-btn"
            >
              <ChevronRight :size="20" />
            </button>
          </nav>
        </div>
      </template>
    </main>
  </div>
</template>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2.5rem;
}

.title-section {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.logo-box {
  background: var(--primary);
  padding: 0.75rem;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 15px rgba(99, 102, 241, 0.4);
}

.logo-icon {
  color: white;
}

h1 {
  font-size: 2.25rem;
  letter-spacing: -0.02em;
  font-weight: 700;
  background: linear-gradient(to right, #fff, #94a3b8);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.subtitle {
  color: var(--text-dim);
  font-size: 0.95rem;
  margin-top: 0.25rem;
}

.header-actions {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.5rem 1rem;
  border-radius: 12px;
}

.file-name {
  font-size: 0.85rem;
  color: #fff;
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.upload-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.85rem;
  font-weight: 600;
  transition: all 0.2s ease;
  border: 1px solid var(--border-color);
}

.upload-btn:hover {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.3);
}

.stats {
  display: flex;
  padding: 0.5rem 1rem;
  gap: 1rem;
  align-items: center;
}

.stat-item {
  display: flex;
  flex-direction: column;
  min-width: 60px;
}

.stat-value {
  font-family: 'Outfit', sans-serif;
  font-size: 1.1rem;
  font-weight: 700;
  color: white;
  line-height: 1;
}

.stat-label {
  font-size: 0.65rem;
  color: var(--text-dim);
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.stat-divider {
  width: 1px;
  height: 24px;
  background: var(--border-color);
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  margin-bottom: 2.5rem;
}

.p-6 { padding: 1.5rem; }

.search-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 1rem;
  color: var(--text-dim);
}

.search-input {
  padding-left: 3rem;
  background: rgba(15, 23, 42, 0.4);
}

.filters {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.section-filter {
  width: 100%;
}

.chip-container {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  padding: 0.25rem 0;
}

.chip {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid var(--border-color);
  color: var(--text-dim);
  padding: 0.4rem 1rem;
  border-radius: 20px;
  font-size: 0.8rem;
  font-weight: 500;
  transition: all 0.2s ease;
  min-width: unset;
}

.chip:hover {
  background: rgba(255, 255, 255, 0.1);
  border-color: rgba(255, 255, 255, 0.2);
}

.chip.active {
  background: var(--primary);
  color: white;
  border-color: var(--primary);
  box-shadow: 0 0 10px rgba(99, 102, 241, 0.3);
}

.filter-controls {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.filter-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  flex: 1;
  min-width: 200px;
}

.filter-group label {
  font-size: 0.75rem;
  font-weight: 600;
  color: var(--text-dim);
  display: flex;
  align-items: center;
  gap: 0.4rem;
  text-transform: uppercase;
}

select, .date-input {
  background: rgba(15, 23, 42, 0.6);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  padding: 0.6rem 2rem 0.6rem 1rem;
  color: white;
  cursor: pointer;
  appearance: none;
  font-family: inherit;
}

select {
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%2394a3b8' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m6 9 6 6 6-6'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 0.75rem center;
}

.date-input {
  padding-right: 1rem;
}

.date-input::-webkit-calendar-picker-indicator {
  filter: invert(1);
  opacity: 0.5;
  cursor: pointer;
}

.export-btn {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: var(--primary);
  border: none;
  font-weight: 600;
  padding: 0.75rem 1.25rem;
  box-shadow: 0 4px 15px rgba(99, 102, 241, 0.3);
  transition: all 0.3s ease;
  min-width: 180px;
  justify-content: center;
}

.btn-success {
  background: #10b981 !important;
  box-shadow: 0 4px 15px rgba(16, 185, 129, 0.3) !important;
}

.table-wrapper {
  overflow-x: auto;
  margin-bottom: 2rem;
  border-radius: 12px;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
  font-size: 0.9rem;
}

.data-table th {
  background: rgba(30, 41, 59, 0.9);
  padding: 1.25rem 1rem;
  font-weight: 600;
  color: var(--text-dim);
  text-transform: uppercase;
  font-size: 0.75rem;
  letter-spacing: 0.05em;
  position: sticky;
  top: 0;
  z-index: 10;
  border-bottom: 1px solid var(--border-color);
}

.data-table td {
  padding: 1rem;
  border-bottom: 1px solid var(--border-color);
  color: #e2e8f0;
}

.data-table tbody tr {
  transition: background-color 0.2s ease;
}

.data-table tbody tr:hover {
  background: rgba(255, 255, 255, 0.03);
}

.name-col { min-width: 250px; }
.name-text {
  font-weight: 500;
  color: #fff;
}

.notes-col { min-width: 300px; }
.notes-text {
  font-size: 0.85rem;
  color: var(--text-dim);
  line-height: 1.4;
}

.id-col {
  font-family: monospace;
  color: var(--text-dim);
}

.asana-link {
  color: var(--primary-hover);
  text-decoration: none;
  font-weight: 500;
  transition: all 0.2s ease;
}

.asana-link:hover {
  color: #fff;
  text-decoration: underline;
}

.tag-pill {
  background: rgba(99, 102, 241, 0.1);
  color: var(--primary-hover);
  padding: 0.25rem 0.6rem;
  border-radius: 6px;
  font-size: 0.75rem;
  display: inline-block;
}

.flex-center { display: flex; align-items: center; }
.gap-2 { gap: 0.5rem; }
.text-dim { color: var(--text-dim); }

.pagination-nav {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 2rem;
  margin-top: 3rem;
}

.nav-btn {
  background: var(--bg-card);
  border: 1px solid var(--border-color);
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
}

.nav-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.page-info {
  font-size: 0.9rem;
  color: var(--text-dim);
}

.page-info span {
  color: white;
  font-weight: 600;
}

.loading-state, .error-state, .empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 5rem;
  text-align: center;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(99, 102, 241, 0.1);
  border-top-color: var(--primary);
  border-radius: 50%;
  animation: s 1s linear infinite;
  margin-bottom: 1rem;
}

.landing-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 8rem 2rem;
  text-align: center;
  border-radius: 24px;
}

.landing-content {
  max-width: 500px;
}

.landing-icon-box {
  background: rgba(99, 102, 241, 0.1);
  width: 100px;
  height: 100px;
  border-radius: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 2rem;
  color: var(--primary);
}

.landing-state h2 {
  font-size: 2rem;
  font-weight: 700;
  margin-bottom: 1rem;
  color: white;
}

.landing-state p {
  color: var(--text-dim);
  margin-bottom: 2.5rem;
  line-height: 1.6;
}

.primary-upload-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.75rem;
  background: var(--primary);
  color: white;
  padding: 1rem 2rem;
  border-radius: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 10px 30px rgba(99, 102, 241, 0.3);
}

.primary-upload-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 15px 40px rgba(99, 102, 241, 0.4);
  background: var(--primary-hover);
}

@keyframes s { to { transform: rotate(360deg); } }

@media (max-width: 768px) {
  .header { flex-direction: column; align-items: flex-start; gap: 1.5rem; }
  .controls { grid-template-columns: 1fr; }
  .filters { flex-wrap: wrap; }
}
</style>
