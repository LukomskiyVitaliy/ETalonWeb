<script setup lang="ts">
import { ref, onMounted } from 'vue'
import axios from 'axios'

// --- Інтерфейси ---
interface User {
  chatId: number
  accessExpiresAt?: string
  notificationsEnabled?: boolean
}

interface Department {
  id: number
  fullAddress: string
}

// --- Реактивні змінні ---
const chatId = ref<number | null>(null)
const isLoading = ref(true)
const isLoggedIn = ref(false)
const showLogin = ref(false)
const keyInput = ref('')

const user = ref<User | null>(null)
const timerText = ref('')

const departments = ref<Department[]>([])
const subscriptions = ref<Set<number>>(new Set())

// --- Конфігурація ---
const API_BASE = 'http://localhost:5216/api'
const SECRET_KEY = 'abc123' // ⚠️ У продакшені — не хардкодити!

// --- Емуляція Telegram (тільки для dev) ---
onMounted(() => {

    window.Telegram = {
      WebApp: {
        initDataUnsafe: { user: { id: 123456789 } },
        ready: () => console.log('ready'),
        expand: () => console.log('expand')
      }
    }


  const tg = window.Telegram?.WebApp
  if (tg && tg.initDataUnsafe.user) {
    chatId.value = tg.initDataUnsafe.user.id
    tg.ready()
    tg.expand()
  }

  if (!chatId.value) {
    alert('Відкрийте через Telegram')
    isLoading.value = false
    return
  }

  checkUserStatus()
})

// --- Перевірка користувача ---
async function checkUserStatus() {
  try {
    const res = await axios.get<User>(`${API_BASE}/user/${chatId.value}`)
    user.value = res.data
    isLoggedIn.value = true
    if (user.value.accessExpiresAt) {
      startTimer(user.value.accessExpiresAt)
    }
    await loadDepartments()
    await loadSubscriptions()
  } catch (error: any) {
    if (error.response?.status === 404) {
      showLogin.value = true
    } else {
      alert('Помилка з’єднання')
    }
  } finally {
    isLoading.value = false
  }
}

// --- Логін ---
async function handleLogin() {
  if (!keyInput.value.trim()) {
    alert('Введіть ключ')
    return
  }

  try {
    await axios.post(`${API_BASE}/user/login?apiKey=${keyInput.value}`, { chatId: chatId.value })
    alert('✅ Успішний вхід!')
    keyInput.value = ''
    showLogin.value = false
    await checkUserStatus()
  } catch (error: any) {
    if (error.response?.status === 401) {
      alert('❌ Невірний або протермінований ключ')
    } else {
      alert('❌ Помилка сервера')
    }
  }
}

// --- Таймер ---
function startTimer(expiresAt: string) {
  const endTime = new Date(expiresAt).getTime()
  const update = () => {
    const now = Date.now()
    const diff = endTime - now
    if (diff <= 0) {
      timerText.value = '❌ Доступ закінчено'
      isLoggedIn.value = false
      return
    }
    const days = Math.floor(diff / (1000 * 60 * 60 * 24))
    const hours = Math.floor((diff % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60))
    const minutes = Math.floor((diff % (1000 * 60 * 60)) / (1000 * 60))
    timerText.value = `⏱️ Закінчується через: ${days}д ${hours}г ${minutes}хв`
  }
  update()
  setInterval(update, 60000)
}

// Тепер використовуємо PUT /api/user
async function toggleMonitoring() {
  if (!user.value) return;

  const updatedUser = {
    ...user.value,
    notificationsEnabled: !user.value.notificationsEnabled
  };

  try {
    await axios.put(`${API_BASE}/user`, updatedUser);
    user.value.notificationsEnabled = updatedUser.notificationsEnabled;
  } catch (error: any) {
    if (error.response?.status === 404) {
      alert('❌ Користувач не знайдений');
    } else {
      alert('❌ Помилка при оновленні налаштувань');
    }
  }
}

// --- Завантаження департаментів ---
async function loadDepartments() {
  try {
    const res = await axios.get<Record<number, string>>(`${API_BASE}/subscriptions/departments`)
    departments.value = Object.entries(res.data).map(([id, fullAddress]) => ({
      id: Number(id),
      fullAddress
    }))
  } catch (error) {
    alert('Не вдалося завантажити заклади')
  }
}

// --- Завантаження підписок ---
async function loadSubscriptions() {
  try {
    const res = await axios.get<number[]>(`${API_BASE}/subscriptions/user/${chatId.value}`)
    subscriptions.value = new Set(res.data)
  } catch (error) {
    console.warn('Не вдалося завантажити підписки')
  }
}

// --- Підписка ---
async function handleSubscription(deptId: number, checked: boolean) {
  try {
    if (checked) {
      await axios.put(`${API_BASE}/subscriptions`, { userId: chatId.value, departmentId: deptId })
      subscriptions.value.add(deptId)
    } else {
      await axios.delete(`${API_BASE}/subscriptions`, {
        data: { userId: chatId.value, departmentId: deptId } // ⚠️ data, не body
      })
      subscriptions.value.delete(deptId)
    }
  } catch (error: any) {
    if (error.response?.status === 404) {
      alert('❌ Заклад не існує')
    } else {
      alert(checked ? '❌ Помилка підписки' : '❌ Помилка відписки')
    }
  }
}
</script>

<template>
  <div v-if="isLoading" class="center">Завантаження...</div>

  <div v-else>
    <div v-if="showLogin" class="form">
      <h2>🔐 Вхід</h2>
      <input v-model="keyInput" type="text" placeholder="Секретний ключ" class="input" />
      <button @click="handleLogin" class="btn">Відправити</button>
    </div>

    <div v-if="isLoggedIn && !showLogin" class="main">
      <h2>Профіль</h2>
      <div class="timer">{{ timerText }}</div>

      <button
        @click="toggleMonitoring"
        :class="['btn', user?.notificationsEnabled ? 'btn-stop' : 'btn-start']"
      >
        {{ user?.notificationsEnabled ? '⛔ Зупинити' : '✅ Стежити' }}
      </button>

      <h3>📌 Заклади</h3>
      <div class="departments">
        <div v-for="dept in departments" :key="dept.id" class="dept-item">
          <label>
            <input
              type="checkbox"
              :checked="subscriptions.has(dept.id)"
              @change="handleSubscription(dept.id, ($event.target as HTMLInputElement).checked)"
            />
            <span>{{ dept.fullAddress }}</span>
          </label>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* --- Загальний фон та кольори --- */
body, .form, .main {
  background-color: #121212;
  color: #e0e0e0;
  font-family: 'Segoe UI', Roboto, sans-serif;
}

/* --- Центрування для loading --- */
.center {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 70vh;
  font-size: 18px;
  color: #bbb;
  padding: 20px;
}

/* --- Контейнер форм та основного контенту --- */
.form, .main {
  padding: 20px;
  max-width: 500px;
  margin: 0 auto;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.5);
  display: flex;
  flex-direction: column;
  gap: 15px;
  background-color: #1e1e1e;
}

/* --- Заголовки --- */
h2, h3 {
  text-align: center;
  color: #ffffff;
}

h2 {
  font-size: 20px;
  font-weight: 600;
  margin-bottom: 10px;
}

h3 {
  font-size: 18px;
  font-weight: 500;
  margin-top: 20px;
}

/* --- Інпут --- */
.input {
  width: 100%;
  padding: 14px;
  border: 1px solid #333;
  border-radius: 10px;
  font-size: 16px;
  background-color: #2c2c2c;
  color: #e0e0e0;
}

.input::placeholder {
  color: #888;
}

.input:focus {
  border-color: #4caf50;
  outline: none;
}

/* --- Кнопки --- */
.btn {
  width: 100%;
  padding: 14px;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: background 0.2s, transform 0.1s;
}

.btn:hover {
  transform: translateY(-2px);
}

.btn-start {
  background-color: #4caf50;
  color: #fff;
}

.btn-start:hover {
  background-color: #45a049;
}

.btn-stop {
  background-color: #f44336;
  color: #fff;
}

.btn-stop:hover {
  background-color: #e53935;
}

/* --- Таймер --- */
.timer {
  font-size: 16px;
  padding: 12px;
  background: #2a2a2a;
  border-radius: 10px;
  text-align: center;
  font-weight: 500;
  color: #4caf50;
}

/* --- Список департаментів --- */
.departments {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 10px;
  max-height: 300px;  /* Висота блоку для скролу */
  overflow-y: auto;    /* Вертикальний скрол */
  padding-right: 4px;  /* Щоб контент не обрізався */
  scroll-behavior: smooth; /* Плавний скрол */
}

/* Стиль скролбару для WebKit (Chrome, Safari) */
.departments::-webkit-scrollbar {
  width: 6px;
}

.departments::-webkit-scrollbar-thumb {
  background-color: #4caf50;
  border-radius: 3px;
}

.departments::-webkit-scrollbar-track {
  background: #1e1e1e;
  border-radius: 3px;
}

/* --- Картки департаментів --- */
.dept-item {
  background: #1f1f1f;
  border-radius: 10px;
  padding: 12px 10px;
  transition: background 0.2s;
}

.dept-item:hover {
  background: #2a2a2a;
}

.dept-item label {
  display: flex;
  align-items: center;
  gap: 12px;
  cursor: pointer;
  font-size: 15px;
  color: #e0e0e0;
}

/* --- Чекбокси --- */
.dept-item input[type="checkbox"] {
  width: 20px;
  height: 20px;
  accent-color: #4caf50;
  cursor: pointer;
}

/* --- Адаптивність --- */
@media (max-width: 480px) {
  .form, .main {
    padding: 15px;
    border-radius: 8px;
  }

  .btn {
    padding: 12px;
    font-size: 15px;
  }

  .timer {
    font-size: 14px;
    padding: 10px;
  }

  .dept-item label {
    font-size: 14px;
  }

  .departments {
    max-height: 250px;
  }
}
</style>


