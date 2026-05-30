# Категорія 2, тест розробки фронтенд-компонента Angular: Claude Opus 4.7 (запуск 1)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | Claude |
| Модель | Opus 4.7 |
| Дата | травень 2026 |
| Інтерфейс | Cursor Chat |
| Категорія | Генерація програмного коду |
| Тест | Розробка фронтенд-компонента з HTTP-запитом і реактивним фільтруванням |
| Запуск у серії | 1 з 3 |
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
Я перевірив структуру вашого проєкту — це чистий Angular 21 із standalone-компонентами та SSR. Нижче наведу повноцінне рішення з використанням сучасних можливостей Angular 21: standalone-компонент, функція inject(), сигнали (signal, computed), toSignal() для інтеграції з RxJS, новий control flow (@if, @for) та takeUntilDestroyed() для автоматичного відписування.

Я в режимі Ask, тому файли не правлю — нижче готовий код, який ви можете додати в src/app/.


### 1. Інтерфейс користувача — user.model.ts

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

Використовуємо публічний API JSONPlaceholder, який повертає саме таку структуру.


### 2. Сервіс — user.service.ts

Створіть файл src/app/services/user.service.ts:

```typescript
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable, delay, retry, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { User } from '../models/user.model';
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly apiUrl = 'https://jsonplaceholder.typicode.com/users';
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      delay(500),
      retry({ count: 2, delay: 1000 }),
      catchError((err) =>
        throwError(() => new Error(
          err.status === 0
            ? 'Немає з’єднання з сервером'
            : `Не вдалося завантажити користувачів (код ${err.status})`
        ))
      )
    );
  }
}
```

Пояснення:

inject(HttpClient) — сучасний спосіб DI (без конструктора).
providedIn: 'root' — tree-shakable singleton.
delay(500) — штучна затримка, щоб побачити індикатор завантаження.
retry({ count: 2, delay: 1000 }) — автоматичні повторні спроби у разі мережевої помилки.
catchError нормалізує помилку у зрозумілий для користувача текст.

### 3. Налаштування HTTP — app.config.ts

Додайте provideHttpClient у src/app/app.config.ts:

```typescript
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideHttpClient, withFetch } from '@angular/common/http';
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

withFetch() — рекомендована опція для SSR-сумісності (а у вас саме SSR-проєкт).


### 4. Компонент — user-list.ts

Створіть файл src/app/components/user-list/user-list.ts:

```typescript
import { ChangeDetectionStrategy, Component, computed, inject, signal } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { toSignal } from '@angular/core/rxjs-interop';
import { catchError, finalize, of, tap } from 'rxjs';
import { User } from '../../models/user.model';
import { UserService } from '../../services/user.service';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [FormsModule],
  templateUrl: './user-list.html',
  styleUrl: './user-list.scss',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent {
  private readonly userService = inject(UserService);
  protected readonly searchTerm = signal('');
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);
  protected readonly users = toSignal(
    this.userService.getUsers().pipe(
      tap(() => this.error.set(null)),
      catchError((err: Error) => {
        this.error.set(err.message);
        return of([] as User[]);
      }),
      finalize(() => this.loading.set(false))
    ),
    { initialValue: [] as User[] }
  );
  protected readonly filteredUsers = computed<User[]>(() => {
    const term = this.searchTerm().trim().toLowerCase();
    const list = this.users();
    if (!term) return list;
    return list.filter(
      (u) =>
        u.name.toLowerCase().includes(term) ||
        u.username.toLowerCase().includes(term)
    );
  });
  protected onSearchChange(value: string): void {
    this.searchTerm.set(value);
  }
  protected trackById(_: number, user: User): number {
    return user.id;
  }
}
```

Чому саме так — ключові моменти:

Прийом	Що дає
toSignal(...)
Перетворює Observable на сигнал і сам викликає unsubscribe() при знищенні компонента — це і є коректне керування життєвим циклом без ngOnDestroy та Subject.
signal для searchTerm, loading, error
Реактивний стан без BehaviorSubject.
computed для filteredUsers
Перерахунок лише коли змінюються users() або searchTerm() — фільтрація в шаблоні не виконується на кожен tick.
ChangeDetectionStrategy.OnPush
Працює ідеально разом із сигналами — мінімум зайвих рендерів.
catchError + finalize
Помилка не зриває потік, а loading гарантовано стає false.
Альтернатива з takeUntilDestroyed() (якщо потрібен прямий .subscribe()):

```typescript
this.userService.getUsers()
  .pipe(takeUntilDestroyed())
  .subscribe(users => this.users.set(users));
```

Імпорт: import { takeUntilDestroyed } from '@angular/core/rxjs-interop';. Викликати його потрібно у полі класу або в конструкторі — там, де є injection context.


### 5. Шаблон — user-list.html

Створіть файл src/app/components/user-list/user-list.html:

```html
<section class="user-list">
  <header class="user-list__header">
    <h1>Користувачі</h1>
    <input
      type="search"
      class="user-list__search"
      placeholder="Пошук за іменем або логіном..."
      [ngModel]="searchTerm()"
      (ngModelChange)="onSearchChange($event)"
      [disabled]="loading() || !!error()"
      aria-label="Пошук користувачів"
    />
  </header>
  @if (loading()) {
    <p class="user-list__status user-list__status--loading">
      Завантаження користувачів...
    </p>
  } @else if (error()) {
    <p class="user-list__status user-list__status--error" role="alert">
      {{ error() }}
    </p>
  } @else if (filteredUsers().length === 0) {
    <p class="user-list__status">
      Користувачів за запитом «{{ searchTerm() }}» не знайдено.
    </p>
  } @else {
    <ul class="user-list__items">
      @for (user of filteredUsers(); track trackById($index, user)) {
        <li class="user-card">
          <h3 class="user-card__name">{{ user.name }}</h3>
          <p class="user-card__meta">&#64;{{ user.username }}</p>
          <p class="user-card__contact">
            <a [href]="'mailto:' + user.email">{{ user.email }}</a>
          </p>
          <p class="user-card__contact">{{ user.phone }}</p>
          <p class="user-card__contact">
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

Що тут важливо:

@if / @else if / @else — новий control flow Angular 17+, без *ngIf.
```typescript
@for ... track ... — обов'язковий track (тут через trackById) для ефективного оновлення DOM.
```

[ngModel] + (ngModelChange) — однонаправлене зв'язування з сигналом (краще, ніж [(ngModel)], бо джерело істини — сигнал).
Усі гілки стану (loading, error, порожній результат, дані) обробляються явно.

### 6. Стилі — user-list.scss

Створіть файл src/app/components/user-list/user-list.scss:

```scss
.user-list {
  max-width: 800px;
  margin: 2rem auto;
  padding: 1rem;
  font-family: system-ui, sans-serif;
  &__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
    margin-bottom: 1.5rem;
  }
  &__search {
    flex: 1;
    max-width: 320px;
    padding: 0.5rem 0.75rem;
    border: 1px solid #ccc;
    border-radius: 6px;
    font-size: 1rem;
    &:focus {
      outline: 2px solid #3f51b5;
      border-color: #3f51b5;
    }
  }
  &__status {
    padding: 1rem;
    text-align: center;
    color: #555;
    &--loading { color: #3f51b5; }
    &--error   { color: #c62828; background: #ffebee; border-radius: 6px; }
  }
  &__items {
    list-style: none;
    padding: 0;
    display: grid;
    gap: 1rem;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  }
}
.user-card {
  padding: 1rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  background: #fff;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.06);
  transition: transform 0.15s, box-shadow 0.15s;
  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
  }
  &__name { margin: 0 0 0.25rem; font-size: 1.1rem; }
  &__meta { margin: 0 0 0.75rem; color: #757575; font-size: 0.9rem; }
  &__contact { margin: 0.25rem 0; font-size: 0.9rem; }
}
```


### 7. Підключення компонента

У src/app/app.ts додайте імпорт та використання:

```typescript
import { Component } from '@angular/core';
import { UserListComponent } from './components/user-list/user-list';
@Component({
  selector: 'app-root',
  standalone: true,
  imports: [UserListComponent],
  template: `<app-user-list />`,
})
export class App {}
```

Як це працює — короткий огляд потоку
При створенні компонента toSignal() одразу підписується на getUsers().
loading = true до моменту finalize (успіх або помилка).
У разі помилки — error отримує текст, users лишається [].
Користувач вводить текст → signal searchTerm оновлюється → computed filteredUsers перераховується → шаблон оновлюється тільки в потрібних місцях завдяки OnPush + сигналам.
При знищенні компонента toSignal автоматично викликає внутрішній unsubscribe() — витоків пам'яті немає.
Перевірка
Після створення всіх файлів запустіть:

npm start
і відкрийте http://localhost:4200. Має з'явитись 10 користувачів з JSONPlaceholder, з робочим пошуком по імені та юзернейму.

Якщо потрібен варіант без зовнішнього API (із замоканими даними через of([...]) всередині сервісу) або з AsyncPipe замість сигналів для порівняння — скажіть, і я надам альтернативну реалізацію.

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

Повністю сучасний стек Angular 21: standalone-компонент, inject(), сигнали (signal, computed, toSignal), новий control flow (@if, @else if, @for з track), ChangeDetectionStrategy.OnPush. Для HTTP-клієнта застосовано provideHttpClient(withFetch()) — оптимально для SSR-конфігурації проекту. Модель самостійно вичитала структуру проекту (app.config.ts, package.json). Реактивний підхід через toSignal() автоматично керує підписками. Присутні a11y-атрибути, стилі, кнопка повторної спроби, обробка порожнього стану через @empty.
