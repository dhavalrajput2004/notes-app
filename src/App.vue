<script setup>
import { ref, computed, onMounted, watch, onUnmounted } from 'vue'

// State
const notes = ref([])
const currentNote = ref(null)
const searchQuery = ref('')
const showEditor = ref(false)
const editorTitle = ref('')
const editorContent = ref('')
const showSaveIndicator = ref(false)
const viewMode = ref('grid') // 'grid' or 'list'

// Load notes from localStorage on mount
onMounted(() => {
  const savedNotes = localStorage.getItem('notes')
  if (savedNotes) {
    notes.value = JSON.parse(savedNotes)
  }
  
  // Add keyboard shortcuts
  document.addEventListener('keydown', handleKeyboardShortcuts)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeyboardShortcuts)
})

// Keyboard shortcuts
const handleKeyboardShortcuts = (e) => {
  // Alt + N for new note (avoiding Ctrl+N browser conflict)
  if (e.altKey && e.key === 'n' && !showEditor.value) {
    e.preventDefault()
    createNewNote()
  }
  // Ctrl/Cmd + S for save
  if ((e.ctrlKey || e.metaKey) && e.key === 's' && showEditor.value) {
    e.preventDefault()
    saveNote()
  }
  // Escape to close editor
  if (e.key === 'Escape' && showEditor.value) {
    closeEditor()
  }
}

// Save notes to localStorage whenever they change
let isInitialLoad = true
watch(notes, (newNotes) => {
  localStorage.setItem('notes', JSON.stringify(newNotes))
  // Don't show save indicator on initial load
  if (!isInitialLoad) {
    showSaveIndicator.value = true
    setTimeout(() => {
      showSaveIndicator.value = false
    }, 1500)
  }
  isInitialLoad = false
}, { deep: true })

// Computed properties
const filteredNotes = computed(() => {
  if (!searchQuery.value) return notes.value
  const query = searchQuery.value.toLowerCase()
  return notes.value.filter(note => 
    note.title.toLowerCase().includes(query) || 
    note.content.toLowerCase().includes(query)
  )
})

const notesCount = computed(() => notes.value.length)

// Methods
const createNewNote = () => {
  editorTitle.value = ''
  editorContent.value = ''
  currentNote.value = null
  showEditor.value = true
}

const editNote = (note) => {
  currentNote.value = note
  editorTitle.value = note.title
  editorContent.value = note.content
  showEditor.value = true
}

const saveNote = () => {
  if (!editorTitle.value.trim()) {
    alert('Please enter a title')
    return
  }

  if (currentNote.value) {
    // Update existing note
    currentNote.value.title = editorTitle.value
    currentNote.value.content = editorContent.value
    currentNote.value.updatedAt = new Date().toISOString()
  } else {
    // Create new note
    const newNote = {
      id: Date.now(),
      title: editorTitle.value,
      content: editorContent.value,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    }
    notes.value.unshift(newNote)
  }

  closeEditor()
}

const deleteNote = (noteId) => {
  if (confirm('Are you sure you want to delete this note?')) {
    notes.value = notes.value.filter(note => note.id !== noteId)
    if (currentNote.value?.id === noteId) {
      closeEditor()
    }
  }
}

const closeEditor = () => {
  showEditor.value = false
  currentNote.value = null
  editorTitle.value = ''
  editorContent.value = ''
}

const formatDate = (dateString) => {
  const date = new Date(dateString)
  const now = new Date()
  const diffMs = now - date
  const diffMins = Math.floor(diffMs / 60000)
  const diffHours = Math.floor(diffMs / 3600000)
  const diffDays = Math.floor(diffMs / 86400000)
  
  if (diffMins < 1) return 'Just now'
  if (diffMins < 60) return `${diffMins}m ago`
  if (diffHours < 24) return `${diffHours}h ago`
  if (diffDays < 7) return `${diffDays}d ago`
  
  return date.toLocaleDateString('en-US', { 
    month: 'short', 
    day: 'numeric', 
    year: date.getFullYear() !== now.getFullYear() ? 'numeric' : undefined
  })
}

const getWordCount = (text) => {
  return text.trim().split(/\s+/).filter(word => word.length > 0).length
}

// Text formatting functions
const applyFormat = (format) => {
  const textarea = document.querySelector('.editor-content')
  const start = textarea.selectionStart
  const end = textarea.selectionEnd
  const selectedText = editorContent.value.substring(start, end)
  
  let formattedText = ''
  
  switch(format) {
    case 'bold':
      formattedText = `**${selectedText}**`
      break
    case 'italic':
      formattedText = `*${selectedText}*`
      break
    case 'heading':
      formattedText = `# ${selectedText}`
      break
    case 'bullet':
      formattedText = `• ${selectedText}`
      break
  }
  
  editorContent.value = 
    editorContent.value.substring(0, start) + 
    formattedText + 
    editorContent.value.substring(end)
}

const toggleViewMode = () => {
  viewMode.value = viewMode.value === 'grid' ? 'list' : 'grid'
}
</script>

<template>
  <div class="app">
    <!-- Save Indicator -->
    <transition name="fade">
      <div v-if="showSaveIndicator" class="save-indicator">
        ✓ Saved
      </div>
    </transition>

    <!-- Header -->
    <header class="header">
      <div class="header-left">
        <h1>📝 Notes App</h1>
        <span v-if="notesCount > 0" class="notes-count">{{ notesCount }} {{ notesCount === 1 ? 'note' : 'notes' }}</span>
      </div>
      <div class="header-right">
        <button 
          v-if="!showEditor && notesCount > 0" 
          @click="toggleViewMode" 
          class="btn-view-toggle"
          :title="`Switch to ${viewMode === 'grid' ? 'list' : 'grid'} view`"
        >
          {{ viewMode === 'grid' ? '☰' : '⊞' }}
        </button>
        <button @click="createNewNote" class="btn-primary" title="Create new note (Alt+N)">
          <span class="icon">+</span> New Note
        </button>
      </div>
    </header>

    <!-- Search Bar -->
    <div class="search-container" v-if="!showEditor && notesCount > 0">
      <div class="search-wrapper">
        <span class="search-icon">🔍</span>
        <input 
          v-model="searchQuery" 
          type="text" 
          placeholder="Search notes..." 
          class="search-input"
        />
        <button v-if="searchQuery" @click="searchQuery = ''" class="clear-search">✕</button>
      </div>
    </div>

    <!-- Main Content -->
    <div class="main-content">
      <!-- Notes List -->
      <div class="notes-list" :class="viewMode" v-if="!showEditor">
        <div v-if="filteredNotes.length === 0" class="empty-state">
          <div class="empty-icon">📝</div>
          <h2 v-if="searchQuery">No notes found</h2>
          <p v-if="searchQuery">Try adjusting your search terms</p>
          <p v-else>Create your first note and start organizing your thoughts</p>
          <button v-if="!searchQuery" @click="createNewNote" class="btn-primary btn-large">
            <span class="icon">+</span> Create First Note
          </button>
        </div>
        
        <transition-group name="note-list" tag="div" class="notes-grid">
          <div 
            v-for="note in filteredNotes" 
            :key="note.id" 
            class="note-card"
            @click="editNote(note)"
          >
            <div class="note-header">
              <h3 class="note-title">{{ note.title }}</h3>
              <button 
                @click.stop="deleteNote(note.id)" 
                class="btn-delete"
                title="Delete note"
                aria-label="Delete note"
              >
                🗑️
              </button>
            </div>
            <p class="note-preview">{{ note.content.substring(0, 150) }}{{ note.content.length > 150 ? '...' : '' }}</p>
            <div class="note-footer">
              <span class="note-date">{{ formatDate(note.updatedAt) }}</span>
              <span class="note-word-count">{{ getWordCount(note.content) }} words</span>
            </div>
          </div>
        </transition-group>
      </div>

      <!-- Editor -->
      <transition name="slide-fade">
        <div v-if="showEditor" class="editor">
          <div class="editor-header">
            <button @click="closeEditor" class="btn-back" title="Close editor (Esc)">
              ← Back
            </button>
            <div class="editor-info">
              <span v-if="editorContent" class="word-count">{{ getWordCount(editorContent) }} words</span>
            </div>
            <button @click="saveNote" class="btn-save" title="Save note (Ctrl+S)">
              💾 Save
            </button>
          </div>

          <input 
            v-model="editorTitle" 
            type="text" 
            placeholder="Untitled Note" 
            class="editor-title"
            autofocus
          />

          <!-- Formatting Toolbar -->
          <div class="toolbar">
            <button @click="applyFormat('bold')" class="toolbar-btn" title="Bold (Ctrl+B)">
              <strong>B</strong>
            </button>
            <button @click="applyFormat('italic')" class="toolbar-btn" title="Italic (Ctrl+I)">
              <em>I</em>
            </button>
            <button @click="applyFormat('heading')" class="toolbar-btn" title="Heading">
              <strong>H</strong>
            </button>
            <button @click="applyFormat('bullet')" class="toolbar-btn" title="Bullet List">
              •
            </button>
          </div>

          <textarea 
            v-model="editorContent" 
            placeholder="Start writing your note..." 
            class="editor-content"
          ></textarea>
        </div>
      </transition>
    </div>

    <!-- Keyboard Shortcuts Hint -->
    <div class="shortcuts-hint" v-if="!showEditor">
      <kbd>Alt</kbd> + <kbd>N</kbd> New note
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
  position: relative;
}

/* Save Indicator */
.save-indicator {
  position: fixed;
  top: 20px;
  right: 20px;
  background: #10b981;
  color: white;
  padding: 12px 20px;
  border-radius: 8px;
  font-weight: 600;
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.3);
  z-index: 1000;
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s, transform 0.3s;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}

/* Header */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  padding-bottom: 20px;
  border-bottom: 2px solid #e0e0e0;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 15px;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 10px;
}

.header h1 {
  margin: 0;
  font-size: 2rem;
  color: #333;
  font-weight: 700;
}

.notes-count {
  background: #f0f0f0;
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 0.85rem;
  color: #666;
  font-weight: 500;
}

.btn-view-toggle {
  background: #f0f0f0;
  color: #333;
  border: none;
  padding: 10px 16px;
  border-radius: 8px;
  font-size: 1.2rem;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-view-toggle:hover {
  background: #e0e0e0;
  transform: translateY(-2px);
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  transition: all 0.2s;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.2);
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.4);
}

.btn-primary:active {
  transform: translateY(0);
}

.btn-large {
  padding: 16px 32px;
  font-size: 1.1rem;
  margin-top: 20px;
}

.icon {
  font-size: 1.5rem;
  line-height: 1;
}

/* Search */
.search-container {
  margin-bottom: 30px;
}

.search-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 20px;
  font-size: 1.2rem;
  pointer-events: none;
}

.search-input {
  width: 100%;
  padding: 15px 20px 15px 50px;
  font-size: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  transition: all 0.3s;
  box-sizing: border-box;
  background: white;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.clear-search {
  position: absolute;
  right: 15px;
  background: none;
  border: none;
  font-size: 1.2rem;
  color: #999;
  cursor: pointer;
  padding: 5px;
  border-radius: 50%;
  transition: all 0.2s;
}

.clear-search:hover {
  background: #f0f0f0;
  color: #333;
}

/* Main Content */
.main-content {
  min-height: 400px;
}

/* Notes List */
.notes-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
}

.notes-list.list .notes-grid {
  grid-template-columns: 1fr;
}

.notes-list.list .note-card {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 20px;
}

.notes-list.list .note-header {
  flex: 0 0 200px;
  margin-bottom: 0;
}

.notes-list.list .note-preview {
  flex: 1;
  margin: 0;
}

.notes-list.list .note-footer {
  flex: 0 0 150px;
  border-top: none;
  padding-top: 0;
  flex-direction: column;
  align-items: flex-end;
}

/* Empty State */
.empty-state {
  grid-column: 1 / -1;
  text-align: center;
  padding: 80px 20px;
  color: #999;
}

.empty-icon {
  font-size: 4rem;
  margin-bottom: 20px;
  opacity: 0.5;
}

.empty-state h2 {
  color: #333;
  margin-bottom: 10px;
  font-size: 1.8rem;
}

.empty-state p {
  font-size: 1.1rem;
  margin-bottom: 10px;
}

/* Note Card */
.note-card {
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  padding: 20px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.note-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 3px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  transform: scaleX(0);
  transition: transform 0.3s;
}

.note-card:hover::before {
  transform: scaleX(1);
}

.note-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.1);
  border-color: #667eea;
}

.note-list-enter-active, .note-list-leave-active {
  transition: all 0.3s;
}

.note-list-enter-from {
  opacity: 0;
  transform: translateY(20px);
}

.note-list-leave-to {
  opacity: 0;
  transform: scale(0.9);
}

.note-header {
  display: flex;
  justify-content: space-between;
  align-items: start;
  margin-bottom: 12px;
  gap: 10px;
}

.note-title {
  margin: 0;
  font-size: 1.3rem;
  color: #333;
  flex: 1;
  word-break: break-word;
  font-weight: 600;
}

.btn-delete {
  background: none;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 4px 8px;
  opacity: 0;
  transition: all 0.2s;
  border-radius: 4px;
}

.note-card:hover .btn-delete {
  opacity: 0.6;
}

.btn-delete:hover {
  opacity: 1 !important;
  background: #fee;
  transform: scale(1.1);
}

.note-preview {
  color: #666;
  margin: 0 0 12px 0;
  line-height: 1.6;
  word-break: break-word;
}

.note-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 12px;
  border-top: 1px solid #f0f0f0;
  gap: 10px;
}

.note-date {
  font-size: 0.85rem;
  color: #999;
  font-weight: 500;
}

.note-word-count {
  font-size: 0.85rem;
  color: #999;
  background: #f8f8f8;
  padding: 4px 8px;
  border-radius: 4px;
}

/* Editor */
.editor {
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 16px;
  padding: 25px;
  max-width: 750px;
  margin: 0 auto;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.08);
}

.slide-fade-enter-active {
  transition: all 0.3s ease-out;
}

.slide-fade-leave-active {
  transition: all 0.2s ease-in;
}

.slide-fade-enter-from {
  transform: translateX(20px);
  opacity: 0;
}

.slide-fade-leave-to {
  transform: translateX(-20px);
  opacity: 0;
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.editor-info {
  flex: 1;
  text-align: center;
}

.word-count {
  font-size: 0.9rem;
  color: #999;
  background: #f8f8f8;
  padding: 6px 12px;
  border-radius: 6px;
}

.btn-back, .btn-save {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-back {
  background: #f0f0f0;
  color: #333;
}

.btn-back:hover {
  background: #e0e0e0;
  transform: translateY(-2px);
}

.btn-save {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.2);
}

.btn-save:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 16px rgba(102, 126, 234, 0.4);
}

.editor-title {
  width: 100%;
  padding: 12px 0;
  font-size: 1.75rem;
  font-weight: 700;
  border: none;
  border-bottom: 3px solid #e0e0e0;
  margin-bottom: 18px;
  box-sizing: border-box;
  transition: border-color 0.3s;
}

.editor-title:focus {
  outline: none;
  border-bottom-color: #667eea;
}

.toolbar {
  display: flex;
  gap: 6px;
  margin-bottom: 12px;
  padding: 10px;
  background: #f8f8f8;
  border-radius: 10px;
  flex-wrap: wrap;
}

.toolbar-btn {
  padding: 8px 12px;
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.95rem;
  transition: all 0.2s;
  font-weight: 600;
}

.toolbar-btn:hover {
  background: white;
  border-color: #667eea;
  transform: translateY(-2px);
  box-shadow: 0 2px 8px rgba(102, 126, 234, 0.2);
}

.toolbar-btn:active {
  transform: translateY(0);
}

.editor-content {
  width: 100%;
  min-height: 350px;
  padding: 18px;
  font-size: 1rem;
  line-height: 1.7;
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  resize: vertical;
  font-family: inherit;
  box-sizing: border-box;
  transition: all 0.3s;
}

.editor-content:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

/* Keyboard Shortcuts Hint */
.shortcuts-hint {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 10px 16px;
  border-radius: 8px;
  font-size: 0.85rem;
  backdrop-filter: blur(10px);
  display: flex;
  align-items: center;
  gap: 6px;
}

.shortcuts-hint kbd {
  background: rgba(255, 255, 255, 0.2);
  padding: 4px 8px;
  border-radius: 4px;
  font-family: monospace;
  font-size: 0.8rem;
  border: 1px solid rgba(255, 255, 255, 0.3);
}

/* Dark Mode */
@media (prefers-color-scheme: dark) {
  .app {
    color: #e0e0e0;
  }
  
  .header h1 {
    color: #e0e0e0;
  }
  
  .header {
    border-bottom-color: #444;
  }
  
  .notes-count {
    background: #333;
    color: #999;
  }
  
  .btn-view-toggle {
    background: #333;
    color: #e0e0e0;
  }
  
  .btn-view-toggle:hover {
    background: #444;
  }
  
  .search-input {
    background: #2a2a2a;
    border-color: #444;
    color: #e0e0e0;
  }
  
  .empty-state h2 {
    color: #e0e0e0;
  }
  
  .note-card {
    background: #2a2a2a;
    border-color: #444;
  }
  
  .note-title {
    color: #e0e0e0;
  }
  
  .note-word-count {
    background: #333;
  }
  
  .editor {
    background: #2a2a2a;
    border-color: #444;
  }
  
  .editor-title {
    background: #2a2a2a;
    color: #e0e0e0;
    border-bottom-color: #444;
  }
  
  .editor-content {
    background: #2a2a2a;
    color: #e0e0e0;
    border-color: #444;
  }
  
  .toolbar {
    background: #333;
  }
  
  .toolbar-btn {
    background: #2a2a2a;
    border-color: #444;
    color: #e0e0e0;
  }
  
  .btn-back {
    background: #333;
    color: #e0e0e0;
  }
  
  .btn-back:hover {
    background: #444;
  }
  
  .word-count {
    background: #333;
  }
  
  .clear-search:hover {
    background: #333;
  }
}

/* Responsive Design */
@media (max-width: 768px) {
  .app {
    padding: 15px;
  }
  
  .header {
    flex-direction: column;
    gap: 15px;
    align-items: stretch;
  }
  
  .header-left {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }
  
  .header-right {
    width: 100%;
    justify-content: flex-end;
    gap: 10px;
  }
  
  .notes-grid {
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  }
  
  .notes-list.list .note-card {
    flex-direction: column;
    align-items: stretch;
  }
  
  .notes-list.list .note-header {
    flex: 1;
    margin-bottom: 12px;
  }
  
  .notes-list.list .note-preview {
    margin-bottom: 12px;
  }
  
  .notes-list.list .note-footer {
    flex: 1;
    flex-direction: row;
    align-items: center;
    border-top: 1px solid #f0f0f0;
    padding-top: 12px;
  }
  
  .editor {
    padding: 18px;
    border-radius: 12px;
    max-width: 100%;
  }
  
  .editor-header {
    flex-wrap: wrap;
    gap: 10px;
  }
  
  .editor-info {
    order: 3;
    width: 100%;
    text-align: left;
  }
  
  .editor-title {
    font-size: 1.5rem;
  }
  
  .shortcuts-hint {
    display: none;
  }
  
  .empty-icon {
    font-size: 3rem;
  }
  
  .empty-state h2 {
    font-size: 1.5rem;
  }
  
  .empty-state p {
    font-size: 1rem;
  }
}

@media (max-width: 480px) {
  .header h1 {
    font-size: 1.5rem;
  }
  
  .notes-count {
    font-size: 0.8rem;
    padding: 4px 10px;
  }
  
  .note-card {
    padding: 15px;
  }
  
  .note-title {
    font-size: 1.1rem;
  }
  
  .editor {
    padding: 15px;
  }
  
  .editor-title {
    font-size: 1.3rem;
  }
  
  .editor-content {
    min-height: 250px;
    font-size: 0.95rem;
    padding: 15px;
  }
}
</style>
