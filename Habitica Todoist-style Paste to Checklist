// ==UserScript==
// @name         Habitica Todoist-style Paste to Checklist
// @namespace    habitica-paste-checklist
// @version      1.3.2
// @description  Paste a multiline list into Habitica to create a To-Do with checklist items, or append it to the task being edited. Uses your existing Habitica session automatically.
// @author       You
// @match        https://habitica.com/*
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_deleteValue
// @grant        GM_registerMenuCommand
// @run-at       document-start
// ==/UserScript==

(function () {
  'use strict';

  const API_BASE = 'https://habitica.com/api/v3';
  // These were empty strings before, which broke GM_getValue/GM_setValue.
  const USER_ID_KEY = 'habitica_user_id';
  const TOKEN_KEY = 'habitica_api_token';

  const css = `
    #hk-paste-toast {
      position: fixed;
      left: 50%;
      bottom: 28px;
      transform: translateX(-50%);
      z-index: 2147483000;
      background: #fff;
      color: #222;
      border: 1px solid #bbb;
      border-radius: 10px;
      padding: 12px 14px;
      font: 14px/1.45 system-ui, sans-serif;
      box-shadow: 0 8px 30px rgba(0,0,0,.25);
      display: none;
      max-width: min(620px, calc(100vw - 32px));
    }
    #hk-paste-toast.show { display: block; }
    #hk-paste-toast .hk-title { font-weight: 700; margin-bottom: 6px; }
    #hk-paste-toast .hk-preview {
      max-height: 110px;
      overflow: auto;
      background: #f5f5f5;
      border-radius: 6px;
      padding: 8px;
      margin: 8px 0;
      white-space: pre-wrap;
    }
    #hk-paste-toast .hk-actions {
      display: flex;
      flex-wrap: wrap;
      gap: 8px;
      margin-top: 8px;
    }
    #hk-paste-toast button {
      border: 0;
      border-radius: 6px;
      padding: 8px 12px;
      font-weight: 600;
      cursor: pointer;
    }
    .hk-create { background: #4f2a93; color: #fff; }
    .hk-append { background: #e8eaf6; color: #283593; }
    .hk-cancel { background: #eee; color: #333; }
    #hk-paste-status { margin-top: 8px; font-size: 13px; color: #444; }
    #hk-paste-status.error { color: #b3261e; }
  `;

  function injectStyle() {
    const style = document.createElement('style');
    style.textContent = css;
    (document.head || document.documentElement).appendChild(style);
  }

  function escapeHtml(str) {
    return str
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  }

  // Habitica's web app keeps its login in localStorage under "habit-mobile",
  // as { auth: { apiId, apiToken }, ... }. We check that first, then fall back
  // to searching every localStorage entry for an apiId/apiToken pair.
  function findCredsInObject(obj, depth = 0) {
    if (!obj || typeof obj !== 'object' || depth > 5) return null;

    const userId = obj.apiId || obj.userId;
    const token = obj.apiToken;
    if (typeof userId === 'string' && typeof token === 'string' && userId && token) {
      return { userId, token };
    }

    for (const key of Object.keys(obj)) {
      const found = findCredsInObject(obj[key], depth + 1);
      if (found) return found;
    }
    return null;
  }

  function readCredentialsFromHabiticaSession() {
    try {
      const raw = localStorage.getItem('habit-mobile');
      if (raw) {
        const found = findCredsInObject(JSON.parse(raw));
        if (found) return found;
      }
    } catch (e) {
      // fall through
    }

    try {
      for (let i = 0; i < localStorage.length; i++) {
        const raw = localStorage.getItem(localStorage.key(i));
        if (!raw || raw.charAt(0) !== '{' || raw.indexOf('apiToken') === -1) continue;
        try {
          const found = findCredsInObject(JSON.parse(raw));
          if (found) return found;
        } catch (e) {
          // ignore non-JSON entries
        }
      }
    } catch (e) {
      // ignore
    }

    return null;
  }

  function getCredentials() {
    const sessionCredentials = readCredentialsFromHabiticaSession();

    if (sessionCredentials) {
      // Backup copy in case Habitica changes where it stores the session.
      GM_setValue(USER_ID_KEY, sessionCredentials.userId);
      GM_setValue(TOKEN_KEY, sessionCredentials.token);
      return sessionCredentials;
    }

    const userId = GM_getValue(USER_ID_KEY);
    const token = GM_getValue(TOKEN_KEY);

    if (userId && token) return { userId, token };

    return null;
  }

  async function habiticaRequest(method, path, body) {
    const credentials = getCredentials();

    if (!credentials) {
      throw new Error(
        'Could not find your Habitica login in this browser. Make sure you are logged in at habitica.com, then reload the page.'
      );
    }

    const doFetch = () =>
      fetch(`${API_BASE}${path}`, {
        method,
        headers: {
          'Content-Type': 'application/json',
          'x-api-user': credentials.userId,
          'x-api-key': credentials.token,
          'x-client': `${credentials.userId}-Tampermonkey-PasteChecklist`
        },
        body: body ? JSON.stringify(body) : undefined
      });

    let response = await doFetch();

    // If rate limited, wait as long as Habitica asks (max 2 retries).
    for (let attempt = 0; attempt < 2 && response.status === 429; attempt++) {
      const retryAfter = parseFloat(response.headers.get('retry-after'));
      const waitMs = Math.min((isNaN(retryAfter) ? 10 : retryAfter) + 1, 60) * 1000;
      await new Promise(resolve => setTimeout(resolve, waitMs));
      response = await doFetch();
    }

    const data = await response.json().catch(() => ({}));

    if (!response.ok) {
      const message =
        data?.message ||
        data?.error?.message ||
        `Habitica API returned HTTP ${response.status}`;
      throw new Error(message);
    }

    return data;
  }

  function parseClipboardLines(text) {
    return text
      .split(/\r?\n/)
      .map(line => line.trim())
      .filter(Boolean);
  }

  function getTaskTitleFromFocusedField(target) {
    // Prefer the title box of the open task-edit window. The paste may have
    // happened in the notes box or a checklist box, whose text is NOT the title.
    const modal =
      target?.closest?.('.modal, [role="dialog"]') ||
      document.querySelector('.modal.show, [role="dialog"]');

    if (modal) {
      const selectors = [
        'input.title-input',
        '[class*="title"] input[type="text"]',
        'input[type="text"]'
      ];

      for (const selector of selectors) {
        for (const el of modal.querySelectorAll(selector)) {
          const v = (el.value || '').trim();
          if (v && !v.includes('\n')) return v;
        }
      }
    }

    const value = target?.value || target?.textContent || '';
    return value.trim().split('\n')[0].trim();
  }

  async function findTaskByTitle(title) {
    const response = await habiticaRequest('GET', '/tasks/user');
    const tasks = (response.data || []).filter(
      task => task.type === 'todo' || task.type === 'daily'
    );
    const wanted = title.trim().toLowerCase();
    const norm = task => (task.text || '').trim().toLowerCase();

    return (
      tasks.find(task => norm(task) === wanted) ||
      tasks.find(task => norm(task).startsWith(wanted)) ||
      tasks.find(task => norm(task).includes(wanted))
    );
  }

  async function createTodoWithChecklist(title, checklist, status) {
    status.textContent = `Creating “${title}” with ${checklist.length} checklist item(s)…`;

    const response = await habiticaRequest('POST', '/tasks/user', {
      type: 'todo',
      text: title,
      checklist: checklist.map(text => ({ text, completed: false })),
      collapseChecklist: true
    });

    return `Created “${response.data.text}” with ${checklist.length} checklist item(s).`;
  }

  async function appendChecklistToTask(task, lines, status) {
    status.textContent = `Appending ${lines.length} item(s) to “${task.text}”…`;

    // One single request instead of one per line, to stay under Habitica's
    // rate limit (30 requests/minute). Existing items are kept as they are.
    const existing = (task.checklist || []).map(item => ({
      id: item.id,
      text: item.text,
      completed: !!item.completed
    }));
    const added = lines.map(text => ({ text, completed: false }));

    await habiticaRequest('PUT', `/tasks/${task.id}`, {
      checklist: existing.concat(added)
    });

    return `Added ${lines.length} checklist item(s) to “${task.text}”.`;
  }

  function showToast(lines, focusedElement) {
    const existing = document.getElementById('hk-paste-toast');
    if (existing) existing.remove();

    const toast = document.createElement('div');
    toast.id = 'hk-paste-toast';

    const firstLine = lines[0];
    const remaining = lines.slice(1);
    const currentTitle = getTaskTitleFromFocusedField(focusedElement);

    const preview = lines
      .slice(0, 6)
      .map(line => `• ${escapeHtml(line)}`)
      .join('\n');

    toast.innerHTML = `
      <div class="hk-title">Import ${lines.length} pasted line(s)?</div>
      <div class="hk-preview">${preview}${lines.length > 6 ? '\n…' : ''}</div>
      <div class="hk-actions">
        <button class="hk-create" data-action="create">
          Create To-Do + ${remaining.length || lines.length} checklist item(s)
        </button>
        <button class="hk-append" data-action="append">
          Append to “${escapeHtml(currentTitle || 'current task')}”
        </button>
        <button class="hk-cancel" data-action="cancel">Cancel</button>
      </div>
      <div id="hk-paste-status"></div>
    `;

    document.body.appendChild(toast);
    requestAnimationFrame(() => toast.classList.add('show'));

    toast.addEventListener('click', async event => {
      const action = event.target?.dataset?.action;
      if (!action) return;

      const status = document.getElementById('hk-paste-status');
      status.className = '';

      if (action === 'cancel') {
        toast.remove();
        return;
      }

      if (action === 'reload') {
        location.reload();
        return;
      }

      try {
        if (action === 'create') {
          const checklist = remaining.length ? remaining : [firstLine];
          const message = await createTodoWithChecklist(firstLine, checklist, status);

          status.textContent = message;
          setTimeout(() => toast.remove(), 2500);
          return;
        }

        if (action === 'append') {
          const title = getTaskTitleFromFocusedField(focusedElement) || currentTitle;

          if (!title) {
            status.textContent =
              'Could not identify the task being edited. Open the task’s edit window and paste again.';
            status.className = 'error';
            return;
          }

          const task = await findTaskByTitle(title);

          if (!task) {
            status.textContent = `No existing task exactly titled “${title}”.`;
            status.className = 'error';
            return;
          }

          const message = await appendChecklistToTask(task, lines, status);
          status.textContent =
            message +
            ' Reload to see them, and don’t press Save in the open task window or it may overwrite them.';
          toast.querySelector('.hk-actions').innerHTML =
            '<button class="hk-create" data-action="reload">Reload page now</button>' +
            '<button class="hk-cancel" data-action="cancel">Close</button>';
        }
      } catch (error) {
        status.textContent = `Error: ${error.message}`;
        status.className = 'error';
      }
    });

    setTimeout(() => {
      document.addEventListener(
        'click',
        function closeOnOutsideClick(event) {
          if (!toast.contains(event.target)) {
            toast.remove();
          }
        },
        { once: true }
      );
    }, 0);
  }

  function shouldOfferImport(lines) {
    return lines.length >= 2;
  }

  // Optional manual fallback (Tampermonkey menu). Not needed if you're logged in.
  function registerManualCredentialFallback() {
    GM_registerMenuCommand('Set Habitica credentials manually', () => {
      const userId = window.prompt('Enter your Habitica User ID:');
      if (!userId) return;

      const token = window.prompt('Enter your Habitica API token:');
      if (!token) return;

      GM_setValue(USER_ID_KEY, userId.trim());
      GM_setValue(TOKEN_KEY, token.trim());
      window.alert('Credentials saved.');
    });

    GM_registerMenuCommand('Clear saved Habitica credentials', () => {
      GM_deleteValue(USER_ID_KEY);
      GM_deleteValue(TOKEN_KEY);
      window.alert('Saved credentials cleared.');
    });
  }

  document.addEventListener(
    'paste',
    event => {
      const target = event.target;

      const isEditable =
        target instanceof HTMLInputElement ||
        target instanceof HTMLTextAreaElement ||
        target?.isContentEditable;

      if (!isEditable) return;

      const text = event.clipboardData?.getData('text/plain') || '';
      const lines = parseClipboardLines(text);

      if (!shouldOfferImport(lines)) return;

      showToast(lines, target);
    },
    true
  );

  // document.head doesn't exist yet at document-start, so wait for it.
  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', injectStyle);
  } else {
    injectStyle();
  }
  registerManualCredentialFallback();
})();
