# Категорія 2, тест розробки фронтенд-компонента Angular: ChatGPT GPT-5.5 (запуск 1)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | ChatGPT |
| Модель | GPT-5.5 |
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

Нижче приклад для Angular 21 зі standalone-компонентом, HttpClient, пошуком, станами loading/error і безпечною підпискою через takeUntilDestroyed.

user.model.ts
```typescript
export interface User {
  id: number;
  name: string;
  email: string;
}
```

user.service.ts
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';
@Injectable({
  providedIn: 'root',
})
export class UserService {
  private readonly apiUrl = 'https://jsonplaceholder.typicode.com/users';
  constructor(private readonly http: HttpClient) {}
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
}
```

Якщо не хочеш використовувати реальний API, можна замінити getUsers() на мок:

```typescript
import { of, delay } from 'rxjs';
```

getUsers(): Observable<User[]> {
```typescript
  return of([
    { id: 1, name: 'Іван Петренко', email: 'ivan@example.com' },
    { id: 2, name: 'Олена Коваль', email: 'olena@example.com' },
    { id: 3, name: 'Андрій Шевченко', email: 'andrii@example.com' },
  ]).pipe(delay(500));
}
```

user-list.component.ts
```typescript
import { Component, DestroyRef, OnInit, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { UserService } from './user.service';
import { User } from './user.model';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './user-list.component.html',
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);
  users: User[] = [];
  filteredUsers: User[] = [];
  searchTerm = '';
  isLoading = false;
  errorMessage = '';
  ngOnInit(): void {
    this.loadUsers();
  }
  loadUsers(): void {
    this.isLoading = true;
    this.errorMessage = '';
    this.userService
      .getUsers()
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (users) => {
          this.users = users;
          this.filteredUsers = users;
          this.isLoading = false;
        },
        error: () => {
          this.errorMessage = 'Не вдалося завантажити список користувачів.';
          this.isLoading = false;
        },
      });
  }
  onSearchChange(): void {
    const normalizedSearch = this.searchTerm.trim().toLowerCase();
    this.filteredUsers = this.users.filter((user) =>
      user.name.toLowerCase().includes(normalizedSearch),
    );
  }
  trackByUserId(index: number, user: User): number {
    return user.id;
  }
}
```

user-list.component.html
```html
<section>
  <h2>Список користувачів</h2>
  <label for="user-search">Пошук за іменем</label>
  <input
    id="user-search"
    type="text"
    name="search"
    placeholder="Введіть імʼя користувача"
    [(ngModel)]="searchTerm"
    (ngModelChange)="onSearchChange()"
  />
  <p *ngIf="isLoading">Завантаження користувачів...</p>
  <p *ngIf="errorMessage && !isLoading">
    {{ errorMessage }}
  </p>
  <ul *ngIf="!isLoading && !errorMessage">
    <li *ngFor="let user of filteredUsers; trackBy: trackByUserId">
      <strong>{{ user.name }}</strong>
      <span>{{ user.email }}</span>
    </li>
  </ul>
  <p *ngIf="!isLoading && !errorMessage && filteredUsers.length === 0">
    Користувачів не знайдено.
  </p>
</section>
```

Підключення HttpClient
У standalone Angular-застосунку потрібно додати provideHttpClient() у app.config.ts:

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
export const appConfig: ApplicationConfig = {
  providers: [provideHttpClient()],
};
```

Основна ідея: UserService відповідає тільки за отримання даних, компонент керує станами інтерфейсу, а takeUntilDestroyed(this.destroyRef) автоматично завершує підписку при знищенні компонента, тому витоків памʼяті не буде.

## Оцінка експерта за рубрикою

| Критерій | Бал |
|----------|-----|
| К1 — точність ідентифікації | 5 |
| К2 — фактична коректність | 5 |
| К3 — повнота відповіді | 3 |
| К4 — відтворюваність (виставляється на серію з 3 запусків) | 4 |
| К5 — педагогічна цінність | 3 |
| **Q — підсумкова оцінка** | **4,30** |

Q обчислюється за формулою: Q = 0,30·К1 + 0,25·К2 + 0,15·К3 + 0,20·К4 + 0,10·К5

## Коментар експерта

Технічно коректний, скомпільовуваний код, але з помітно меншим використанням сучасних практик Angular 21. Класичний синтаксис шаблонів *ngIf і *ngFor замість нового control flow @if/@for. Не використано сигнали. У полі пошуку не задіяно debounceTime і distinctUntilChanged. Архітектурно цей запуск використовує двостороннє звʼязування [(ngModel)] з ручною фільтрацією. Сучасним є лише оператор takeUntilDestroyed(destroyRef) для уникнення витоків пам'яті.
