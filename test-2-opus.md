# Категорія 2, тест розробки фронтенд-компонента Angular: Claude Opus 4.7 (запуск 2)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | Claude |
| Модель | Opus 4.7 |
| Дата | травень 2026 |
| Інтерфейс | Cursor Chat |
| Категорія | Генерація програмного коду |
| Тест | Розробка фронтенд-компонента з HTTP-запитом і реактивним фільтруванням |
| Запуск у серії | 2 з 3 |
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

Я перебуваю в режимі Ask, тому не можу безпосередньо створювати або змінювати файли у вашому проєкті, але повний код із поясненнями нижче. Ви побачили чинний package.json — у вас Angular 21.2 із standalone-архітектурою та RxJS 7.8, тож я використовуватиму сучасні API: inject(), сигнали (signal, computed), новий синтаксис шаблонів (@if, @for), а також takeUntilDestroyed для контролю підписок.


### 1. Налаштування HttpClient у app.config.ts

Щоб HttpClient працював, його потрібно зареєструвати у провайдерах застосунку. У вашому файлі src/app/app.config.ts слід додати provideHttpClient():

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

withFetch() рекомендується для SSR (у вас підключений @angular/ssr) — він використовує Fetch API замість застарілого XMLHttpRequest.


### 2. Модель даних — src/app/models/user.model.ts

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

Інтерфейс відповідає структурі відповіді тестового публічного API JSONPlaceholder.


### 3. Сервіс — src/app/services/user.service.ts

```typescript
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable, catchError, throwError } from 'rxjs';
import { User } from '../models/user.model';
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = 'https://jsonplaceholder.typicode.com/users';
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      catchError((error) => {
        const message =
          error?.status === 0
            ? 'Немає з’єднання з сервером. Перевірте інтернет.'
            : `Помилка ${error?.status ?? ''}: не вдалося завантажити користувачів.`;
        return throwError(() => new Error(message));
      }),
    );
  }
}
```

Що тут важливо:

providedIn: 'root' — сервіс є singleton-ом, дерево-shake-friendly.
inject(HttpClient) — сучасний спосіб ін’єкції замість конструкторної (Angular 14+).
catchError перетворює HTTP-помилку на людиночитне повідомлення й «прокидає» його далі через throwError, щоб компонент сам вирішував, що з нею робити.

### 4. Компонент — src/app/components/user-list/user-list.ts

```typescript
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  computed,
  inject,
  signal,
} from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { FormsModule } from '@angular/forms';
import { User } from '../../models/user.model';
import { UserService } from '../../services/user.service';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [FormsModule],
  templateUrl: './user-list.html',
  styleUrl: './user-list.css',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserList {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);
  protected readonly users = signal<User[]>([]);
  protected readonly loading = signal<boolean>(false);
  protected readonly error = signal<string | null>(null);
  protected readonly searchTerm = signal<string>('');
  protected readonly filteredUsers = computed<User[]>(() => {
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
  protected loadUsers(): void {
    this.loading.set(true);
    this.error.set(null);
    this.userService
      .getUsers()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (data) => {
          this.users.set(data);
          this.loading.set(false);
        },
        error: (err: Error) => {
          this.error.set(err.message);
          this.loading.set(false);
        },
      });
  }
  protected onSearchChange(value: string): void {
    this.searchTerm.set(value);
  }
}
```

Ключові рішення:

Сигнали замість властивостей класу. users, loading, error, searchTerm — це signal(...). Шаблон автоматично переотрисовується лише там, де ці сигнали читаються. Це відмінно поєднується з ChangeDetectionStrategy.OnPush і дає найшвидший рендеринг.

computed для фільтрації. filteredUsers — це похідний сигнал: він автоматично перераховується тільки тоді, коли змінюється searchTerm або users. Не потрібно ні BehaviorSubject, ні combineLatest.

Керування життєвим циклом. takeUntilDestroyed(this.destroyRef) — стандартний підхід Angular 16+ для гарантії, що підписка завершиться разом із компонентом. Це замінює стару конструкцію з Subject + ngOnDestroy() і повністю запобігає витокам пам’яті.

Стан завантаження та помилки. Сигнали loading й error керують відображенням індикатора та повідомлення про помилку. Метод loadUsers() можна викликати повторно з кнопки «Спробувати ще раз».

OnPush. Зі сигналами Angular точно знає, які поля впливають на DOM, тому стратегія OnPush тут безпечна та продуктивна.


### 5. Шаблон — src/app/components/user-list/user-list.html

```html
<section class="user-list">
  <header class="user-list__header">
    <h2>Список користувачів</h2>
    <label class="search">
      <span class="search__label">Пошук за іменем:</span>
      <input
        type="search"
        class="search__input"
        placeholder="Введіть ім'я..."
        [ngModel]="searchTerm()"
        (ngModelChange)="onSearchChange($event)"
        [disabled]="loading() || !!error()"
        aria-label="Пошук користувача за іменем"
      />
    </label>
  </header>
  @if (loading()) {
    <p class="state state--loading" role="status">Завантаження...</p>
  } @else if (error(); as errMsg) {
    <div class="state state--error" role="alert">
      <p>{{ errMsg }}</p>
      <button type="button" (click)="loadUsers()">Спробувати ще раз</button>
    </div>
  } @else {
    @if (filteredUsers().length === 0) {
      <p class="state state--empty">
        Користувачів за запитом «{{ searchTerm() }}» не знайдено.
      </p>
    } @else {
      <ul class="users">
        @for (user of filteredUsers(); track user.id) {
          <li class="users__item">
            <h3 class="users__name">{{ user.name }}</h3>
            <p class="users__meta">
              <span>&#64;{{ user.username }}</span> &middot;
              <a [href]="'mailto:' + user.email">{{ user.email }}</a>
            </p>
            <p class="users__meta">
              {{ user.phone }} &middot;
              <a [href]="'https://' + user.website" target="_blank" rel="noopener">
                {{ user.website }}
              </a>
            </p>
          </li>
        }
      </ul>
    }
  }
</section>
```

Що тут важливо:

[ngModel]="searchTerm()" (ngModelChange)="onSearchChange($event)" — двостороння взаємодія з сигналом. Можна було б використати [(ngModel)] із простим property, але із сигналом потрібна явна форма «одностороннє читання + обробник». В Angular 21 також доступні signal-forms (Forms.signal()), але FormsModule лишається найпростішим варіантом для одного поля.
Сучасний control flow @if / @else if / @else / @for — це новий синтаксис, рекомендований замість *ngIf / *ngFor.
track user.id — обов’язковий у @for для оптимального rerender.
Атрибути role="status" / role="alert" поліпшують доступність (a11y).

### 6. Стилі — src/app/components/user-list/user-list.css

```scss
.user-list {
  max-width: 720px;
  margin: 2rem auto;
  font-family: system-ui, sans-serif;
}
.user-list__header {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.25rem;
}
.search {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}
.search__input {
  padding: 0.5rem 0.75rem;
  font-size: 1rem;
  border: 1px solid #c9c9c9;
  border-radius: 8px;
  outline: none;
  transition: border-color 0.15s ease;
}
.search__input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.25);
}
.state {
  padding: 1rem;
  border-radius: 8px;
  text-align: center;
}
.state--loading {
  background: #eef2ff;
  color: #3730a3;
}
.state--error {
  background: #fee2e2;
  color: #991b1b;
}
.state--empty {
  background: #f3f4f6;
  color: #4b5563;
}
.users {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  gap: 0.75rem;
}
.users__item {
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  background: #fff;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
}
.users__name {
  margin: 0 0 0.25rem;
  font-size: 1.05rem;
}
.users__meta {
  margin: 0.15rem 0;
  font-size: 0.9rem;
  color: #4b5563;
}
```


### 7. Підключення компонента

Якщо ви відображаєте його через маршрут, у src/app/app.routes.ts:

```typescript
import { Routes } from '@angular/router';
export const routes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./components/user-list/user-list').then((m) => m.UserList),
  },
```

];
Або просто додайте <app-user-list /> у шаблон app.html та імпортуйте UserList у imports кореневого App-компонента.

Підсумок виконання вимог
Вимога	Як вирішено
UserService повертає Observable<User[]>
getUsers() повертає http.get<User[]>(...) (JSONPlaceholder)
Підписка в компоненті й відображення списку
subscribe({ next, error }) з оновленням сигналу users, який рендериться через @for
Поле пошуку з фільтрацією у міру введення
[ngModel] + (ngModelChange) оновлюють сигнал searchTerm; computed filteredUsers миттєво перераховується
Запобігання витокам пам’яті
takeUntilDestroyed(this.destroyRef) гарантує відписку при знищенні компонента
Стан завантаження та помилки
Сигнали loading/error + блоки @if/@else if/@else у шаблоні; є кнопка «Спробувати ще раз»
Сучасний підхід (Angular 21)
Standalone-компонент, сигнали, inject(), новий control flow, OnPush, provideHttpClient(withFetch())
Якщо захочете, щоб я вніс ці файли у ваш проєкт — перемкніться, будь ласка, в Agent mode, і я створю всі ці файли та зроблю потрібні зміни в app.config.ts і app.routes.ts за один захід.

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

Стабільно високий рівень з повним використанням сучасних практик Angular 21. Архітектурний варіант через takeUntilDestroyed(destroyRef) з ручним subscribe та явним керуванням сигналами (signal для users, loading, error; computed для filteredUsers). Той самий обсяг функціональності, що й у запуску 1, але інше архітектурне рішення — стилістична варіативність без зниження якості.
