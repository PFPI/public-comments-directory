<template>
  <div class="container">
    <header class="header">
      <div class="header-content">
        <h1>PFPI Comment Tracker</h1>
        <div class="tabs">
          <button :class="{ active: view === 'public' }" @click="view = 'public'">Library</button>
          <button :class="{ active: view === 'admin' }" @click="view = 'admin'">Admin</button>
        </div>
      </div>
    </header>

    <main v-if="view === 'public'" class="view-panel">

      <div class="filter-bar">
        <div class="search-box">
          <input v-model="searchQuery" placeholder="Search summaries, keywords..." />
        </div>

        <div class="dropdowns">
          <select v-model="filters.Topic">
            <option value="">All Topics</option>
            <option v-for="t in uniqueTopics" :key="t" :value="t">{{ t }}</option>
          </select>

          <select v-model="filters.Target">
            <option value="">All Targets</option>
            <option v-for="t in uniqueTargets" :key="t" :value="t">{{ t }}</option>
          </select>

          <select v-model="filters.Jurisdiction">
            <option value="">All Jurisdictions</option>
            <option v-for="j in uniqueJurisdictions" :key="j" :value="j">{{ j }}</option>
          </select>

          <button v-if="hasActiveFilters" @click="clearFilters" class="btn-clear">Clear</button>
        </div>
      </div>

      <div v-if="loading" class="loading">Loading library...</div>

      <div v-else class="card-grid">
        <div v-for="(item, i) in filteredRows" :key="i" class="card">
          <div class="card-header">
            <div class="topic-group">
              <span v-for="t in item.parsedTopics" :key="t" class="badge-topic">{{ t }}</span>
            </div>
            <span class="date">{{ item.Date }}</span>
          </div>

          <div class="card-body">
            <h3>
              {{ item.Type }} to {{ item.Jurisdiction }} {{ item.Target }}
              <span class="author-byline">by {{ item.Author }}</span>
            </h3>
            <p class="summary">{{ item.Summary }}</p>
          </div>

          <div class="card-footer">
            <div class="tags">
              <span v-for="tag in item.parsedKeywords" :key="tag" class="tag">#{{ tag }}</span>
            </div>
            <a v-if="item['File Link']" :href="resolveLink(item['File Link'])" target="_blank" class="btn-link">
              View PDF
            </a>
          </div>
        </div>
      </div>

      <div v-if="!loading && filteredRows.length === 0" class="no-results">
        No documents match your search.
      </div>
    </main>

    <main v-else class="admin-panel">
      <div v-if="!isAuthorized" class="login-box">
        <p>Restricted Access</p>
        <button @click="handleLogin" class="btn-primary">Sign in with Google</button>
      </div>

      <div v-else class="form-box">
        <h3>Add New Entry</h3>
        <form @submit.prevent="submitEntry" class="admin-form">
          <template v-for="h in headers" :key="h">
            <div v-if="!HIDDEN_FIELDS.includes(h)" class="form-group">
              <label>{{ h }}</label>

              <input v-if="h === 'Date'" type="date" v-model="formData[h]" required />

              <select v-else-if="h === 'Type'" v-model="formData[h]" required>
                <option disabled value="">Select Type</option>
                <option v-for="opt in TYPE_OPTIONS" :key="opt" :value="opt">{{ opt }}</option>
              </select>

              <div v-else-if="h === 'Jurisdiction'">
                <input list="list-jurisdictions" v-model="formData[h]" placeholder="United States..." />
                <datalist id="list-jurisdictions">
                  <option v-for="item in uniqueJurisdictions" :key="item" :value="item" />
                </datalist>
              </div>

              <div v-else-if="h === 'Target'">
                <input list="list-targets" v-model="formData[h]" placeholder="EPA, PUC..." />
                <datalist id="list-targets">
                  <option v-for="item in uniqueTargets" :key="item" :value="item" />
                </datalist>
              </div>

              <div v-else-if="h === 'Author'">
                <input list="list-authors" v-model="formData[h]" placeholder="PFPI..." />
                <datalist id="list-authors">
                  <option v-for="item in uniqueAuthors" :key="item" :value="item" />
                </datalist>
              </div>

              <textarea v-else-if="isLongText(h)" v-model="formData[h]" rows="4"></textarea>

              <input v-else type="text" v-model="formData[h]" />
            </div>
          </template>

          <button type="submit" class="btn-success" :disabled="submitting">
            {{ submitting ? 'Saving...' : 'Save to Sheet' }}
          </button>
        </form>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, onMounted, reactive, computed } from 'vue';

// --- CONFIGURATION ---
// --- CONFIGURATION ---
// 1. For Public View: File > Share > Publish to web > CSV
const CSV_URL = import.meta.env.VITE_CSV_URL;
const CLIENT_ID = import.meta.env.VITE_GOOGLE_CLIENT_ID;
const SHEET_ID = import.meta.env.VITE_GOOGLE_SHEET_ID;
const HIDDEN_FIELDS = ['File Chip', '2nd Link', '3rd Link', 'Notes']; // Fields to hide from the form
const TYPE_OPTIONS = ["Comments", "Joint Comments", "Sign On"]; // Dropdown options
const AUTHOR_OPTIONS = ["PFPI", "JMP", "CURE", "MNPL", "MNEJT", "Sierra Club",
  "Cent. Bio. Div.", "CCANAF", "App Mtn Club", "Dogwood Alliance",
  "Michelle's Earth Foundation", "TWS"]
const JURISDICTION_OPTIONS = ["United States", "US - California", "US - Connecticut",
  "US - Massachusetts", "US - Minnesota", "US - New York",
  "US - Vermont", "US - Virginia"];
const TARGET_OPTIONS = ["Climate Action Council", "DEC", "DEEP", "DEP", "DOE", "EEA",
  "EPA", "Governor", "IRS", "President", "PSC", "PUC",
  "Representatives", "Senators", "USDA", "DEQ", "SASB",
  "RGGI", "USDI", "US F&W"]
// ---------------------------
// ---------------------

// If your CSV links are just filenames, add a base URL prefix here. 
// If they are full http links, leave this as empty string.
const FILE_BASE_URL = '';
// ---------------------

const view = ref('public');
const headers = ref([]);
const rows = ref([]);
const loading = ref(false);

// Filter State
const searchQuery = ref('');
const filters = reactive({
  Topic: '',
  Target: '',
  Jurisdiction: ''
});

// Admin State
const isAuthorized = ref(false);
const tokenClient = ref(null);
const formData = reactive({});
const submitting = ref(false);

onMounted(() => {
  loadPublicData();
  loadGoogleScript();
});

// --- DATA LOGIC ---
async function loadPublicData() {
  loading.value = true;
  try {
    const res = await fetch(CSV_URL);
    const text = await res.text();
    parseCSV(text);
  } catch (e) {
    console.error("CSV Load Error:", e);
  } finally {
    loading.value = false;
  }
}

function parseCSV(text) {
  const lines = text.split('\n').filter(l => l.trim());
  const rawHeaders = lines[0].split(',').map(h => h.trim());
  headers.value = rawHeaders;

  rows.value = lines.slice(1).map(line => {
    // FIX: This regex splits by comma ONLY if followed by an even number of quotes.
    // This allows "United States" to stay together, while still respecting "City, State" in quotes.
    const vals = line.split(/,(?=(?:(?:[^"]*"){2})*[^"]*$)/);

    // Clean up quotes and whitespace
    const cleanVals = vals.map(v => v.replace(/^"|"$/g, '').trim());

    let obj = {};
    rawHeaders.forEach((h, i) => obj[h] = cleanVals[i] || '');

    // Topics: Split by comma
    obj.parsedTopics = obj.Topic
      ? obj.Topic.split(',').map(t => t.trim())
      : [];

    // Keywords: Split by comma, limit to 3
    obj.parsedKeywords = obj.Keywords
      ? obj.Keywords.split(',').map(k => k.trim()).slice(0, 3)
      : [];

    return obj;
  });

  // Reset form data
  rawHeaders.forEach(h => formData[h] = '');
}

// --- COMPUTED PROPERTIES FOR FILTERS ---
const uniqueTopics = computed(() => {
  const allTopics = rows.value.flatMap(r => r.parsedTopics || []);
  return [...new Set(allTopics)].sort();
});
const uniqueTargets = computed(() => [...new Set(rows.value.map(r => r.Target).filter(Boolean))].sort());
const uniqueJurisdictions = computed(() => [...new Set(rows.value.map(r => r.Jurisdiction).filter(Boolean))].sort());
const uniqueAuthors = computed(() => [...new Set(rows.value.map(r => r.Author).filter(Boolean))].sort());

const hasActiveFilters = computed(() => {
  return searchQuery.value || filters.Topic || filters.Target || filters.Jurisdiction;
});

const filteredRows = computed(() => {
  return rows.value.filter(row => {
    const searchStr = searchQuery.value.toLowerCase();

    // 1. Text Search
    const matchesSearch = !searchStr ||
      (row.Summary && row.Summary.toLowerCase().includes(searchStr)) ||
      (row.Keywords && row.Keywords.toLowerCase().includes(searchStr)) ||
      (row.Target && row.Target.toLowerCase().includes(searchStr));

    // 2. Dropdown Filters
    // --- UPDATED: Check if the array INCLUDES the selected topic ---
    const matchesTopic = !filters.Topic || (row.parsedTopics && row.parsedTopics.includes(filters.Topic));
    const matchesTarget = !filters.Target || row.Target === filters.Target;
    const matchesJurisdiction = !filters.Jurisdiction || row.Jurisdiction === filters.Jurisdiction;

    return matchesSearch && matchesTopic && matchesTarget && matchesJurisdiction;
  });
});

function clearFilters() {
  searchQuery.value = '';
  filters.Topic = '';
  filters.Target = '';
  filters.Jurisdiction = '';
}

function resolveLink(link) {
  if (link.startsWith('http')) return link;
  return FILE_BASE_URL + link;
}

// --- ADMIN LOGIC (Unchanged) ---
function loadGoogleScript() {
  const script = document.createElement('script');
  script.src = 'https://accounts.google.com/gsi/client';
  script.onload = () => {
    tokenClient.value = google.accounts.oauth2.initTokenClient({
      client_id: CLIENT_ID,
      scope: 'https://www.googleapis.com/auth/spreadsheets',
      callback: (resp) => { if (resp.access_token) isAuthorized.value = true; },
    });
  };
  document.head.appendChild(script);
}

function handleLogin() { tokenClient.value.requestAccessToken({ prompt: 'consent' }); }

async function submitEntry() {
  submitting.value = true;
  const rowArray = headers.value.map(h => formData[h]);
  try {
    await fetch(`https://sheets.googleapis.com/v4/spreadsheets/${SHEET_ID}/values/Sheet1!A:A:append?valueInputOption=USER_ENTERED`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${gapi.client.getToken().access_token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ values: [rowArray] })
    });
    alert('Saved!');
    loadPublicData();
  } catch (e) { alert('Check console for auth errors'); console.error(e); }
  finally { submitting.value = false; }
}

const isLongText = (h) => /desc|comment|summary|notes/i.test(h);
</script>

<style>
/* Global Reset */
body {
  background: #f0f2f5;
  font-family: 'Segoe UI', sans-serif;
  margin: 0;
  color: #333;
}

/* Main Container - Optimized for Wide Screens */
.container {
  width: 98%;
  /* Occupy most of the screen */
  max-width: 1800px;
  /* Cap it for ultra-wide monitors */
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* Header */
.header {
  background: #1a202c;
  color: white;
  padding: 1rem 2rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.header h1 {
  margin: 0;
  font-size: 1.4rem;
  font-weight: 600;
}

.tabs button {
  background: none;
  border: none;
  color: #a0aec0;
  padding: 0.5rem 1rem;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  transition: color 0.2s;
}

.tabs button.active {
  color: white;
  border-bottom: 2px solid #63b3ed;
}

/* Filter Bar */
.filter-bar {
  background: white;
  padding: 1.2rem;
  margin: 1.5rem 0;
  border-radius: 8px;
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  align-items: center;
}

.search-box {
  flex-grow: 2;
  min-width: 300px;
}

/* Wider search box */
.search-box input {
  width: 100%;
  padding: 12px;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 1rem;
}

.dropdowns {
  display: flex;
  gap: 15px;
  flex-wrap: wrap;
  flex-grow: 1;
  justify-content: flex-end;
}

.dropdowns select {
  padding: 11px;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  background: #fff;
  min-width: 160px;
  cursor: pointer;
  font-size: 0.95rem;
}

.btn-clear {
  background: none;
  border: none;
  color: #e53e3e;
  cursor: pointer;
  font-size: 0.9rem;
  text-decoration: underline;
  padding: 0 10px;
}

/* Grid & Cards - Desktop First Approach */
.card-grid {
  display: grid;
  /* minmax(400px, 1fr) ensures cards are wide. On a 1080p screen this naturally creates 3 columns. */
  grid-template-columns: repeat(auto-fill, minmax(400px, 1fr));
  gap: 2rem;
  padding: 0 0 4rem 0;
}

.card {
  background: white;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.02);
  border: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
  transform: translateY(-3px);
  box-shadow: 0 12px 20px rgba(0, 0, 0, 0.08);
  border-color: #cbd5e0;
}

.card-header {
  background: #f8fafc;
  padding: 16px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #edf2f7;
}

.badge-topic {
  background: #ebf8ff;
  color: #2c5282;
  padding: 5px 12px;
  border-radius: 20px;
  font-size: 0.8rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.date {
  font-size: 0.9rem;
  color: #718096;
}

.card-body {
  padding: 20px;
  flex-grow: 1;
}

.card-body h3 {
  margin: 0 0 10px 0;
  font-size: 1.3rem;
  color: #1a202c;
  line-height: 1.3;
}

.subtitle {
  color: #718096;
  font-weight: 400;
  font-size: 1rem;
  margin-left: 6px;
}

.summary {
  font-size: 1rem;
  color: #4a5568;
  line-height: 1.6;
  margin-top: 10px;
}

.card-footer {
  padding: 16px 20px;
  border-top: 1px solid #edf2f7;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #fff;
}

.tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.tag {
  font-size: 0.8rem;
  color: #4a5568;
  background: #edf2f7;
  padding: 4px 8px;
  border-radius: 4px;
}

.btn-link {
  background: #3182ce;
  color: white;
  padding: 8px 18px;
  border-radius: 6px;
  text-decoration: none;
  font-size: 0.95rem;
  font-weight: 500;
  transition: background 0.2s;
}

.btn-link:hover {
  background: #2b6cb0;
}

/* Admin Styles (Centered) */
.admin-panel {
  padding: 4rem 2rem;
  display: flex;
  justify-content: center;
}

.form-box,
.login-box {
  width: 100%;
  max-width: 700px;
  background: white;
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  border: 1px solid #e2e8f0;
}

.admin-form {
  display: grid;
  gap: 1.5rem;
}

.form-group label {
  font-weight: 600;
  display: block;
  margin-bottom: 6px;
  color: #2d3748;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid #cbd5e0;
  border-radius: 6px;
  font-size: 1rem;
  box-sizing: border-box;
}

.btn-success {
  background: #38a169;
  color: white;
  border: none;
  padding: 14px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 600;
  width: 100%;
}

.btn-success:disabled {
  background: #cbd5e0;
  cursor: not-allowed;
}

.btn-primary {
  background: #3182ce;
  color: white;
  padding: 12px 24px;
  border-radius: 6px;
  border: none;
  cursor: pointer;
  font-size: 1rem;
}

.loading,
.no-results {
  text-align: center;
  padding: 4rem;
  color: #718096;
  font-size: 1.2rem;
}

/* Add this to your <style> block */
.topic-group {
  display: flex;
  gap: 6px;
  /* Space between badges */
  flex-wrap: wrap;
}

/* Add to <style> */
.author-byline {
  font-weight: 400;
  color: #718096;
  font-size: 0.9em;
  margin-left: 4px;
}

/* Update this rule in your <style> block */
.form-group input,
.form-group textarea,
.form-group select {
  /* Added select */
  width: 100%;
  padding: 12px;
  border: 1px solid #cbd5e0;
  border-radius: 6px;
  font-size: 1rem;
  box-sizing: border-box;
  background: white;
  /* Ensures date/select backgrounds are white */
}
</style>