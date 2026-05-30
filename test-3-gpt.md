# Категорія 2, тест розробки фронтенд-компонента Angular: ChatGPT GPT-5.5 (запуск 3)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | ChatGPT |
| Модель | GPT-5.5 |
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

Нижче приклад для Angular 21 зі standalone-компонентом. Він використовує HttpClient, Observable, ReactiveFormsModule, takeUntilDestroyed() для автоматичного очищення підписок, а також стани loading і error.

user.service.ts
```typescript
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable } from 'rxjs';
export interface User {
  id: number;
  name: string;
  email: string;
  username: string;
}
@Injectable({
  providedIn: 'root',
})
export class UserService {
  private readonly http = inject(HttpClient);
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('https://jsonplaceholder.typicode.com/users');
  }
}
```

user-list.component.ts
```typescript
import { CommonModule } from '@angular/common';
import { Component, DestroyRef, inject, OnInit } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { catchError, combineLatest, finalize, of, startWith } from 'rxjs';
import { User, UserService } from './user.service';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './user-list.component.html',
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);
  searchControl = new FormControl('', { nonNullable: true });
  users: User[] = [];
  filteredUsers: User[] = [];
  isLoading = false;
  errorMessage = '';
  ngOnInit(): void {
    this.isLoading = true;
    const users$ = this.userService.getUsers().pipe(
      catchError(() => {
        this.errorMessage = 'Не вдалося завантажити список користувачів.';
        return of([]);
      }),
      finalize(() => {
        this.isLoading = false;
      }),
    );
    const search$ = this.searchControl.valueChanges.pipe(
      startWith(this.searchControl.value),
    );
    combineLatest([users$, search$])
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(([users, searchTerm]) => {
        this.users = users;
        const normalizedSearch = searchTerm.trim().toLowerCase();
        this.filteredUsers = users.filter((user) =>
          user.name.toLowerCase().includes(normalizedSearch),
        );
      });
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
    type="search"
    [formControl]="searchControl"
    placeholder="Введіть ім’я користувача"
  />
  <p *ngIf="isLoading">Завантаження користувачів...</p>
  <p *ngIf="errorMessage && !isLoading">
    {{ errorMessage }}
  </p>
  <p *ngIf="!isLoading && !errorMessage && filteredUsers.length === 0">
    Користувачів не знайдено.
  </p>
  <ul *ngIf="!isLoading && !errorMessage && filteredUsers.length > 0">
    <li *ngFor="let user of filteredUsers">
      <strong>{{ user.name }}</strong>
      <span>({{ user.username }})</span>
      <br />
      <a [href]="'mailto:' + user.email">{{ user.email }}</a>
    </li>
  </ul>
</section>
```

Щоб HttpClient працював у standalone Angular-застосунку, у app.config.ts має бути підключений provideHttpClient():

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
export const appConfig: ApplicationConfig = {
  providers: [provideHttpClient()],
};
```

Ключовий момент для уникнення витоків пам’яті тут — takeUntilDestroyed(this.destroyRef). Він автоматично завершує підписку, коли компонент знищується, тому вручну писати ngOnDestroy() і Subject не потрібно.

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

Технічно коректний код, аналогічний до запуску 2. Серія з трьох запусків ChatGPT демонструє стабільно прийнятну якість на консервативному стеку. Без сигналів, без нового control flow, без debounceTime, але з коректним керуванням життєвим циклом.
