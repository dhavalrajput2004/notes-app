<script setup>
import { ref, computed, onMounted, watch } from 'vue'

// State
const notes = ref([])
const currentNote = ref(null)
const searchQuery = ref('')
const showEditor = ref(false)
const editorTitle = ref('')
const editorContent = ref('')

// Load notes from localStorage on mount
onMounted(() => {
  const savedNotes = localStorage.getItem('notes')
  if (savedNotes) {
    notes.value = JSON.parse(savedNotes)
  }
})

// Save notes to localStorage whenever they change
watch(notes, (newNotes) => {
  localStorage.setItem('notes', JSON.stringify(newNotes))
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
  return date.toLocaleDateString('en-US', { 
    month: 'short', 
    day: 'numeric', 
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
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
</script>

<template>
  <div class="app">
    <!-- Header -->
    <header class="header">
      <h1>📝 Notes App</h1>
      <button @click="createNewNote" class="btn-primary">
        <span class="icon">+</span> New Note
      </button>
    </header>

    <!-- Search Bar -->
    <div class="search-container">
      <input 
        v-model="searchQuery" 
        type="text" 
        placeholder="🔍 Search notes..." 
        class="search-input"
      />
    </div>

    <!-- Main Content -->
    <div class="main-content">
      <!-- Notes List -->
      <div class="notes-list" v-if="!showEditor">
        <div v-if="filteredNotes.length === 0" class="empty-state">
          <p v-if="searchQuery">No notes found matching "{{ searchQuery }}"</p>
          <p v-else>No notes yet. Click "New Note" to create your first note!</p>
        </div>
        
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
            >
              🗑️
            </button>
          </div>
          <p class="note-preview">{{ note.content.substring(0, 150) }}{{ note.content.length > 150 ? '...' : '' }}</p>
          <div class="note-footer">
            <span class="note-date">{{ formatDate(note.updatedAt) }}</span>
          </div>
        </div>
      </div>

      <!-- Editor -->
      <div v-if="showEditor" class="editor">
        <div class="editor-header">
          <button @click="closeEditor" class="btn-back">← Back</button>
          <button @click="saveNote" class="btn-save">💾 Save</button>
        </div>

        <input 
          v-model="editorTitle" 
          type="text" 
          placeholder="Note Title" 
          class="editor-title"
          autofocus
        />

        <!-- Formatting Toolbar -->
        <div class="toolbar">
          <button @click="applyFormat('bold')" class="toolbar-btn" title="Bold">
            <strong>B</strong>
          </button>
          <button @click="applyFormat('italic')" class="toolbar-btn" title="Italic">
            <em>I</em>
          </button>
          <button @click="applyFormat('heading')" class="toolbar-btn" title="Heading">
            H
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
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  padding-bottom: 20px;
  border-bottom: 2px solid #e0e0e0;
}

.header h1 {
  margin: 0;
  font-size: 2rem;
  color: #333;
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
  transition: transform 0.2s, box-shadow 0.2s;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.icon {
  font-size: 1.5rem;
  line-height: 1;
}

.search-container {
  margin-bottom: 30px;
}

.search-input {
  width: 100%;
  padding: 15px 20px;
  font-size: 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  transition: border-color 0.3s;
  box-sizing: border-box;
}

.search-input:focus {
  outline: none;
  border-color: #667eea;
}

.main-content {
  min-height: 400px;
}

.notes-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.empty-state {
  grid-column: 1 / -1;
  text-align: center;
  padding: 60px 20px;
  color: #999;
  font-size: 1.1rem;
}

.note-card {
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  padding: 20px;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s, border-color 0.2s;
}

.note-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.1);
  border-color: #667eea;
}

.note-header {
  display: flex;
  justify-content: space-between;
  align-items: start;
  margin-bottom: 12px;
}

.note-title {
  margin: 0;
  font-size: 1.3rem;
  color: #333;
  flex: 1;
  word-break: break-word;
}

.btn-delete {
  background: none;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 4px;
  opacity: 0.6;
  transition: opacity 0.2s, transform 0.2s;
}

.btn-delete:hover {
  opacity: 1;
  transform: scale(1.2);
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
}

.note-date {
  font-size: 0.85rem;
  color: #999;
}

.editor {
  background: white;
  border: 2px solid #e0e0e0;
  border-radius: 12px;
  padding: 30px;
  max-width: 800px;
  margin: 0 auto;
}

.editor-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
}

.btn-back, .btn-save {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
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
}

.btn-save:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.editor-title {
  width: 100%;
  padding: 15px;
  font-size: 1.8rem;
  font-weight: 600;
  border: none;
  border-bottom: 2px solid #e0e0e0;
  margin-bottom: 20px;
  box-sizing: border-box;
}

.editor-title:focus {
  outline: none;
  border-bottom-color: #667eea;
}

.toolbar {
  display: flex;
  gap: 8px;
  margin-bottom: 15px;
  padding: 10px;
  background: #f8f8f8;
  border-radius: 8px;
}

.toolbar-btn {
  padding: 8px 12px;
  background: white;
  border: 1px solid #ddd;
  border-radius: 6px;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.2s, border-color 0.2s;
}

.toolbar-btn:hover {
  background: #f0f0f0;
  border-color: #667eea;
}

.editor-content {
  width: 100%;
  min-height: 400px;
  padding: 20px;
  font-size: 1rem;
  line-height: 1.8;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  resize: vertical;
  font-family: inherit;
  box-sizing: border-box;
}

.editor-content:focus {
  outline: none;
  border-color: #667eea;
}

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
  
  .search-input {
    background: #2a2a2a;
    border-color: #444;
    color: #e0e0e0;
  }
  
  .note-card {
    background: #2a2a2a;
    border-color: #444;
  }
  
  .note-title {
    color: #e0e0e0;
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
}

@media (max-width: 768px) {
  .app {
    padding: 10px;
  }
  
  .header {
    flex-direction: column;
    gap: 15px;
    align-items: stretch;
  }
  
  .notes-list {
    grid-template-columns: 1fr;
  }
  
  .editor {
    padding: 15px;
  }
}
</style>
