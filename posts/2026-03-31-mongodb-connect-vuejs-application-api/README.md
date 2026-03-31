# How to Connect MongoDB to a Vue.js Application via API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Vue, API, Node.js, JavaScript

Description: Learn how to connect MongoDB to a Vue.js application through a Node.js Express API using Axios and Pinia for state management.

---

## Architecture

Vue.js is a browser framework and cannot connect to MongoDB directly. Use a Node.js/Express backend API as the data access layer. Vue communicates with the API via HTTP, keeping MongoDB credentials and logic server-side.

```text
Vue.js (browser)  -->  Express API (Node.js)  -->  MongoDB
```

## Setting Up the Express API

```bash
mkdir mongo-api && cd mongo-api
npm init -y
npm install express mongoose cors dotenv
```

```javascript
// server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors({ origin: process.env.CLIENT_URL || 'http://localhost:5173' }));
app.use(express.json());

mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('MongoDB connected'));

const TaskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  completed: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now }
});

const Task = mongoose.model('Task', TaskSchema);

app.get('/api/tasks', async (req, res) => {
  const tasks = await Task.find().sort({ createdAt: -1 });
  res.json(tasks);
});

app.post('/api/tasks', async (req, res) => {
  try {
    const task = await Task.create(req.body);
    res.status(201).json(task);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.patch('/api/tasks/:id', async (req, res) => {
  const task = await Task.findByIdAndUpdate(
    req.params.id, req.body, { new: true }
  );
  res.json(task);
});

app.delete('/api/tasks/:id', async (req, res) => {
  await Task.findByIdAndDelete(req.params.id);
  res.status(204).end();
});

app.listen(process.env.PORT || 5000);
```

## Setting Up the Vue App with Vite

```bash
npm create vite@latest frontend -- --template vue
cd frontend
npm install axios pinia
```

Configure the API base URL using Vite environment variables:

```text
# .env
VITE_API_URL=http://localhost:5000
```

## Creating the API Layer

```javascript
// src/api/tasks.js
import axios from 'axios';

const http = axios.create({
  baseURL: import.meta.env.VITE_API_URL
});

export const taskApi = {
  getAll: () => http.get('/api/tasks'),
  create: (data) => http.post('/api/tasks', data),
  update: (id, data) => http.patch(`/api/tasks/${id}`, data),
  remove: (id) => http.delete(`/api/tasks/${id}`)
};
```

## Pinia Store for Task State

```javascript
// src/stores/taskStore.js
import { defineStore } from 'pinia';
import { ref } from 'vue';
import { taskApi } from '../api/tasks';

export const useTaskStore = defineStore('tasks', () => {
  const tasks = ref([]);
  const loading = ref(false);
  const error = ref(null);

  async function fetchTasks() {
    loading.value = true;
    try {
      const res = await taskApi.getAll();
      tasks.value = res.data;
    } catch (err) {
      error.value = err.message;
    } finally {
      loading.value = false;
    }
  }

  async function addTask(title) {
    const res = await taskApi.create({ title });
    tasks.value.unshift(res.data);
  }

  async function toggleTask(id) {
    const task = tasks.value.find(t => t._id === id);
    const res = await taskApi.update(id, { completed: !task.completed });
    const idx = tasks.value.findIndex(t => t._id === id);
    tasks.value[idx] = res.data;
  }

  async function removeTask(id) {
    await taskApi.remove(id);
    tasks.value = tasks.value.filter(t => t._id !== id);
  }

  return { tasks, loading, error, fetchTasks, addTask, toggleTask, removeTask };
});
```

## Vue Component Using the Store

```vue
<!-- src/components/TaskList.vue -->
<template>
  <div>
    <form @submit.prevent="handleAdd">
      <input v-model="newTitle" placeholder="New task" required />
      <button type="submit">Add</button>
    </form>
    <p v-if="taskStore.loading">Loading...</p>
    <p v-if="taskStore.error" class="error">{{ taskStore.error }}</p>
    <ul>
      <li v-for="task in taskStore.tasks" :key="task._id">
        <input type="checkbox"
          :checked="task.completed"
          @change="taskStore.toggleTask(task._id)" />
        {{ task.title }}
        <button @click="taskStore.removeTask(task._id)">Delete</button>
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useTaskStore } from '../stores/taskStore';

const taskStore = useTaskStore();
const newTitle = ref('');

onMounted(() => taskStore.fetchTasks());

async function handleAdd() {
  if (!newTitle.value.trim()) return;
  await taskStore.addTask(newTitle.value.trim());
  newTitle.value = '';
}
</script>
```

Register Pinia in `main.js`:

```javascript
// src/main.js
import { createApp } from 'vue';
import { createPinia } from 'pinia';
import App from './App.vue';

const app = createApp(App);
app.use(createPinia());
app.mount('#app');
```

## Summary

Vue.js connects to MongoDB through a Node.js/Express API, not directly. Use Vite environment variables (`VITE_API_URL`) for API configuration, Axios for HTTP calls, and Pinia for reactive state management. Structure your data access in a dedicated API module separate from your stores and components to keep concerns separated and make testing straightforward.
