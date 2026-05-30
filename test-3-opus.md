# Категорія 2, тест розробки фронтенд-компонента Angular: Claude Opus 4.7 (запуск 3)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | Claude |
| Модель | Opus 4.7 |
| Дата | травень 2026 |
| Інтерфейс | Cursor Chat |
| Категорія | Генерація програмного коду |
| Тест | Розробка фронтенд-компонента з HTTP-запитом і реактивним фільтруванням |
| Запуск у серії | 3 з 3 |
| Версія фреймворку | Angular 21 (SSR-конфігурація) |

## Промпт

> Напиши компонент на Angular (версія 21), який відображає список користувачів, завантажений із сервісу через HTTP. Виконай такі вимоги:
>
> 1. Створи сервіс UserService з методом, що повертає Observable зі списком користувачів (можеш використати тестовий публічний API або замокати дані).
> 2. У компоненті підпишись на цей Observable і відобрази список користувачів у шаблоні.
> 3. Додай поле пошуку, яке фільтрує список за іменем у міру введення тексту.
> 4. Передбач коректне керування життєвим циклом компонента, щоб уникнути витоків пам'яті.
> 5. Додай обробку стану завантаження та помилки.
>
> Відповідай українською мовою, наведи повний код компонента, сервісу та шаблону з поясненнями.

## Відповідь моделі

Компонент списку користувачів на Angular 21
Нижче — повне рішення з використанням сучасних можливостей Angular 21: standalone-компонентів, сигналів (signals), нового control flow (@if/@for), функції inject(), а також takeUntilDestroyed() для коректного керування підписками.

Примітка: я зараз у режимі Ask і не вношу зміни у файли — нижче наведено готовий код, який ви можете скопіювати у проєкт.


### 1. Модель даних user.model.ts

Створіть файл src/app/models/user.model.ts:

```typescript
export interface User {
  id: number;
  name: string;
  username: string;
  email: string;
  phone: string;
  website: string;
}
```


### 2. Сервіс UserService

Файл src/app/services/user.service.ts. Використовуємо публічний API JSONPlaceholder, HttpClient та inject():

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, delay } from 'rxjs/operators';
import { User } from '../models/user.model';
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = 'https://jsonplaceholder.typicode.com/users';
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      // штучна затримка, щоб показати індикатор завантаження
      delay(500),
      catchError((error) => {
        console.error('Помилка завантаження користувачів:', error);
        return throwError(
          () => new Error('Не вдалося завантажити список користувачів. Спробуйте пізніше.')
        );
      })
    );
  }
}
```

Пояснення:

```typescript
@Injectable({ providedIn: 'root' }) — сервіс реєструється у кореневому інжекторі (singleton).
```

inject(HttpClient) — сучасний спосіб впорскування залежностей (замість конструктора).
catchError перетворює помилку HTTP на зрозуміле повідомлення для користувача.
delay(500) лише для демонстрації стану «завантаження» — у проді його не потрібно.

### 3. Реєстрація HttpClient у app.config.ts

У вас уже є app.config.ts. До нього треба додати провайдер provideHttpClient():

```typescript
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { routes } from './app.routes';
export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes),
    provideClientHydration(withEventReplay()),
    provideHttpClient(withFetch()),
  ],
};
```

withFetch() рекомендовано в Angular 21, особливо для SSR (у вас вже налаштовано @angular/ssr).


### 4. Компонент UsersListComponent

Файл src/app/components/users-list/users-list.component.ts:

```typescript
import {
  Component,
  inject,
  signal,
  computed,
  DestroyRef,
  ChangeDetectionStrategy,
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { UserService } from '../../services/user.service';
import { User } from '../../models/user.model';
@Component({
  selector: 'app-users-list',
  standalone: true,
  imports: [FormsModule],
  templateUrl: './users-list.component.html',
  styleUrl: './users-list.component.css',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UsersListComponent {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);
  readonly users = signal<User[]>([]);
  readonly searchTerm = signal<string>('');
  readonly isLoading = signal<boolean>(false);
  readonly errorMessage = signal<string | null>(null);
  readonly filteredUsers = computed<User[]>(() => {
    const term = this.searchTerm().trim().toLowerCase();
    const list = this.users();
    if (!term) {
      return list;
    }
    return list.filter((u) => u.name.toLowerCase().includes(term));
  });
  constructor() {
    this.loadUsers();
  }
  loadUsers(): void {
    this.isLoading.set(true);
    this.errorMessage.set(null);
    this.userService
      .getUsers()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (data) => {
          this.users.set(data);
          this.isLoading.set(false);
        },
        error: (err: Error) => {
          this.errorMessage.set(err.message);
          this.isLoading.set(false);
        },
      });
  }
  onSearchChange(value: string): void {
    this.searchTerm.set(value);
  }
  retry(): void {
    this.loadUsers();
  }
}
```

Ключові моменти:

Елемент	Призначення
signal<T>()
Реактивний стан — Angular автоматично перерендерить шаблон при зміні
computed()
Похідний сигнал — фільтрований список перераховується сам, коли змінюється users або searchTerm
inject(DestroyRef)
Доступ до життєвого циклу компонента без OnDestroy
takeUntilDestroyed(destroyRef)
Автоматично відписується від Observable при знищенні компонента — захист від витоків пам'яті
ChangeDetectionStrategy.OnPush
Оптимізація — Angular перевіряє компонент тільки при зміні сигналів/інпутів

### 5. Шаблон users-list.component.html

```html
<section class="users-container">
  <header class="users-header">
    <h2>Список користувачів</h2>
    <input
      type="text"
      class="search-input"
      placeholder="Пошук за іменем..."
      [ngModel]="searchTerm()"
      (ngModelChange)="onSearchChange($event)"
      [disabled]="isLoading() || !!errorMessage()"
    />
  </header>
  @if (isLoading()) {
    <div class="state state--loading">
      <span class="spinner" aria-hidden="true"></span>
      <p>Завантаження користувачів...</p>
    </div>
  } @else if (errorMessage()) {
    <div class="state state--error" role="alert">
      <p>{{ errorMessage() }}</p>
      <button type="button" class="retry-btn" (click)="retry()">
        Спробувати ще раз
      </button>
    </div>
  } @else if (filteredUsers().length === 0) {
    <div class="state state--empty">
      <p>Користувачів не знайдено.</p>
    </div>
  } @else {
    <ul class="users-list">
      @for (user of filteredUsers(); track user.id) {
        <li class="user-card">
          <h3 class="user-name">{{ user.name }}</h3>
          <p class="user-username">&#64;{{ user.username }}</p>
          <p class="user-info">
            <strong>Email:</strong>
            <a [href]="'mailto:' + user.email">{{ user.email }}</a>
          </p>
          <p class="user-info"><strong>Телефон:</strong> {{ user.phone }}</p>
          <p class="user-info">
            <strong>Сайт:</strong>
            <a [href]="'https://' + user.website" target="_blank" rel="noopener">
              {{ user.website }}
            </a>
          </p>
        </li>
      } @empty {
        <li>Список порожній</li>
      }
    </ul>
  }
</section>
```

Пояснення шаблону:

```typescript
@if / @else if / @else — новий синтаксис control flow Angular 17+ (замість *ngIf).
@for ... track user.id — обов'язково вказувати track для оптимізації відображення (це вимога нового синтаксису).
```

signal() викликається як функція в шаблоні: searchTerm(), isLoading(), filteredUsers().
[ngModel] + (ngModelChange) замість двостороннього [(ngModel)] — даємо сигналу повний контроль над станом.

### 6. Стилі users-list.component.css

```scss
.users-container {
  max-width: 800px;
  margin: 24px auto;
  padding: 16px;
  font-family: system-ui, -apple-system, sans-serif;
}
.users-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}
.search-input {
  flex: 1;
  min-width: 220px;
  padding: 10px 14px;
  font-size: 14px;
  border: 1px solid #ccc;
  border-radius: 8px;
  outline: none;
  transition: border-color 0.2s;
}
.search-input:focus {
  border-color: #3f51b5;
}
.users-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
}
.user-card {
  padding: 16px;
  border: 1px solid #e0e0e0;
  border-radius: 10px;
  background: #fafafa;
  transition: box-shadow 0.2s;
}
.user-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}
.user-name { margin: 0 0 4px; font-size: 18px; }
.user-username { margin: 0 0 12px; color: #666; font-size: 14px; }
.user-info { margin: 4px 0; font-size: 14px; }
.state {
  padding: 24px;
  text-align: center;
  border-radius: 8px;
}
.state--loading { color: #555; }
.state--error { background: #fdecea; color: #b71c1c; }
.state--empty { color: #888; }
.spinner {
  display: inline-block;
  width: 18px;
  height: 18px;
  border: 2px solid #ccc;
  border-top-color: #3f51b5;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  vertical-align: middle;
  margin-right: 8px;
}
```

@keyframes spin {
```scss
  to { transform: rotate(360deg); }
}
.retry-btn {
  margin-top: 12px;
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  background: #b71c1c;
  color: #fff;
  cursor: pointer;
}
.retry-btn:hover { background: #8e0000; }
```


### 7. Підключення у app.ts

Подивіться на ваш кореневий src/app/app.ts (наразі це файл, що відповідає за AppComponent) — додайте UsersListComponent до imports та використайте його в шаблоні app.html:

```typescript
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { UsersListComponent } from './components/users-list/users-list.component';
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, UsersListComponent],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {}
```

У app.html достатньо додати:

```html
<app-users-list />
```

Як це працює — короткий розбір
Завантаження даних. У конструкторі компонента викликається loadUsers(). Сервіс повертає Observable<User[]>, на який підписуємось через .subscribe().
Відписка. Оператор takeUntilDestroyed(this.destroyRef) автоматично завершує підписку при знищенні компонента — витоків пам'яті не буде без жодного OnDestroy чи Subject.
Реактивність через сигнали. Зміна searchTerm → перерахунок computed-сигналу filteredUsers → перерендер тільки потрібної частини DOM.
Стани UI. isLoading, errorMessage та порожній результат обробляються через @if/@else if, тож користувач завжди бачить адекватний фідбек.
OnPush + Signals — оптимальна продуктивність: Angular перевіряє компонент лише коли реально змінюється сигнал.
Альтернатива: повністю реактивний підхід через toSignal
Якщо хочете ще менше коду — можна перетворити Observable одразу на сигнал і взагалі не викликати subscribe() вручну:

```typescript
import { toSignal } from '@angular/core/rxjs-interop';
import { catchError, of, startWith, map } from 'rxjs';
interface State {
  users: User[];
  loading: boolean;
  error: string | null;
}
```

readonly state = toSignal<State>(
```typescript
  this.userService.getUsers().pipe(
    map((users) => ({ users, loading: false, error: null })),
    catchError((err: Error) =>
      of({ users: [], loading: false, error: err.message })
    ),
    startWith({ users: [], loading: true, error: null })
  ),
  { initialValue: { users: [], loading: true, error: null } }
);
```

Тоді в шаблоні: state().loading, state().error, state().users. toSignal() сам обробляє відписку при знищенні компонента.

Підсумок
Це рішення задовольняє всі вимоги:

✅ Сервіс із методом, що повертає Observable<User[]>
✅ Компонент підписується і виводить список
✅ Поле пошуку з реактивною фільтрацією (через computed)
✅ Захист від витоків пам'яті (takeUntilDestroyed)
✅ Стани завантаження, помилки, порожнього результату
✅ Сучасні практики Angular 21: standalone, signals, control flow, inject()
Якщо хочете, щоб я застосував усі ці зміни у проєкті — переключіть мене в Agent mode, і я створю всі файли та оновлю app.config.ts за вас.

## Оцінка експерта за рубрикою

| Критерій | Бал |
|----------|-----|
| К1 — точність ідентифікації | 5 |
| К2 — фактична коректність | 5 |
| К3 — повнота відповіді | 5 |
| К4 — відтворюваність (виставляється на серію з 3 запусків) | 4 |
| К5 — педагогічна цінність | 5 |
| **Q — підсумкова оцінка** | **4,80** |

Q обчислюється за формулою: Q = 0,30·К1 + 0,25·К2 + 0,15·К3 + 0,20·К4 + 0,10·К5

## Коментар експерта

Стабільно високий рівень. Архітектурний варіант з takeUntilDestroyed(destroyRef) і додатковими варіаціями. Серія з трьох запусків Claude демонструє стабільно високу якість з невеликою стилістичною варіативністю архітектурних рішень (toSignal vs takeUntilDestroyed) — обидва підходи валідні і сучасні.
