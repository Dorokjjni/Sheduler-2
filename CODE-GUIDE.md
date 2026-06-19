'use strict';

// Названия шести страниц. Они используются в верхней панели и при навигации.
const PAGE_TITLES = {
  home: 'Главная',
  calendar: 'Календарь',
  regular: 'Регулярное',
  planning: 'Планирование',
  tasks: 'Задачи и цели',
  settings: 'Настройки'
};

// Ключи, под которыми приложение хранит настройки и данные в localStorage.
const STORAGE = {
  lastPage: 'sixPagePwa.lastPage',
  tasks: 'sixPagePwa.tasks',
  compactMenu: 'sixPagePwa.compactMenu',
  showDate: 'sixPagePwa.showDate',
  todayPriority: 'sixPagePwa.todayPriority'
};

// Получаем ссылки на элементы, с которыми будет работать JavaScript.
const body = document.body;
const sideMenu = document.querySelector('#sideMenu');
const menuOverlay = document.querySelector('#menuOverlay');
const openMenuButton = document.querySelector('#openMenuButton');
const closeMenuButton = document.querySelector('#closeMenuButton');
const pageTitle = document.querySelector('#pageTitle');
const currentDate = document.querySelector('#currentDate');
const navItems = [...document.querySelectorAll('[data-page]')];
const pagePanels = [...document.querySelectorAll('[data-page-panel]')];
const quickLinks = [...document.querySelectorAll('[data-go-to]')];
const calendarDays = document.querySelector('#calendarDays');
const taskForm = document.querySelector('#taskForm');
const taskInput = document.querySelector('#taskInput');
const taskList = document.querySelector('#taskList');
const taskEmpty = document.querySelector('#taskEmpty');
const compactToggle = document.querySelector('#compactToggle');
const dateToggle = document.querySelector('#dateToggle');
const resetButton = document.querySelector('#resetButton');
const editableNote = document.querySelector('[data-storage-key="todayPriority"]');

let tasks = loadJson(STORAGE.tasks, []);
let activePage = 'home';

// Безопасно читаем JSON из localStorage. При ошибке возвращаем запасное значение.
function loadJson(key, fallback) {
  try {
    const saved = localStorage.getItem(key);
    return saved ? JSON.parse(saved) : fallback;
  } catch (error) {
    console.warn(`Не удалось прочитать ${key}`, error);
    return fallback;
  }
}

// Открываем боковое меню и обновляем атрибуты доступности.
function openMenu() {
  body.classList.add('menu-open');
  sideMenu.setAttribute('aria-hidden', 'false');
  openMenuButton.setAttribute('aria-expanded', 'true');

  // Переводим фокус на активный пункт, чтобы меню было удобно с клавиатуры.
  requestAnimationFrame(() => {
    document.querySelector('.nav-item.is-active')?.focus();
  });
}

// Закрываем меню. focusBack=true возвращает фокус на кнопку с тремя полосками.
function closeMenu(focusBack = false) {
  body.classList.remove('menu-open');
  sideMenu.setAttribute('aria-hidden', 'true');
  openMenuButton.setAttribute('aria-expanded', 'false');
  if (focusBack) openMenuButton.focus();
}

// Показываем одну из шести страниц и скрываем остальные.
function showPage(pageName, { updateHash = true } = {}) {
  if (!PAGE_TITLES[pageName]) pageName = 'home';
  activePage = pageName;

  pagePanels.forEach((panel) => {
    const isActive = panel.dataset.pagePanel === pageName;
    panel.hidden = !isActive;
    panel.classList.toggle('is-active', isActive);
  });

  navItems.forEach((item) => {
    const isActive = item.dataset.page === pageName;
    item.classList.toggle('is-active', isActive);
    item.setAttribute('aria-current', isActive ? 'page' : 'false');
  });

  pageTitle.textContent = PAGE_TITLES[pageName];
  document.title = `${PAGE_TITLES[pageName]} — Мой планировщик`;
  localStorage.setItem(STORAGE.lastPage, pageName);

  // Хеш позволяет использовать кнопки «Назад/Вперёд» и прямые ссылки на страницы.
  if (updateHash && location.hash !== `#${pageName}`) {
    history.pushState({ page: pageName }, '', `#${pageName}`);
  }

  closeMenu();
  window.scrollTo({ top: 0, behavior: 'smooth' });
}

// Формируем демонстрационные числа календаря. Это отдельный экран, а не картинка.
function renderCalendarPreview() {
  if (!calendarDays) return;
  const numbers = Array.from({ length: 35 }, (_, index) => index + 1);
  calendarDays.innerHTML = numbers.map((number) => {
    const shown = number <= 30 ? number : number - 30;
    const className = number === 19 ? 'today' : '';
    return `<span class="${className}">${shown}</span>`;
  }).join('');
}

// Обновляем дату в правой части шапки.
function renderDate() {
  currentDate.textContent = new Intl.DateTimeFormat('ru-RU', {
    day: 'numeric',
    month: 'short'
  }).format(new Date());
}

// Отрисовываем задачи пятой страницы.
function renderTasks() {
  taskList.innerHTML = '';
  taskEmpty.hidden = tasks.length > 0;

  tasks.forEach((task) => {
    const item = document.createElement('li');
    item.classList.toggle('done', task.done);

    const checkbox = document.createElement('input');
    checkbox.type = 'checkbox';
    checkbox.checked = task.done;
    checkbox.setAttribute('aria-label', `Выполнить задачу: ${task.text}`);
    checkbox.addEventListener('change', () => {
      task.done = checkbox.checked;
      saveTasks();
    });

    const text = document.createElement('span');
    text.textContent = task.text;

    const removeButton = document.createElement('button');
    removeButton.type = 'button';
    removeButton.textContent = '×';
    removeButton.setAttribute('aria-label', `Удалить задачу: ${task.text}`);
    removeButton.addEventListener('click', () => {
      tasks = tasks.filter((itemTask) => itemTask.id !== task.id);
      saveTasks();
    });

    item.append(checkbox, text, removeButton);
    taskList.append(item);
  });
}

// Сохраняем задачи и сразу обновляем интерфейс.
function saveTasks() {
  localStorage.setItem(STORAGE.tasks, JSON.stringify(tasks));
  renderTasks();
}

// Применяем настройки шестой страницы.
function applySettings() {
  const compact = localStorage.getItem(STORAGE.compactMenu) === 'true';
  const showDate = localStorage.getItem(STORAGE.showDate) !== 'false';
  compactToggle.checked = compact;
  dateToggle.checked = showDate;
  body.classList.toggle('compact-menu', compact);
  currentDate.hidden = !showDate;
}

// События открытия и закрытия меню.
openMenuButton.addEventListener('click', openMenu);
closeMenuButton.addEventListener('click', () => closeMenu(true));
menuOverlay.addEventListener('click', () => closeMenu(true));

// Закрываем меню клавишей Escape.
document.addEventListener('keydown', (event) => {
  if (event.key === 'Escape' && body.classList.contains('menu-open')) closeMenu(true);
});

// Каждый пункт меню открывает связанную страницу.
navItems.forEach((item) => {
  item.addEventListener('click', () => showPage(item.dataset.page));
});

// Кнопки быстрого доступа на главной используют ту же функцию навигации.
quickLinks.forEach((button) => {
  button.addEventListener('click', () => showPage(button.dataset.goTo));
});

// Добавление новой задачи на пятой странице.
taskForm.addEventListener('submit', (event) => {
  event.preventDefault();
  const text = taskInput.value.trim();
  if (!text) return;

  tasks.unshift({
    id: crypto.randomUUID ? crypto.randomUUID() : String(Date.now()),
    text,
    done: false
  });
  taskInput.value = '';
  saveTasks();
});

// Сохраняем редактируемую заметку на странице планирования.
const savedPriority = localStorage.getItem(STORAGE.todayPriority);
if (savedPriority) editableNote.textContent = savedPriority;
editableNote.addEventListener('input', () => {
  localStorage.setItem(STORAGE.todayPriority, editableNote.textContent.trim());
});

// Переключатели настроек реагируют сразу после нажатия.
compactToggle.addEventListener('change', () => {
  localStorage.setItem(STORAGE.compactMenu, String(compactToggle.checked));
  applySettings();
});
dateToggle.addEventListener('change', () => {
  localStorage.setItem(STORAGE.showDate, String(dateToggle.checked));
  applySettings();
});

// Удаляем только данные этого приложения, не затрагивая другие сайты.
resetButton.addEventListener('click', () => {
  const confirmed = window.confirm('Удалить задачи, заметки и настройки приложения?');
  if (!confirmed) return;
  Object.values(STORAGE).forEach((key) => localStorage.removeItem(key));
  tasks = [];
  editableNote.textContent = 'Нажмите и напишите приоритет…';
  applySettings();
  renderTasks();
  showPage('home');
});

// При изменении хеша или нажатии «Назад» показываем нужную страницу.
window.addEventListener('hashchange', () => {
  const pageFromHash = location.hash.slice(1);
  showPage(pageFromHash, { updateHash: false });
});
window.addEventListener('popstate', () => {
  const pageFromHash = location.hash.slice(1);
  showPage(pageFromHash, { updateHash: false });
});

// Регистрируем Service Worker после полной загрузки страницы.
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('./service-worker.js').catch((error) => {
      console.warn('Service Worker не зарегистрирован:', error);
    });
  });
}

// Первоначальный запуск интерфейса.
renderDate();
renderCalendarPreview();
renderTasks();
applySettings();
const initialPage = location.hash.slice(1) || localStorage.getItem(STORAGE.lastPage) || 'home';
showPage(initialPage, { updateHash: false });
