<script setup>
  const todos = ref([])
  const newTodo = ref('')
  const newTags = ref('')
  const selectedTag = ref('')
  const statusFilter = ref('active')

  const config = ref({
    soundEnabled: false,
  })

  const toggleSound = () => {
    config.value.soundEnabled = !config.value.soundEnabled
    localStorage.setItem('config', JSON.stringify(config.value))
  }

  const tagFrequencies = computed(() => {
    const frequencies = {}
    todos.value.forEach(todo => {
      todo.tags.forEach(tag => {
        frequencies[tag] = (frequencies[tag] || 0) + 1
      })
    })
    return Object.entries(frequencies)
      .sort((a, b) => b[1] - a[1])
      .map(([tag]) => tag)
  })

  const getTodosByStatus = (status) => {
    return todos.value.filter(todo => {
      if (status === 'active') return !todo.completed
      if (status === 'completed') return todo.completed
      return true
    })
  }

  const selectTag = (tag) => {
    if (tag === selectedTag.value) {
      selectedTag.value = ''
    } else {
      selectedTag.value = tag
    }
  }

  const toggleStatusFilter = () => {
    statusFilter.value = statusFilter.value === 'active' ? 'completed' : 'active'
  }

  const addTagToNew = (tag) => {
    const currentTags = newTags.value.split(',').map(t => t.trim()).filter(t => t)
    if (!currentTags.includes(tag)) {
      newTags.value = currentTags.length ?
        `${newTags.value}, ${tag}` :
        tag
    }
  }

  const addTodo = async () => {
    if (newTodo.value.trim()) {
      const tags = newTags.value.split(',')
        .map(tag => tag.trim())
        .filter(tag => tag.length > 0)

      todos.value.push({
        id: Date.now(),
        text: newTodo.value,
        tags: tags,
        completed: false
      })
      newTodo.value = ''
      newTags.value = ''

      if(config.value.soundEnabled){
        try {
          const audio = new Audio('/new.mp3')
          await audio.play()
        } catch (error) {
          console.error('Failed to play sound:', error)
        }
      }

      saveTodos()
    }
  }

  const removeTodo = (id) => {
    todos.value = todos.value.filter(todo => todo.id !== id)
    saveTodos()
  }

  const toggleTodo = async (id) => {
    const todo = todos.value.find(todo => todo.id === id)
    todo.completed = !todo.completed

    if(config.value.soundEnabled){
      if (todo.completed) {
        try {
          const audio = new Audio('/complete.mp3')
          await audio.play()
        } catch (error) {
          console.error('Failed to play sound:', error)
        }
      }else{
        try {
          const audio = new Audio('/new.mp3')
          await audio.play()
        } catch (error) {
          console.error('Failed to play sound:', error)
        }
      }
    }

    saveTodos()
  }

  const saveTodos = () => {
    localStorage.setItem('todos', JSON.stringify(todos.value))
  }

  onMounted(() => {
    const savedTodos = localStorage.getItem('todos')
    if (savedTodos) {
      todos.value = JSON.parse(savedTodos)
    }

    const savedConfig = localStorage.getItem('config')
    if (savedConfig) {
      config.value = JSON.parse(savedConfig)
    }
  })
</script>

<template>
  <main>
    <div class="todo-container">
      <div class="todo-list-column">
        <div class="filters">
          <div class="filters-row">
            <button @click="toggleStatusFilter" :class="['status-toggle', { active: statusFilter === 'completed' }]">
              Show {{ statusFilter === 'active' ? 'Completed' : 'Active' }}
            </button>
            <div class="separator"></div>
            <div class="tag-filters">
              <span v-for="tag in tagFrequencies" :key="tag" :class="['filter-tag', { active: selectedTag === tag }]"
                @click="selectTag(tag)">
                {{ tag }}
              </span>
            </div>
          </div>
        </div>
        <div class="list-container">
          <Transition name="category" mode="out-in">
            <div v-if="statusFilter == 'active'">
              <TransitionGroup name="active" tag="ul" v-if="getTodosByStatus('active')" class="todo-list">
                <li v-for="todo in getTodosByStatus('active').filter(i=>i.tags.includes(selectedTag) || !selectedTag)"
                  :key="todo.id" :class="['todo-item', { 'todo-completed': todo.completed }]">
                  <div class="todo-item-top">
                    <div class="tags-container">
                      <span v-for="tag in todo.tags" :key="tag" class="tag">
                        {{ tag }}
                      </span>
                    </div>
                    <button @click="removeTodo(todo.id)" class="delete-btn">×</button>
                  </div>
                  <div class="todo-main">
                    <label class="custom-checkbox">
                      <input type="checkbox" :checked="todo.completed" @change="toggleTodo(todo.id)"
                        class="hidden-checkbox">
                      <div class="checkbox-display"></div>
                    </label>
                    <span :class="{ completed: todo.completed }" class="todo-text">{{ todo.text }}</span>
                  </div>
                </li>
              </TransitionGroup>
              <p v-else class="empty-message">No todos</p>
            </div>
            <div v-else>
              <TransitionGroup name="completed" tag="ul" v-if="getTodosByStatus('completed')" class="todo-list">
                <li v-for="todo in getTodosByStatus('completed')" :key="todo.id"
                  :class="['todo-item', { 'todo-completed': todo.completed }]">
                  <div class="todo-item-top">
                    <div class="tags-container">
                      <span v-for="tag in todo.tags" :key="tag" class="tag">
                        {{ tag }}
                      </span>
                    </div>
                    <button @click="removeTodo(todo.id)" class="delete-btn">×</button>
                  </div>
                  <div class="todo-main">
                    <label class="custom-checkbox">
                      <input type="checkbox" :checked="todo.completed" @change="toggleTodo(todo.id)"
                        class="hidden-checkbox">
                      <div class="checkbox-display"></div>
                    </label>
                    <span :class="{ completed: todo.completed }" class="todo-text">{{ todo.text }}</span>
                  </div>
                </li>
              </TransitionGroup>
              <p v-else class="empty-message">No todos</p>
            </div>
          </Transition>
        </div>
      </div>
      <div class="input-column">
        <div class="input-container">
          <TransitionGroup name="form">
            <div v-if="tagFrequencies.length" class="quick-tags">
              <span v-for="tag in tagFrequencies" :key="tag" class="quick-tag" @click="addTagToNew(tag)">
                {{ tag }}
              </span>
            </div>
            <input v-model="newTags" placeholder="Add tags (comma-separated)" type="text">
            <textarea v-model="newTodo" @keyup.enter="addTodo" placeholder="Add new todo"></textarea>
            <button @click="addTodo">Add</button>
            <div class="options">
              <div>
                <p>
                  Autogun by <a href="https://github.com/hubret">hubret</a> CC0
                </p>
              </div>
              <label class="custom-checkbox" style="display: flex; gap: 10px; align-items: center;">
                <input type="checkbox" :checked="config.soundEnabled" @change="toggleSound()" class="hidden-checkbox">
                <div class="checkbox-display"></div>
                <span>Enable Sound</span>
              </label>
            </div>
          </TransitionGroup>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
  .hidden-checkbox {
    position: absolute;
    opacity: 0;
    cursor: pointer;
  }

  .custom-checkbox {
    position: relative;
    cursor: pointer;
    display: inline-block;
  }

  .checkbox-display {
    width: 24px;
    height: 24px;
    background: transparent;
    border: 2px solid var(--border-color);
    border-radius: 4px;
    position: relative;
    transition: all 0.5s ease;
  }

  .checkbox-display:after {
    content: '';
    position: absolute;
    display: block;
    left: 7px;
    top: 3px;
    width: 6px;
    height: 12px;
    border: solid white;
    border-width: 0 2px 2px 0;
    transform: rotate(45deg);
    opacity: 0;
    transition: all 0.5s ease;
  }

  .hidden-checkbox:checked+.checkbox-display {
    background: #444;
    border-color: #444;
  }

  .hidden-checkbox:checked+.checkbox-display:after {
    opacity: 1;
  }

  .active-move,
  .active-enter-active,
  .active-leave-active {
    transition: all 10s ease;
  }

  .active-leave-active {
    position: absolute;
    width: 100%;
    box-sizing: border-box;
  }

  .active-enter-from {
    opacity: 0;
    transform: translate(30px);
    filter: blur(5px);
  }

  .active-leave-to {
    opacity: 0;
    filter: blur(20px);
    transform: scale(1.2);
  }

  .completed-move,
  .completed-enter-active,
  .completed-leave-active {
    transition: all 10s ease;
  }

  .completed-leave-active {
    position: absolute;
    width: 100%;
    box-sizing: border-box;
  }

  .completed-enter-from {
    opacity: 0;
    transform: translateY(30px);
    filter: blur(5px);
  }

  .completed-leave-to {
    opacity: 0;
    filter: blur(5px);
    transform: scale(0.9);
  }

  .form-move,
  .form-enter-active,
  .form-leave-active {
    transition: all 0.5s ease;
  }

  .form-leave-active {
    position: absolute;
    width: 100%;
    box-sizing: border-box;
  }

  .form-enter-from {
    opacity: 0;
    transform: translateY(30px);
    filter: blur(5px);
  }

  .form-leave-to {
    opacity: 0;
    filter: blur(5px);
    transform: scale(0.9);
  }

  .list-container {
    position: relative;
    width: 100%;
  }

  .category-enter-active,
  .category-leave-active {
    transition: all 0.25s ease;
    position: absolute;
    width: 100%;
  }

  .category-enter-from {
    opacity: 0;
    filter: blur(5px);
    translate: -50px;
  }

  .category-leave-to {
    opacity: 0;
    filter: blur(5px);
    translate: 50px;
  }

  .filters {
    margin-bottom: 20px;
  }

  .filters-row {
    display: flex;
    align-items: center;
    gap: 15px;
  }

  .separator {
    width: 1px;
    height: 24px;
    background-color: var(--border-color);
  }

  .status-toggle {
    white-space: nowrap;
    padding: 6px 12px;
    background-color: #2a2a2a;
    border: 1px solid var(--border-color);
    color: var(--text-color);
    border-radius: 4px;
    transition: all 0.2s;
  }

  .status-toggle.active {
    background-color: #444;
    color: white;
  }

  .tag-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
  }

  .filter-tag {
    background-color: #2a2a2a;
    color: var(--text-color);
    padding: 4px 12px;
    border-radius: 12px;
    font-size: 0.9em;
    cursor: pointer;
    border: 1px solid var(--border-color);
    transition: all 0.2s;
  }

  .filter-tag.active {
    background-color: var(--accent-color);
    color: white;
  }

  .todo-container {
    display: grid;
    grid-template-columns: 3fr 2fr;
    gap: 20px;
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    box-sizing: border-box;
    min-height: 100vh;
  }

  .todo-list-column {
    padding-right: 20px;
    border-right: 1px solid var(--border-color);
  }

  .empty-message {
    text-align: center;
    color: #888;
    font-style: italic;
    margin-top: 20px;
  }

  .input-column {
    padding-left: 20px;
  }

  .input-container {
    height: 100%;
    display: flex;
    flex-direction: column;
    gap: 10px;
    position: sticky;
    top: 20px;
  }

  .quick-tags {
    display: flex;
    flex-wrap: wrap;
    align-items: flex-start;
    gap: 8px;
    height: 28px;
  }

  .quick-tag {
    background-color: #2a2a2a;
    color: var(--text-color);
    padding: 4px 12px;
    border-radius: 12px;
    font-size: 0.9em;
    cursor: pointer;
    border: 1px solid var(--border-color);
    transition: all 0.2s;
  }

  .quick-tag:hover {
    background-color: #444;
    color: white;
  }

  .options {
    opacity: 0.3;
    margin-top: auto;
  }

  input[type="text"],
  textarea {
    padding: 8px;
    background-color: #2a2a2a;
    border: 1px solid var(--border-color);
    color: var(--text-color);
    border-radius: 4px;
  }

  input[type="text"]::placeholder,
  textarea::placeholder {
    color: #666;
  }

  textarea {
    min-height: 100px;
    resize: vertical;
  }

  .todo-list {
    list-style: none;
    padding: 0;
    margin: 0;
    position: relative;
    width: 100%;
  }

  .todo-item {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 15px;
    margin-bottom: 15px;
    background-color: #2a2a2a;
    border-radius: 8px;
    border: 1px solid var(--border-color);
    transition: all 0.5s ease;
  }

  .todo-completed {
    background-color: #1a1a1a;
  }

  .todo-item-top {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
  }

  .todo-main {
    display: flex;
    align-items: center;
    gap: 15px;
  }

  .todo-text {
    font-size: 1.2em;
    flex-grow: 1;
    transition: all 0.5s ease;
  }

  .completed {
    text-decoration: line-through;
    color: #666;
    opacity: 0.6;
  }

  .tags-container {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-bottom: 5px;
  }

  .tag {
    background-color: var(--accent-color);
    color: white;
    padding: 4px 12px;
    border-radius: 12px;
    font-size: 0.9em;
  }

  .todo-completed .tag {
    background-color: #444;
  }

  button {
    padding: 8px 16px;
    background-color: var(--accent-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s;
  }

  button:hover {
    background-color: #555;
  }

  .delete-btn {
    background: none;
    border: none;
    color: #888;
    font-size: 1.5em;
    padding: 0;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
  }

  .delete-btn:hover {
    background-color: #333;
    color: white;
  }
</style>