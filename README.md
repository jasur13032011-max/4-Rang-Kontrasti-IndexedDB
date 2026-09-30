# 4-Rang-Kontrasti-IndexedDB
Albatta. Mana **AccessBoard** uchun to‘liq ishlaydigan variant. Bu versiyada IndexedDB, `createObjectStore`, `put`, `getAll`, reloaddan keyin tiklash va rang + matn/belgi accessibility talablari bor.

### 1. `index.html`

```html
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AccessBoard</title>
    <link rel="stylesheet" href="./style.css">
</head>
<body>

<header class="header">
    <h1>AccessBoard</h1>
    <p>Accessible Kanban Board</p>
</header>

<main class="container">

    <section class="add-card">
        <h2>Yangi karta qo'shish</h2>

        <form id="cardForm">
            <label for="title">Karta nomi</label>
            <input
                type="text"
                id="title"
                placeholder="Masalan: Login sahifasi"
                required
            >

            <label for="description">Tavsif</label>
            <textarea
                id="description"
                placeholder="Karta haqida..."
                required
            ></textarea>

            <label for="status">Holat</label>
            <select id="status">
                <option value="todo">🔴 Kutilmoqda</option>
                <option value="progress">🟡 Jarayonda</option>
                <option value="done">🟢 Bajarildi</option>
            </select>

            <button type="submit">Karta qo'shish</button>
        </form>
    </section>

    <section class="board">

        <div class="column">
            <h2>🔴 Kutilmoqda</h2>
            <div id="todo" class="cards"></div>
        </div>

        <div class="column">
            <h2>🟡 Jarayonda</h2>
            <div id="progress" class="cards"></div>
        </div>

        <div class="column">
            <h2>🟢 Bajarildi</h2>
            <div id="done" class="cards"></div>
        </div>

    </section>

</main>

<script type="module" src="./app/main.js"></script>

</body>
</html>
```

### 2. `app/db.js`

```javascript
const DB_NAME = "AccessBoardDB";
const DB_VERSION = 1;
const STORE_NAME = "cards";

export function openDB() {
    return new Promise((resolve, reject) => {
        const request = indexedDB.open(DB_NAME, DB_VERSION);

        request.onupgradeneeded = (event) => {
            const db = event.target.result;

            if (!db.objectStoreNames.contains(STORE_NAME)) {
                db.createObjectStore(STORE_NAME, {
                    keyPath: "id",
                    autoIncrement: true
                });
            }
        };

        request.onsuccess = () => {
            resolve(request.result);
        };

        request.onerror = () => {
            reject(request.error);
        };
    });
}

export async function addCard(card) {
    const db = await openDB();

    return new Promise((resolve, reject) => {
        const transaction = db.transaction(STORE_NAME, "readwrite");
        const store = transaction.objectStore(STORE_NAME);

        const request = store.put(card);

        request.onsuccess = () => {
            resolve(request.result);
        };

        request.onerror = () => {
            reject(request.error);
        };
    });
}

export async function getAllCards() {
    const db = await openDB();

    return new Promise((resolve, reject) => {
        const transaction = db.transaction(STORE_NAME, "readonly");
        const store = transaction.objectStore(STORE_NAME);

        const request = store.getAll();

        request.onsuccess = () => {
            resolve(request.result);
        };

        request.onerror = () => {
            reject(request.error);
        };
    });
}
```

### 3. `app/main.js`

```javascript
import { addCard, getAllCards } from "./db.js";

const form = document.getElementById("cardForm");

const todoColumn = document.getElementById("todo");
const progressColumn = document.getElementById("progress");
const doneColumn = document.getElementById("done");

function getStatusText(status) {
    if (status === "todo") {
        return "🔴 Kutilmoqda";
    }

    if (status === "progress") {
        return "🟡 Jarayonda";
    }

    return "🟢 Bajarildi";
}

function createCard(card) {
    const article = document.createElement("article");

    article.className = `card ${card.status}`;

    article.innerHTML = `
        <div class="card-status">
            ${getStatusText(card.status)}
        </div>

        <h3>${escapeHTML(card.title)}</h3>

        <p>${escapeHTML(card.description)}</p>
    `;

    return article;
}

function escapeHTML(text) {
    const div = document.createElement("div");
    div.textContent = text;
    return div.innerHTML;
}

function renderCards(cards) {
    todoColumn.innerHTML = "";
    progressColumn.innerHTML = "";
    doneColumn.innerHTML = "";

    cards.forEach((card) => {
        const element = createCard(card);

        if (card.status === "todo") {
            todoColumn.appendChild(element);
        }

        if (card.status === "progress") {
            progressColumn.appendChild(element);
        }

        if (card.status === "done") {
            doneColumn.appendChild(element);
        }
    });
}

async function loadCards() {
    try {
        const cards = await getAllCards();
        renderCards(cards);
    } catch (error) {
        console.error("Kartalarni yuklashda xato:", error);
    }
}

form.addEventListener("submit", async (event) => {
    event.preventDefault();

    const title = document.getElementById("title").value.trim();
    const description = document
        .getElementById("description")
        .value.trim();

    const status = document.getElementById("status").value;

    if (!title || !description) {
        return;
    }

    const card = {
        title,
        description,
        status,
        createdAt: new Date().toISOString()
    };

    try {
        await addCard(card);

        form.reset();

        await loadCards();
    } catch (error) {
        console.error("Karta saqlashda xato:", error);
    }
});

loadCards();
```

### 4. `style.css`

```css
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: #111827;
    color: #ffffff;
}

.header {
    text-align: center;
    padding: 30px 20px;
    background: #1f2937;
}

.header h1 {
    margin: 0 0 8px;
}

.header p {
    margin: 0;
    color: #e5e7eb;
}

.container {
    width: min(1200px, 94%);
    margin: 30px auto;
}

.add-card {
    background: #1f2937;
    padding: 25px;
    border-radius: 12px;
    margin-bottom: 30px;
}

.add-card h2 {
    margin-top: 0;
}

form {
    display: grid;
    gap: 10px;
}

label {
    font-weight: bold;
}

input,
textarea,
select {
    width: 100%;
    padding: 12px;
    border: 2px solid #9ca3af;
    border-radius: 8px;
    background: #ffffff;
    color: #111827;
    font-size: 16px;
}

textarea {
    min-height: 100px;
    resize: vertical;
}

button {
    padding: 13px;
    margin-top: 5px;
    border: none;
    border-radius: 8px;
    background: #ffffff;
    color: #111827;
    font-weight: bold;
    font-size: 16px;
    cursor: pointer;
}

button:hover {
    background: #e5e7eb;
}

button:focus,
input:focus,
textarea:focus,
select:focus {
    outline: 3px solid #facc15;
    outline-offset: 2px;
}

.board {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 20px;
}

.column {
    background: #1f2937;
    padding: 20px;
    border-radius: 12px;
    min-height: 350px;
}

.column h2 {
    margin-top: 0;
    font-size: 20px;
}

.cards {
    display: grid;
    gap: 15px;
}

.card {
    padding: 18px;
    border-radius: 10px;
    background: #ffffff;
    color: #111827;
    border-left: 7px solid;
}

.card h3 {
    margin: 10px 0;
}

.card p {
    line-height: 1.5;
    margin-bottom: 0;
}

.card-status {
    font-weight: bold;
}

.card.todo {
    border-left-color: #b91c1c;
}

.card.progress {
    border-left-color: #a16207;
}

.card.done {
    border-left-color: #166534;
}

@media (max-width: 800px) {
    .board {
        grid-template-columns: 1fr;
    }
}
```

### 5. `README.md`

````markdown
# AccessBoard

AccessBoard — accessibility talablariga mos Kanban board loyihasi.

## Texnologiyalar

- HTML
- CSS
- JavaScript
- IndexedDB
- WCAG

## Funksiyalar

- Yangi karta qo'shish
- Kartaning holatini tanlash
- Kartalarni IndexedDB'da saqlash
- Sahifa qayta yuklanganda kartalarni IndexedDB'dan tiklash
- Offline ishlash
- Accessibility talablariga mos interfeys

## IndexedDB

Loyihada `app/db.js` faylida IndexedDB ishlatilgan.

Database:

`AccessBoardDB`

Object Store:

`cards`

Quyidagi IndexedDB funksiyalari ishlatiladi:

- `createObjectStore`
- `put`
- `getAll`

## Accessibility

Holatlar faqat rang orqali ko'rsatilmaydi.

Har bir holat rang bilan birga matn va belgi orqali ko'rsatiladi:

- 🔴 Kutilmoqda
- 🟡 Jarayonda
- 🟢 Bajarildi

Masalan, foydalanuvchi ranglarni ajrata olmasa ham,
holat nomini matn orqali tushunishi mumkin.

## WCAG

Matn va fon kombinatsiyalari yuqori kontrast bilan tanlangan.

Asosiy interfeysda:

- oq matn + to'q fon
- qora matn + oq fon

ishlatilgan.

Focus holatlari ham ko'rinadigan qilib sozlangan.

## Loyiha strukturasi

```text
AccessBoard/
│
├── index.html
├── style.css
├── README.md
│
└── app/
    ├── db.js
    └── main.js
````

## Ishga tushirish

Loyihani VS Code orqali oching.

Live Server yordamida `index.html` faylini ishga tushiring.

Karta qo'shing.

Keyin sahifani refresh qiling.

Karta IndexedDB orqali saqlanganligi sababli qayta ko'rinadi.

```

**Muhim:** GitHub'ga aynan shu fayllarni joyla. `app` papkasini ham yaratib, ichiga `db.js` va `main.js`ni qo‘y.

Shunda o‘qituvchi faqat README'ni emas, **haqiqiy JavaScript + IndexedDB + HTML + CSS kodlarini** ham ko‘ra oladi.
```
