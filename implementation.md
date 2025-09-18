# Google Authentication Implementation Plan

## Overview
This document outlines the implementation of Google Sign-In authentication for the KLZ TTD task management app, including the addition of a completed tasks tab.

## Architecture
- **Frontend**: Single-page HTML app with Google Identity Services (GIS)
- **Backend**: n8n workflow with JWT token validation
- **Security**: Google ID tokens validated on each API request
- **Storage**: sessionStorage for tokens (clears on tab close)

## Phase 1: Google Cloud Setup

### Step 1.1: Create Google Cloud Project
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Click "Select a project" → "New Project"
3. Enter project name: "KLZ TTD Auth"
4. Click "Create"

### Step 1.2: Enable APIs
1. Go to "APIs & Services" → "Library"
2. Search for "Google Identity"
3. Click "Google Identity and Access Management (IAM) API"
4. Click "Enable"

### Step 1.3: Create OAuth 2.0 Credentials
1. Go to "APIs & Services" → "Credentials"
2. Click "+ Create Credentials" → "OAuth client ID"
3. If prompted, configure OAuth consent screen:
   - Choose "External" user type
   - App name: "KLZ TTD"
   - User support email: your email
   - Developer contact: your email
4. Application type: "Web application"
5. Name: "KLZ TTD Web Client"
6. Authorized JavaScript origins:
   - `http://localhost:8080` (for local testing)
   - `https://your-domain.com` (for production)
7. Click "Create"
8. **Copy the Client ID** - you'll need this for the frontend

## Phase 2: Frontend Implementation

### Step 2.1: Add Google Sign-In Library
Add to `<head>` section of index.html:

```html
<!-- Google Identity Services -->
<script src="https://accounts.google.com/gsi/client" async defer></script>
<meta name="google-signin-client_id" content="YOUR_GOOGLE_CLIENT_ID">
```

### Step 2.2: Add Authentication UI
Add after the `<header>` section:

```html
<!-- Login Overlay -->
<div id="loginOverlay" class="login-overlay" style="display:none;">
  <div class="login-card">
    <h2>Sign in to KLZ TTD</h2>
    <p>Please sign in with your Google account to access your tasks.</p>
    <div id="g_id_onload"
         data-client_id="YOUR_GOOGLE_CLIENT_ID"
         data-callback="handleCredentialResponse"
         data-auto_prompt="false">
    </div>
    <div class="g_id_signin"
         data-type="standard"
         data-size="large"
         data-theme="outline"
         data-text="sign_in_with"
         data-shape="rectangular"
         data-logo_alignment="left">
    </div>
  </div>
</div>

<!-- User Info in Header -->
<div id="userInfo" class="user-info" style="display:none;">
  <img id="userAvatar" class="user-avatar" src="" alt="User Avatar">
  <span id="userEmail" class="user-email"></span>
  <button id="signOutBtn" class="btn">Sign Out</button>
</div>
```

### Step 2.3: Add Authentication Styles
Add to CSS section:

```css
/* Login Overlay */
.login-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.login-card {
  background: var(--card);
  padding: 32px;
  border-radius: 16px;
  text-align: center;
  max-width: 400px;
  box-shadow: var(--shadow-lg);
}

.login-card h2 {
  margin: 0 0 16px;
  font-size: 24px;
  font-weight: 700;
}

.login-card p {
  color: var(--muted);
  margin: 0 0 24px;
  line-height: 1.5;
}

/* User Info */
.user-info {
  display: flex;
  align-items: center;
  gap: 12px;
}

.user-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
}

.user-email {
  font-size: 14px;
  color: var(--muted);
}
```

### Step 2.4: Add Authentication JavaScript
Replace the existing JavaScript section with:

```javascript
// CONFIGURATION
const API_CONFIG = {
  GET_TASKS_URL: 'https://n8n.chouseservice.com/webhook/get-tasks',
  GET_NOTES_URL: 'https://n8n.chouseservice.com/webhook/get-notes',
  COMPLETE_TASK_URL: 'https://n8n.chouseservice.com/webhook/complete-task',
  SECRET: 'klz-ttd'
};

const GOOGLE_CLIENT_ID = 'YOUR_GOOGLE_CLIENT_ID'; // Replace with your actual Client ID

let STATE = {
  tasks: [],
  completedTasks: [],
  notes: [],
  mode: 'tasks',
  current: null,
  user: null,
  token: null
};

const $ = sel => document.querySelector(sel);
const $$ = sel => Array.from(document.querySelectorAll(sel));

// Authentication Functions
function handleCredentialResponse(response) {
  const token = response.credential;

  // Decode JWT to get user info (basic decode, validation happens server-side)
  const payload = JSON.parse(atob(token.split('.')[1]));

  STATE.user = {
    email: payload.email,
    name: payload.name,
    picture: payload.picture
  };
  STATE.token = token;

  // Store in session storage
  sessionStorage.setItem('google_token', token);
  sessionStorage.setItem('user_info', JSON.stringify(STATE.user));

  showMainApp();
  loadTasks();
}

function checkAuth() {
  const token = sessionStorage.getItem('google_token');
  const userInfo = sessionStorage.getItem('user_info');

  if (token && userInfo) {
    STATE.token = token;
    STATE.user = JSON.parse(userInfo);
    showMainApp();
    return true;
  }

  showLogin();
  return false;
}

function showLogin() {
  $('#loginOverlay').style.display = 'flex';
  $('#main').style.display = 'none';
  $('#header').style.display = 'none';
}

function showMainApp() {
  $('#loginOverlay').style.display = 'none';
  $('#main').style.display = 'block';
  $('#header').style.display = 'block';

  // Update user info in header
  $('#userAvatar').src = STATE.user.picture;
  $('#userEmail').textContent = STATE.user.email;
  $('#userInfo').style.display = 'flex';
}

function signOut() {
  sessionStorage.removeItem('google_token');
  sessionStorage.removeItem('user_info');
  STATE.user = null;
  STATE.token = null;
  google.accounts.id.disableAutoSelect();
  showLogin();
}

// API Functions with Authentication
async function apiCall(url, options = {}) {
  if (!STATE.token) {
    showLogin();
    throw new Error('Not authenticated');
  }

  const headers = {
    'Authorization': `Bearer ${STATE.token}`,
    'X-User-ID': STATE.user.email,
    ...options.headers
  };

  try {
    const response = await fetch(url, {
      ...options,
      headers
    });

    if (response.status === 401) {
      // Token expired or invalid
      signOut();
      throw new Error('Authentication failed');
    }

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    console.error('API call failed:', error);
    throw error;
  }
}

async function loadTasks() {
  const wrap = $('#viewTasks');
  wrap.innerHTML = '<div class="loading">Loading tasks</div>';

  try {
    const data = await apiCall(API_CONFIG.GET_TASKS_URL);
    STATE.tasks = data.active || data.rows || [];
    STATE.completedTasks = data.completed || [];
    renderTasks();
  } catch (err) {
    console.error('Failed to load tasks:', err);
    wrap.innerHTML = '<div class="error">Failed to load tasks.<br>Check your connection and try again.</div>';
    toast('Connection error');
  }
}

async function loadNotes() {
  const wrap = $('#viewNotes');
  wrap.innerHTML = '<div class="loading">Loading notes</div>';

  try {
    const data = await apiCall(API_CONFIG.GET_NOTES_URL);
    STATE.notes = data.rows || [];
    renderNotes();
  } catch (err) {
    console.error('Failed to load notes:', err);
    wrap.innerHTML = '<div class="error">Failed to load notes.<br>Check your connection and try again.</div>';
    toast('Connection error');
  }
}

async function completeTask(task_id) {
  try {
    const result = await apiCall(API_CONFIG.COMPLETE_TASK_URL, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        task_id: task_id,
        secret: API_CONFIG.SECRET
      })
    });

    if (result.ok) {
      toast('✓ Done');
      return true;
    } else {
      toast('Failed to update');
      return false;
    }
  } catch (err) {
    console.error('Failed to complete task:', err);
    toast('Failed to update');
    return false;
  }
}

// Event Listeners
$('#signOutBtn').onclick = signOut;

// Initialize
window.onload = function() {
  // Check authentication on page load
  if (!checkAuth()) {
    return;
  }

  // Continue with app initialization
  loadTasks();
};
```

### Step 2.5: Add Completed Tasks Tab
Update the tabs section in HTML:

```html
<div class="tabs">
  <div id="tabTasks" class="tab active">Tasks</div>
  <div id="tabCompleted" class="tab">Completed</div>
  <div id="tabNotes" class="tab">Notes</div>
</div>
```

Add completed tasks section:

```html
<!-- COMPLETED TASKS LIST -->
<section id="viewCompleted" class="list" style="display:none;">
  <div class="loading">Loading completed tasks</div>
</section>
```

## Phase 3: Backend Implementation (n8n Workflow)

### Step 3.1: Add Token Validation Node
For each webhook (GET Tasks, GET Notes, Complete Task), add these nodes after the webhook trigger:

1. **Code Node: Extract Token**
```javascript
const headers = $input.first().json.headers || {};
const authHeader = headers.authorization || headers.Authorization || '';
const userId = headers['x-user-id'] || headers['X-User-ID'] || '';

if (!authHeader.startsWith('Bearer ')) {
  return [{
    json: {
      error: 'Missing or invalid Authorization header',
      httpCode: 401,
      continueFlow: false
    }
  }];
}

if (!userId) {
  return [{
    json: {
      error: 'Missing X-User-ID header',
      httpCode: 401,
      continueFlow: false
    }
  }];
}

const token = authHeader.substring(7);

return [{
  json: {
    ...($input.first().json),
    authToken: token,
    userId: userId,
    continueFlow: true
  }
}];
```

2. **HTTP Request Node: Validate Token**
- Method: GET
- URL: `https://oauth2.googleapis.com/tokeninfo`
- Query Parameters: `id_token={{$json.authToken}}`

3. **IF Node: Check Token Validity**
- Condition: `{{$json.email}}` is not empty
- True path: Continue to existing logic
- False path: Return 401 error

### Step 3.2: Update Task Response Format
Modify the "Format Tasks Response" code node:

```javascript
const items = $input.all();
const rows = items.map(item => item.json);

// Separate active and completed tasks
const activeTasks = rows.filter(row =>
  row.status !== 'completed' &&
  row.status !== 'done'
);

const completedTasks = rows.filter(row =>
  row.status === 'completed' ||
  row.status === 'done'
);

return [{
  json: {
    ok: true,
    active: activeTasks,
    completed: completedTasks
  }
}];
```

## Phase 4: Testing & Deployment

### Step 4.1: Local Testing
1. Update the Google Client ID in index.html
2. Test authentication flow
3. Verify token validation in n8n
4. Test completed tasks functionality

### Step 4.2: Deployment
1. Update authorized origins in Google Cloud Console
2. Deploy to production domain
3. Test end-to-end functionality

## Security Considerations

1. **Token Expiration**: Google ID tokens expire after 1 hour
2. **HTTPS Required**: Google Sign-In requires HTTPS in production
3. **Domain Validation**: Tokens are tied to specific domains
4. **Server-Side Validation**: Always validate tokens server-side (n8n)

## Configuration Summary

**Required Updates:**
- Replace `YOUR_GOOGLE_CLIENT_ID` with actual Client ID (2 places)
- Update authorized JavaScript origins in Google Cloud Console
- Test both localhost and production domains

**n8n Workflow ID**: kMMvXbFQ43GX0ghd

This implementation provides secure, seamless authentication while maintaining the single-page architecture and adding the completed tasks functionality.