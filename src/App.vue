<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'

const documentTypes = [
  'Certificate of Enrollment',
  'Good Moral Certificate',
  'Transcript of Records',
  'Certificate of Grades',
  'Other school documents',
]
const courses = [
  'Bachelor of Elementary Education',
  'Bachelor of Secondary Education',
  'Bachelor of Science in Accountancy',
  'Bachelor of Science in Biology',
  'Bachelor of Science in Business Administration',
  'Bachelor of Science in Civil Engineering',
  'Bachelor of Science in Computer Science',
  'Bachelor of Science in Criminology',
  'Bachelor of Science in Education',
  'Bachelor of Science in Electrical Engineering',
  'Bachelor of Science in Hospitality Management',
  'Bachelor of Science in Information Technology',
  'Bachelor of Science in Nursing',
  'Bachelor of Science in Psychology',
  'Bachelor of Science in Tourism Management',
]
const yearLevels = ['1st Year', '2nd Year', '3rd Year', '4th Year']
const statuses = ['Pending', 'Processing', 'Ready for Pickup', 'Completed']
const navItems = [
  { id: 'dashboard', label: 'Dashboard', icon: 'grid' },
  { id: 'request', label: 'New request', icon: 'plus' },
  { id: 'requests', label: 'My requests', icon: 'file' },
  { id: 'profile', label: 'My profile', icon: 'user' },
]

const accountsKey = 'campus-accounts'

function readAccounts() {
  try {
    const stored = localStorage.getItem(accountsKey)
    const parsed = stored ? JSON.parse(stored) : []
    return Array.isArray(parsed)
      ? parsed.filter((account) => account && typeof account.email === 'string')
      : []
  } catch (error) {
    console.error('Could not load saved accounts:', error)
    return []
  }
}

const accounts = ref(readAccounts())
const authMode = ref(accounts.value.length ? 'login' : 'register')
const createAccountForm = ref({
  name: '',
  studentId: '',
  email: '',
  course: '',
  year: '',
  password: '',
  confirmPassword: '',
})
const requests = ref([])
const profile = ref({ name: '', studentId: '', email: '', course: '', year: '' })
const isSignedIn = ref(false)
const currentAccountEmail = ref('')
const loginForm = ref({ email: '', password: '' })
const loginError = ref('')
const currentPage = ref('dashboard')
const search = ref('')
const statusFilter = ref('All requests')
const formError = ref('')
const toast = ref('')
const editingId = ref(null)
const requestForm = ref(emptyForm())
const profileSaved = ref(false)
let toastTimer

function emptyForm() {
  return { document: '', purpose: '', copies: '' }
}

function localDate() {
  const today = new Date()
  const year = today.getFullYear()
  const month = String(today.getMonth() + 1).padStart(2, '0')
  const day = String(today.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

function navigate(page, query = '') {
  const url = `/${page === 'dashboard' ? '' : page}${query}`
  window.history.pushState({ page }, '', url)
  currentPage.value = page
  formError.value = ''
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function bytesToHex(bytes) {
  return Array.from(new Uint8Array(bytes), (byte) => byte.toString(16).padStart(2, '0')).join('')
}

function hexToBytes(hex) {
  return new Uint8Array(hex.match(/.{2}/g).map((byte) => Number.parseInt(byte, 16)))
}

async function hashPassword(password, salt) {
  if (!window.crypto?.subtle) {
    throw new Error('Secure password storage is unavailable. Open this app on localhost or HTTPS.')
  }
  const key = await window.crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(password),
    'PBKDF2',
    false,
    ['deriveBits'],
  )
  return bytesToHex(
    await window.crypto.subtle.deriveBits(
      { name: 'PBKDF2', salt, iterations: 310000, hash: 'SHA-256' },
      key,
      256,
    ),
  )
}

function saveAccounts(nextAccounts) {
  try {
    localStorage.setItem(accountsKey, JSON.stringify(nextAccounts))
    accounts.value = nextAccounts
    return true
  } catch (error) {
    console.error('Could not save accounts:', error)
    loginError.value = 'Your account could not be saved in this browser. Check available storage and try again.'
    return false
  }
}

function openAccount(account) {
  currentAccountEmail.value = account.email
  profile.value = { ...account.profile }
  requests.value = Array.isArray(account.requests) ? [...account.requests] : []
  isSignedIn.value = true
  loginError.value = ''
  loginForm.value.password = ''
  navigate('dashboard')
}

async function createAccount() {
  loginError.value = ''
  const form = createAccountForm.value
  const email = form.email.trim().toLowerCase()

  if (accounts.value.some((account) => account.email === email)) {
    loginError.value = 'An account with this email already exists. Sign in instead.'
    return
  }
  if (form.password.length < 8) {
    loginError.value = 'Choose a password with at least 8 characters.'
    return
  }
  if (form.password !== form.confirmPassword) {
    loginError.value = 'The passwords do not match. Check both password fields.'
    return
  }

  try {
    const salt = window.crypto.getRandomValues(new Uint8Array(16))
    const account = {
      email,
      salt: bytesToHex(salt),
      passwordHash: await hashPassword(form.password, salt),
      profile: {
        name: form.name.trim(),
        studentId: form.studentId.trim(),
        email,
        course: form.course.trim(),
        year: form.year,
      },
      requests: [],
    }
    const nextAccounts = [...accounts.value, account]
    if (!saveAccounts(nextAccounts)) return
    createAccountForm.value.password = ''
    createAccountForm.value.confirmPassword = ''
    openAccount(account)
  } catch (error) {
    console.error('Could not create account:', error)
    loginError.value = error.message || 'Your account could not be created. Please try again.'
  }
}

async function submitLogin() {
  loginError.value = ''
  const email = loginForm.value.email.trim().toLowerCase()
  const account = accounts.value.find((item) => item.email === email)
  if (!account) {
    loginError.value = 'No account was found for this email. Create an account first.'
    return
  }

  try {
    const passwordHash = await hashPassword(loginForm.value.password, hexToBytes(account.salt))
    if (passwordHash !== account.passwordHash) {
      loginError.value = 'The password is incorrect. Try again.'
      return
    }
    openAccount(account)
  } catch (error) {
    console.error('Could not sign in:', error)
    loginError.value = error.message || 'Sign-in failed. Please try again.'
  }
}

function changeAuthMode(mode) {
  authMode.value = mode
  loginError.value = ''
  loginForm.value.password = ''
}

function signOut() {
  loginForm.value.email = currentAccountEmail.value
  loginForm.value.password = ''
  currentAccountEmail.value = ''
  profile.value = { name: '', studentId: '', email: '', course: '', year: '' }
  requests.value = []
  isSignedIn.value = false
  authMode.value = 'login'
  loginError.value = ''
  currentPage.value = 'dashboard'
  window.history.replaceState({ page: 'dashboard' }, '', '/')
}

function saveCurrentAccount(changes = {}) {
  const accountIndex = accounts.value.findIndex((account) => account.email === currentAccountEmail.value)
  if (accountIndex === -1) return false

  const updatedAccount = {
    ...accounts.value[accountIndex],
    ...changes,
  }
  const nextAccounts = [...accounts.value]
  nextAccounts[accountIndex] = updatedAccount
  return saveAccounts(nextAccounts)
}

function syncRoute() {
  const path = window.location.pathname.replace(/^\/+|\/+$/g, '')
  currentPage.value = ['request', 'requests', 'profile'].includes(path) ? path : 'dashboard'
  const query = new URLSearchParams(window.location.search)
  const editId = query.get('edit')
  if (currentPage.value === 'request' && editId) {
    const request = requests.value.find((item) => item.id === editId)
    if (request) beginEdit(request)
  }
}

function beginEdit(request) {
  editingId.value = request.id
  requestForm.value = {
    document: request.document,
    purpose: request.purpose,
    copies: request.copies,
  }
}

function resetForm() {
  editingId.value = null
  requestForm.value = emptyForm()
  formError.value = ''
}

function submitRequest() {
  const form = requestForm.value
  if (
    !form.document ||
    !form.purpose.trim() ||
    !Number.isInteger(Number(form.copies)) ||
    Number(form.copies) < 1 ||
    Number(form.copies) > 10
  ) {
    formError.value = 'Complete each required field and choose between 1 and 10 copies.'
    return
  }
  if (editingId.value) {
    const index = requests.value.findIndex((request) => request.id === editingId.value)
    if (index !== -1) {
      requests.value[index] = {
        ...requests.value[index],
        document: form.document,
        purpose: form.purpose.trim(),
        copies: Number(form.copies),
      }
    }
    notify('Your request has been updated.')
  } else {
    const id = `SD-${new Date().getFullYear()}-${String(Date.now()).slice(-4)}`
    requests.value.unshift({
      id,
      document: form.document,
      purpose: form.purpose.trim(),
      copies: Number(form.copies),
      status: 'Pending',
      date: localDate(),
      release: '',
    })
    notify('Your document request has been submitted.')
  }
  resetForm()
  navigate('requests')
}

function deleteRequest(request) {
  if (!window.confirm(`Delete request ${request.id}? This cannot be undone.`)) return
  requests.value = requests.value.filter((item) => item.id !== request.id)
  notify('Request deleted.')
}

function saveProfile() {
  const email = profile.value.email.trim().toLowerCase()
  if (accounts.value.some((account) => account.email === email && account.email !== currentAccountEmail.value)) {
    notify('That email address is already used by another account.')
    return
  }
  const saved = saveCurrentAccount({ profile: { ...profile.value, email }, email })
  if (!saved) {
    notify('Your profile could not be saved.')
    return
  }
  currentAccountEmail.value = email
  loginForm.value.email = email
  profile.value.email = email
  profileSaved.value = true
  notify('Your profile has been saved.')
  setTimeout(() => (profileSaved.value = false), 2500)
}

function notify(message) {
  toast.value = message
  clearTimeout(toastTimer)
  toastTimer = setTimeout(() => (toast.value = ''), 3200)
}

function formatDate(date) {
  if (!date) return 'To be announced'
  return new Intl.DateTimeFormat('en', { month: 'short', day: 'numeric', year: 'numeric' }).format(
    new Date(`${date}T00:00:00`),
  )
}

function statusClass(status) {
  return status.toLowerCase().replaceAll(' ', '-')
}

const filteredRequests = computed(() => {
  const term = search.value.trim().toLowerCase()
  return requests.value.filter((request) => {
    const matchesSearch =
      !term ||
      [request.id, request.document, request.purpose].some((value) => value.toLowerCase().includes(term))
    const matchesStatus = statusFilter.value === 'All requests' || request.status === statusFilter.value
    return matchesSearch && matchesStatus
  })
})

const activeRequest = computed(
  () => requests.value.find((request) => !['Completed'].includes(request.status)) || requests.value[0],
)
const readyCount = computed(() => requests.value.filter((request) => request.status === 'Ready for Pickup').length)
const profileInitials = computed(() =>
  profile.value.name
    .trim()
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((part) => part[0].toUpperCase())
    .join('') || '?',
)

watch(
  requests,
  (value) => {
    if (currentAccountEmail.value && !saveCurrentAccount({ requests: value })) {
      notify('Your request changes could not be saved in this browser.')
    }
  },
  { deep: true },
)

onMounted(() => {
  syncRoute()
  window.addEventListener('popstate', syncRoute)
})
onUnmounted(() => {
  window.removeEventListener('popstate', syncRoute)
  clearTimeout(toastTimer)
})
</script>

<template>
  <main v-if="!isSignedIn" class="login-page">
    <section class="login-card">
      <a class="brand login-brand" href="/" aria-label="Campusdesk home">
        <span class="brand-mark"><span></span><span></span><span></span><span></span></span>
        <span class="brand-name">campus<span>desk</span></span>
      </a>
      <p class="login-school">STUDENT DOCUMENT PORTAL</p>
      <template v-if="authMode === 'register'">
        <h1>Create your account</h1>
        <p class="login-intro">Enter your student and account details to get started. All fields are required.</p>
        <form class="login-form account-form" @submit.prevent="createAccount">
          <label class="field-label" for="account-name">Full name <span class="required">*</span></label>
          <input id="account-name" v-model="createAccountForm.name" type="text" autocomplete="name" placeholder="Enter your full name" required />
          <label class="field-label" for="account-student-id">Student ID <span class="required">*</span></label>
          <input id="account-student-id" v-model="createAccountForm.studentId" type="text" placeholder="Enter your student ID" required />
          <label class="field-label" for="account-email">Email address <span class="required">*</span></label>
          <input id="account-email" v-model="createAccountForm.email" type="email" autocomplete="email" placeholder="Enter your email address" required />
          <label class="field-label" for="account-course">Program or course <span class="required">*</span></label>
          <select id="account-course" v-model="createAccountForm.course" required>
            <option value="" disabled>Select your program or course</option>
            <option v-for="course in courses" :key="course" :value="course">{{ course }}</option>
          </select>
          <label class="field-label" for="account-year">Year level <span class="required">*</span></label>
          <select id="account-year" v-model="createAccountForm.year" required>
            <option value="" disabled>Select your year level</option>
            <option v-for="year in yearLevels" :key="year">{{ year }}</option>
          </select>
          <label class="field-label" for="account-password">Create password <span class="required">*</span></label>
          <input id="account-password" v-model="createAccountForm.password" type="password" autocomplete="new-password" minlength="8" placeholder="At least 8 characters" required />
          <label class="field-label" for="account-confirm-password">Confirm password <span class="required">*</span></label>
          <input id="account-confirm-password" v-model="createAccountForm.confirmPassword" type="password" autocomplete="new-password" minlength="8" placeholder="Enter your password again" required />
          <p v-if="loginError" class="form-error" role="alert">{{ loginError }}</p>
          <button class="button button-primary login-submit" type="submit">Create account <span>→</span></button>
        </form>
        <p class="login-help">Already have an account? <button class="auth-toggle" type="button" @click="changeAuthMode('login')">Sign in</button></p>
      </template>
      <template v-else>
        <h1>Sign in to your account</h1>
        <p class="login-intro">Enter the email address and password you used when you created your account.</p>
        <form class="login-form" @submit.prevent="submitLogin">
          <label class="field-label" for="login-email">Email address <span class="required">*</span></label>
          <input id="login-email" v-model="loginForm.email" type="email" autocomplete="username" placeholder="Enter your email address" required />
          <label class="field-label" for="login-password">Password <span class="required">*</span></label>
          <input id="login-password" v-model="loginForm.password" type="password" autocomplete="current-password" placeholder="Enter your password" required />
          <p v-if="loginError" class="form-error" role="alert">{{ loginError }}</p>
          <button class="button button-primary login-submit" type="submit">Sign in <span>→</span></button>
        </form>
        <p class="login-help">Don't have an account? <button class="auth-toggle" type="button" @click="changeAuthMode('register')">Create an account</button></p>
      </template>
    </section>
    <footer class="login-footer">Create an account to request and track your school documents.</footer>
  </main>

  <div v-else class="app-shell">
    <aside class="sidebar">
      <a class="brand" href="/" aria-label="Campus desk dashboard" @click.prevent="navigate('dashboard')">
        <span class="brand-mark"><span></span><span></span><span></span><span></span></span>
        <span class="brand-name">campus<span>desk</span></span>
      </a>

      <div class="school-switcher">
        <div class="school-copy"><strong>Student portal</strong><span>Campus desk</span></div>
        <span class="chevron">⌄</span>
      </div>

      <div class="nav-heading">MENU</div>
      <nav class="main-nav" aria-label="Main navigation">
        <a
          v-for="item in navItems"
          :key="item.id"
          :href="item.id === 'dashboard' ? '/' : `/${item.id}`"
          :class="['nav-link', { active: currentPage === item.id }]"
          :aria-label="item.label"
          @click.prevent="navigate(item.id)"
        >
          <svg v-if="item.icon === 'grid'" viewBox="0 0 24 24" aria-hidden="true"><rect x="3" y="3" width="7" height="7" rx="1.5"/><rect x="14" y="3" width="7" height="7" rx="1.5"/><rect x="3" y="14" width="7" height="7" rx="1.5"/><rect x="14" y="14" width="7" height="7" rx="1.5"/></svg>
          <svg v-else-if="item.icon === 'plus'" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 5v14M5 12h14"/></svg>
          <svg v-else-if="item.icon === 'file'" viewBox="0 0 24 24" aria-hidden="true"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><path d="M14 2v6h6M8 13h8M8 17h8"/></svg>
          <svg v-else viewBox="0 0 24 24" aria-hidden="true"><circle cx="12" cy="8" r="4"/><path d="M5 21v-2a7 7 0 0 1 14 0v2"/></svg>
          <span>{{ item.label }}</span>
          <span v-if="item.id === 'requests' && readyCount" class="nav-count">{{ readyCount }}</span>
        </a>
      </nav>

      <div class="sidebar-help">
        <div class="help-icon">?</div>
        <strong>Need a hand?</strong>
        <p>Our registrar team is happy to help with your requests.</p>
        <a href="mailto:registrar@northfield.edu.ph">Contact support <span>↗</span></a>
      </div>

      <button class="student-chip" type="button" @click="navigate('profile')">
        <span class="avatar">{{ profileInitials }}</span>
        <span class="student-chip-copy"><strong>{{ profile.name || 'Complete your profile' }}</strong><small>{{ profile.studentId || 'Student details needed' }}</small></span>
        <span class="more-dots">···</span>
      </button>
    </aside>

    <main class="main-area">
      <header class="topbar">
        <div class="breadcrumb"><span>Student portal</span><span class="crumb-slash">/</span><strong>{{ navItems.find((item) => item.id === currentPage)?.label || 'Dashboard' }}</strong></div>
        <div class="topbar-actions">
          <button class="icon-button notification-button" aria-label="Notifications" @click="navigate('requests')">
            <svg viewBox="0 0 24 24" aria-hidden="true"><path d="M18 8a6 6 0 0 0-12 0c0 7-3 7-3 9h18c0-2-3-2-3-9M10 21h4"/></svg>
            <span v-if="readyCount" class="notification-dot"></span>
          </button>
          <span class="topbar-divider"></span>
          <button class="topbar-profile" @click="navigate('profile')"><span class="avatar avatar-small">{{ profileInitials }}</span><span>{{ profile.name.split(' ')[0] || 'My profile' }}</span><span class="chevron">⌄</span></button>
        </div>
      </header>

      <div class="page-content">
        <template v-if="currentPage === 'dashboard'">
          <section class="welcome-row">
            <div>
              <div class="eyebrow"><span class="eyebrow-dot"></span> YOUR STUDENT SERVICES, SIMPLIFIED</div>
              <h1>{{ profile.name ? `Welcome, ${profile.name.split(' ')[0]}` : 'Welcome to your student portal' }}</h1>
              <p class="page-subtitle">Take care of your school documents, all in one place.</p>
            </div>
            <button class="button button-primary" @click="resetForm(); navigate('request')"><span class="button-plus">+</span> Request a document</button>
          </section>

          <section class="portal-intro">
            <div><h2>Before you start</h2><p>Add your student ID, program, and year level in My profile. Each request asks for a document type, purpose, and number of copies.</p></div>
          </section>

          <section class="stats-grid" aria-label="Request summary">
            <article class="stat-card">
              <div class="stat-top"><span class="stat-icon total-icon">▤</span><span class="stat-caption">ALL TIME</span></div>
              <div class="stat-number">{{ requests.length.toString().padStart(2, '0') }}</div>
              <div class="stat-label">Total requests</div>
            </article>
            <article class="stat-card">
              <div class="stat-top"><span class="stat-icon processing-icon">◷</span><span class="stat-caption">IN PROGRESS</span></div>
              <div class="stat-number">{{ requests.filter((request) => ['Pending', 'Processing'].includes(request.status)).length.toString().padStart(2, '0') }}</div>
              <div class="stat-label">Being processed</div>
            </article>
            <article class="stat-card">
              <div class="stat-top"><span class="stat-icon ready-icon">↗</span><span class="stat-caption">ACTION NEEDED</span></div>
              <div class="stat-number">{{ readyCount.toString().padStart(2, '0') }}</div>
              <div class="stat-label">Ready for pickup</div>
            </article>
            <article class="stat-card">
              <div class="stat-top"><span class="stat-icon completed-icon">✓</span><span class="stat-caption">ALL DONE</span></div>
              <div class="stat-number">{{ requests.filter((request) => request.status === 'Completed').length.toString().padStart(2, '0') }}</div>
              <div class="stat-label">Completed</div>
            </article>
          </section>

          <section class="dashboard-lower">
            <div class="panel recent-panel">
              <div class="panel-heading"><div><h3>Recent requests</h3><p>A quick look at your documents</p></div><button class="text-link" @click="navigate('requests')">View all <span>→</span></button></div>
              <div v-if="requests.length" class="recent-list">
                <article v-for="request in requests.slice(0, 3)" :key="request.id" class="recent-item">
                  <span class="document-icon">▤</span>
                  <div class="recent-info"><strong>{{ request.document }}</strong><small>{{ request.id }} <span>·</span> {{ formatDate(request.date) }}</small></div>
                  <span :class="['status-pill', statusClass(request.status)]"><i></i>{{ request.status }}</span>
                  <button class="row-arrow" aria-label="View all requests" @click="navigate('requests')">→</button>
                </article>
              </div>
              <div v-else class="empty-state"><span class="empty-icon">▤</span><strong>No requests yet</strong><p>Your document requests will appear here.</p><button class="text-link" @click="navigate('request')">Make your first request →</button></div>
            </div>
            <aside class="pickup-card">
              <span class="pickup-overline"><span>✦</span> GOOD TO KNOW</span>
              <h3>One less trip<br />to the office.</h3>
              <p>We’ll let you know as soon as your document is ready for pickup.</p>
              <div class="pickup-foot"><span class="pickup-check">✓</span><span>Track updates right here</span></div>
              <div class="pickup-decoration">✳</div>
            </aside>
          </section>
        </template>

        <template v-else-if="currentPage === 'request'">
          <section class="page-heading">
            <div><div class="eyebrow"><span class="eyebrow-dot"></span> REGISTRAR SERVICES</div><h1>{{ editingId ? 'Edit your request' : 'Request a document' }}</h1><p class="page-subtitle">{{ editingId ? 'Update the details of your document request.' : 'Fill in the required fields below. Your student details are under My profile.' }}</p></div>
            <button class="button button-quiet" @click="navigate('requests')">← Back to requests</button>
          </section>
          <div class="form-layout">
            <form class="panel request-form" @submit.prevent="submitRequest">
              <div class="form-section-heading"><span class="step-number">01</span><div><h3>Document details</h3><p>Fields marked with <span class="required">*</span> are required. Please enter your own request details.</p></div></div>
              <label class="field-label" for="document-type">Document type <span class="required">*</span></label>
              <select id="document-type" v-model="requestForm.document" required>
                <option value="" disabled>Select a document</option>
                <option v-for="document in documentTypes" :key="document" :value="document">{{ document }}</option>
              </select>
              <label class="field-label" for="purpose">Purpose of request <span class="required">*</span></label>
              <textarea id="purpose" v-model="requestForm.purpose" rows="3" maxlength="240" placeholder="Explain why you need this document" required></textarea>
              <div class="field-hint">A short explanation helps us process your request faster.</div>
              <label class="field-label" for="copies">Number of copies <span class="required">*</span></label>
              <div class="copies-control"><button type="button" aria-label="Remove one copy" :disabled="requestForm.copies <= 1" @click="requestForm.copies = Math.max(1, Number(requestForm.copies) - 1)">−</button><input id="copies" v-model.number="requestForm.copies" type="number" min="1" max="10" required /><button type="button" aria-label="Add one copy" :disabled="requestForm.copies >= 10" @click="requestForm.copies = Math.min(10, Number(requestForm.copies) + 1)">+</button><span>Enter 1 to 10 copies</span></div>
              <div class="form-note"><span>ⓘ</span><p>Please make sure your student information is up to date. Processing time is usually <strong>3–5 working days.</strong></p></div>
              <p v-if="formError" class="form-error" role="alert">{{ formError }}</p>
              <div class="form-actions"><button type="button" class="button button-quiet" @click="resetForm(); navigate('requests')">Cancel</button><button type="submit" class="button button-primary">{{ editingId ? 'Save changes' : 'Submit request' }} <span>→</span></button></div>
            </form>
            <aside class="form-aside">
              <div class="aside-icon">✦</div><h3>What happens next?</h3>
              <div class="next-step"><span>1</span><div><strong>We review your request</strong><p>The registrar checks your student records.</p></div></div>
              <div class="next-step"><span>2</span><div><strong>We prepare your document</strong><p>Follow the progress from My requests.</p></div></div>
              <div class="next-step"><span>3</span><div><strong>Pick it up when it’s ready</strong><p>We’ll notify you when it’s ready to collect.</p></div></div>
              <div class="aside-contact">Questions? <a href="mailto:registrar@northfield.edu.ph">Contact the registrar →</a></div>
            </aside>
          </div>
        </template>

        <template v-else-if="currentPage === 'requests'">
          <section class="page-heading">
            <div><div class="eyebrow"><span class="eyebrow-dot"></span> YOUR DOCUMENTS</div><h1>My requests <span class="heading-count">{{ requests.length }}</span></h1><p class="page-subtitle">Keep track of every document, from request to ready.</p></div>
            <button class="button button-primary" @click="resetForm(); navigate('request')"><span class="button-plus">+</span> New request</button>
          </section>
          <section v-if="activeRequest && activeRequest.status !== 'Completed'" class="tracking-banner">
            <div class="tracking-icon">✳</div><div class="tracking-copy"><span class="tracking-label">YOUR LATEST UPDATE <span>·</span> {{ activeRequest.id }}</span><h3>{{ activeRequest.status === 'Ready for Pickup' ? 'Your document is ready!' : 'Your request is moving along.' }}</h3><p>{{ activeRequest.document }} <span>·</span> {{ activeRequest.status === 'Ready for Pickup' ? 'Visit the registrar to collect your document.' : `Estimated release ${formatDate(activeRequest.release)}.` }}</p></div><span :class="['status-pill', statusClass(activeRequest.status)]"><i></i>{{ activeRequest.status }}</span>
          </section>
          <section class="panel requests-panel">
            <div class="requests-toolbar"><div><h3>All requests</h3><p>{{ requests.length }} document{{ requests.length === 1 ? '' : 's' }} in your history</p></div><div class="filters"><label class="search-field"><svg viewBox="0 0 24 24" aria-hidden="true"><circle cx="11" cy="11" r="7"/><path d="m20 20-4-4"/></svg><input v-model="search" type="search" placeholder="Search requests..." /></label><select v-model="statusFilter" aria-label="Filter requests by status"><option>All requests</option><option v-for="status in statuses" :key="status">{{ status }}</option></select></div></div>
            <div class="table-wrap"><table><thead><tr><th>DOCUMENT</th><th>REQUEST ID</th><th>DATE REQUESTED</th><th>STATUS</th><th>EST. RELEASE</th><th><span class="sr-only">Actions</span></th></tr></thead><tbody>
              <tr v-for="request in filteredRequests" :key="request.id"><td><div class="table-document"><span class="document-icon">▤</span><span><strong>{{ request.document }}</strong><small>{{ request.copies }} {{ request.copies === 1 ? 'copy' : 'copies' }} · {{ request.purpose }}</small></span></div></td><td class="id-cell">{{ request.id }}</td><td>{{ formatDate(request.date) }}</td><td><span :class="['status-pill', statusClass(request.status)]"><i></i>{{ request.status }}</span></td><td>{{ formatDate(request.release) }}</td><td><div class="row-actions"><button class="action-button" aria-label="Edit request" title="Edit request" @click="navigate('request', `?edit=${encodeURIComponent(request.id)}`); beginEdit(request)">✎</button><button class="action-button delete-action" aria-label="Delete request" title="Delete request" @click="deleteRequest(request)">×</button></div></td></tr>
              <tr v-if="!filteredRequests.length"><td colspan="6"><div class="empty-state table-empty"><span class="empty-icon">⌕</span><strong>{{ requests.length ? 'No matching requests' : 'No requests yet' }}</strong><p>{{ requests.length ? 'Try changing your search or filter.' : 'Your document requests will show up here.' }}</p><button v-if="!requests.length" class="text-link" @click="navigate('request')">Create a request →</button></div></td></tr>
            </tbody></table></div>
            <div class="table-foot"><span>Showing <strong>{{ filteredRequests.length }}</strong> of <strong>{{ requests.length }}</strong> requests</span><span>Need help? <a href="mailto:registrar@northfield.edu.ph">Contact the registrar →</a></span></div>
          </section>
        </template>

        <template v-else>
          <section class="page-heading">
            <div><div class="eyebrow"><span class="eyebrow-dot"></span> YOUR ACCOUNT</div><h1>My profile</h1><p class="page-subtitle">Your student information, all in one place.</p></div>
          </section>
          <div class="profile-layout">
            <form class="panel profile-form" @submit.prevent="saveProfile">
              <div class="panel-heading"><div><h3>Student information</h3><p>Fill in your details as they appear in your school records. All fields are required.</p></div></div>
              <label class="field-label" for="student-name">Full name <span class="required">*</span></label><input id="student-name" v-model="profile.name" type="text" autocomplete="name" placeholder="Enter your full name" required />
              <label class="field-label" for="student-id">Student ID <span class="required">*</span></label><input id="student-id" v-model="profile.studentId" type="text" placeholder="Enter your student ID" required />
              <label class="field-label" for="student-email">School email address <span class="required">*</span></label><input id="student-email" v-model="profile.email" type="email" autocomplete="email" placeholder="you@school.edu" required />
              <label class="field-label" for="student-course">Program <span class="required">*</span></label><select id="student-course" v-model="profile.course" required><option value="" disabled>Select your program or course</option><option v-for="course in courses" :key="course" :value="course">{{ course }}</option></select>
              <label class="field-label" for="student-year">Year level <span class="required">*</span></label><select id="student-year" v-model="profile.year" required><option value="" disabled>Select your year level</option><option v-for="year in yearLevels" :key="year">{{ year }}</option></select>
              <div class="profile-save"><span v-if="profileSaved" class="save-confirm">✓ Changes saved</span><button class="button button-primary" type="submit">Save changes <span>→</span></button></div>
            </form>
          </div>
          <button class="sign-out-button" type="button" @click="signOut">Sign out</button>
        </template>
      </div>
      <footer class="page-footer"><span>© {{ new Date().getFullYear() }} Campus Desk</span></footer>
    </main>
    <Transition name="toast"><div v-if="toast" class="toast-message" role="status"><span>✓</span>{{ toast }}</div></Transition>
  </div>
</template>
