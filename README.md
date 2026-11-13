# Habitica Paste to Checklist

A userscript that lets you paste a multi-line list into [Habitica](https://habitica.com) and turn it into a To-Do with a checklist, or add it to a task you're already editing. It works the way pasting a list into Todoist does.

It uses your existing Habitica login in the browser, so there are no API keys to copy in.

> **Unofficial.** This project is not affiliated with or endorsed by Habitica.

## Features

- **Create a To-Do from a pasted list.** The first line becomes the task title and the remaining lines become checklist items. The checklist is collapsed by default.
- **Append to an existing task.** Paste a list while editing a task and every line is added to that task's checklist. Existing checklist items are kept.
- **Automatic login detection.** Reads your session from Habitica's own browser storage. There is nothing to configure if you're logged in.
- **Preview before importing.** A small prompt shows the lines it found and asks what you want to do before anything is sent.
- **Rate-limit friendly.** Appending uses a single API request however long the list is, and the script retries automatically if Habitica asks it to slow down.

## Install

1. Install a userscript manager such as [Tampermonkey](https://www.tampermonkey.net/). The script is written for Tampermonkey; other managers that support the `GM_*` functions may also work.
2. Create a new script in your manager and paste in the contents of `habitica-paste-to-checklist.user.js`. If you install it from a URL on GitHub, open the raw `.user.js` file and your manager will offer to install it.
3. Reload habitica.com.

## Usage

### Create a new To-Do

1. Copy a list with two or more lines, for example:

   ```
   Pack for trip
   Passport
   Charger
   Toothbrush
   ```

2. Paste it into any text field on Habitica, such as the "Add a To-Do" box.
3. In the prompt, click **Create To-Do + N checklist item(s)**.

The first line, `Pack for trip`, becomes the title and the other lines become the checklist.

### Add to an existing task

1. Open the task you want to change (click it to open the edit window).
2. Paste your list into a field in that window.
3. Click **Append to “Task name”**.
4. Click **Reload page now** when it finishes.

Every pasted line, including the first, is added as a checklist item.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 760 330" width="760" height="330" role="img" aria-labelledby="t d">
  <title id="t">How it works</title>
  <desc id="d">Paste a multi-line list into Habitica, choose in the prompt whether to create a new To-Do or append to the open task, and the checklist is created.</desc>
  <defs>
    <marker id="a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0L10 5L0 10z" fill="#6b6f80"/>
    </marker>
  </defs>
  <rect width="760" height="330" rx="14" fill="#eceef5"/>
  <g font-family="system-ui, -apple-system, Segoe UI, Roboto, sans-serif" font-size="14" text-anchor="middle">
    <rect x="30" y="115" width="170" height="80" rx="10" fill="#ffffff" stroke="#bbbbbb"/>
    <text x="115" y="147" font-weight="700" fill="#222222">1. Copy a list</text>
    <text x="115" y="169" fill="#555555">2+ lines, then paste</text>
    <text x="115" y="187" fill="#555555">into Habitica</text>

    <rect x="295" y="115" width="170" height="80" rx="10" fill="#ffffff" stroke="#bbbbbb"/>
    <text x="380" y="147" font-weight="700" fill="#222222">2. Prompt appears</text>
    <text x="380" y="169" fill="#555555">Preview of the lines,</text>
    <text x="380" y="187" fill="#555555">pick an action</text>

    <rect x="560" y="45" width="170" height="80" rx="10" fill="#4f2a93"/>
    <text x="645" y="77" font-weight="700" fill="#ffffff">Create To-Do</text>
    <text x="645" y="99" fill="#e6dcf7">1st line = title</text>
    <text x="645" y="117" fill="#e6dcf7">rest = checklist</text>

    <rect x="560" y="185" width="170" height="80" rx="10" fill="#e8eaf6" stroke="#9fa8da"/>
    <text x="645" y="217" font-weight="700" fill="#283593">Append to task</text>
    <text x="645" y="239" fill="#3b4aa0">every line becomes</text>
    <text x="645" y="257" fill="#3b4aa0">a checklist item</text>

    <path d="M200 155H293" stroke="#6b6f80" stroke-width="2" fill="none" marker-end="url(#a)"/>
    <path d="M465 140L558 100" stroke="#6b6f80" stroke-width="2" fill="none" marker-end="url(#a)"/>
    <path d="M465 170L558 210" stroke="#6b6f80" stroke-width="2" fill="none" marker-end="url(#a)"/>
    <text x="380" y="300" font-size="13" fill="#555555">After appending, reload the page to see the new items.</text>
  </g>
</svg>

> **Don't press Save in the open edit window after appending.** The window still shows the old checklist and saving it can overwrite the items you just added. Reload the page instead.

## How it finds your login

The script reads your session from the `habit-mobile` entry in Habitica's browser storage. If that isn't found, it searches your other stored entries for a user ID and API token pair.

Whenever it finds them, it keeps a backup copy in your userscript manager's own storage, in case Habitica changes where it stores your session. Credentials are only ever sent to `https://habitica.com/api/v3`.

As a last resort, the userscript menu has **Set Habitica credentials manually** and **Clear saved Habitica credentials** commands. You shouldn't need them while you're logged in.

## Limitations

- The prompt appears only when you paste **two or more non-empty lines**. Single-line pastes behave normally.
- Appending matches the task by its title, using the title box of the open edit window. It looks at To-Dos and Dailies only, and tries an exact match first, then "starts with", then "contains". If you have several tasks with the same title, it may pick the wrong one.
- Pasted text still goes into the field you pasted into as usual. If you used the prompt, you can discard that text by reloading the page.
- Habitica limits API usage to about 30 requests per minute. Each import uses very few requests, but repeated pastes in quick succession can still hit the limit.

## Troubleshooting

| Problem | What to try |
| --- | --- |
| "Could not find your Habitica login" | Make sure you're logged in at habitica.com and reload the page. |
| "Could not identify the task being edited" | Open the task's edit window and paste again. |
| "No existing task exactly titled …" | Check the task title for typos or recent edits. Unsaved title changes won't match the saved task. |
| "Rate limited" errors | Wait a minute and try again. The script already retries on its own. |
| Added items don't show up | Reload the page. The open edit window doesn't update itself. |

If automatic login detection fails, open the browser console on habitica.com and run `localStorage.getItem('habit-mobile')?.slice(0,200)` to see how your session is stored. Remove your ID and token before sharing the output in an issue.

## Privacy and security

- The script runs only on `https://habitica.com/*`.
- It never sends your credentials anywhere except the Habitica API.
- Treat your API token like a password. Don't share screenshots or logs that include it, and don't commit it to a repository.

## Contributing

Issues and pull requests are welcome. When reporting a bug, include your userscript manager and browser, what you pasted (with anything private removed), and the message shown in the prompt.
