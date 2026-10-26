# Vue 3 + Bootstrap 5 + Axios Pagination Tutorial

Build a simple Vue app that fetches posts from an API and displays **10 posts per page** with **Previous** and **Next** buttons.

## 1. Requirements

- Node.js and npm
- A code editor (such as VS Code)

## 2. Create the Vue project

Open a terminal and run:

```bash
npm create vite@latest vue-bootstrap-pagination -- --template vue
cd vue-bootstrap-pagination
npm install
```

Install Axios and Bootstrap:

```bash
npm install axios bootstrap
```

## 3. Project structure

```text
vue-bootstrap-pagination/
├── src/
│   ├── App.vue
│   └── main.js
├── index.html
└── package.json
```

Vite creates other configuration files automatically; you do not need to change them.

## 4. Load Bootstrap in `src/main.js`

Replace the file's content with:

```javascript
import { createApp } from 'vue'
import 'bootstrap/dist/css/bootstrap.min.css'
import App from './App.vue'

createApp(App).mount('#app')
```

Bootstrap supplies the table, button, spacing, and alert classes.

## 5. Build the app in `src/App.vue`

Replace the file with the following complete code:

```vue
<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const posts = ref([])
const page = ref(1)
const limit = 10
const totalPages = ref(10) // JSONPlaceholder has 100 posts
const loading = ref(false)
const error = ref('')

async function fetchPosts() {
  loading.value = true
  error.value = ''

  try {
    const response = await axios.get(
      'https://jsonplaceholder.typicode.com/posts',
      { params: { _page: page.value, _limit: limit } }
    )
    posts.value = response.data
  } catch (err) {
    error.value = 'Could not load posts. Please try again.'
    posts.value = []
  } finally {
    loading.value = false
  }
}

function nextPage() {
  if (page.value < totalPages.value && !loading.value) {
    page.value++
    fetchPosts()
  }
}

function previousPage() {
  if (page.value > 1 && !loading.value) {
    page.value--
    fetchPosts()
  }
}

onMounted(fetchPosts)
</script>

<template>
  <div class="container py-4">
    <h1 class="mb-3">Posts</h1>

    <div v-if="loading" class="alert alert-info">Loading...</div>
    <div v-if="error" class="alert alert-danger">{{ error }}</div>

    <table class="table table-striped table-bordered">
      <thead>
        <tr>
          <th>ID</th>
          <th>Title</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="post in posts" :key="post.id">
          <td>{{ post.id }}</td>
          <td>{{ post.title }}</td>
        </tr>
      </tbody>
    </table>

    <div class="d-flex align-items-center gap-3">
      <button
        class="btn btn-primary"
        :disabled="page === 1 || loading"
        @click="previousPage"
      >
        Previous
      </button>

      <span>Page {{ page }} of {{ totalPages }}</span>

      <button
        class="btn btn-primary"
        :disabled="page === totalPages || loading"
        @click="nextPage"
      >
        Next
      </button>
    </div>
  </div>
</template>
```

## 6. Run your application

```bash
npm run dev
```

Open the local URL printed by Vite (usually `http://localhost:5173`).

You should see a Bootstrap-styled table containing 10 posts and navigation buttons underneath.

## 7. Understand the Axios request

```javascript
const response = await axios.get(
  'https://jsonplaceholder.typicode.com/posts',
  { params: { _page: page.value, _limit: 10 } }
)
```

On page 1, Axios requests:

```text
https://jsonplaceholder.typicode.com/posts?_page=1&_limit=10
```

Click **Next** and it requests:

```text
https://jsonplaceholder.typicode.com/posts?_page=2&_limit=10
```

- `_page` indicates which page the server returns.
- `_limit` indicates how many posts to return.
- `response.data` contains the posts for that page.
- `posts.value = response.data` updates the Vue table automatically.

## 8. How the Next and Previous buttons work

**Next** increases the page number and fetches that page:

```javascript
function nextPage() {
  if (page.value < totalPages.value && !loading.value) {
    page.value++
    fetchPosts()
  }
}
```

**Previous** decreases the page number and fetches the prior page:

```javascript
function previousPage() {
  if (page.value > 1 && !loading.value) {
    page.value--
    fetchPosts()
  }
}
```

The `:disabled` bindings prevent navigating past the first/last page and disable the buttons while loading.

## 9. Why `onMounted(fetchPosts)` is needed

```javascript
onMounted(fetchPosts)
```

It loads page 1 when the component first appears, before the user clicks any button.

## 10. Quick testing checklist

1. Load the app: page 1 should show posts 1–10.
2. Click **Next**: page 2 should show posts 11–20.
3. Click **Previous**: it should return to page 1.
4. On page 1, **Previous** should be disabled.
5. On page 10, **Next** should be disabled.

## 11. Use your own API later

Replace the URL with your endpoint and adapt the query parameter names and response format to that API. The tutorial uses `totalPages = 10` because the demonstration API has exactly 100 posts. A production API should return a total count or total pages so you can calculate the last page dynamically. For example, if your server returns `{ items: [...], total: 87 }`:

```javascript
posts.value = response.data.items
totalPages.value = Math.ceil(response.data.total / limit)
```

**Summary:** Vue's `ref()` stores the current page and posts, Axios requests only that page, and Bootstrap styles the table and pagination controls.
