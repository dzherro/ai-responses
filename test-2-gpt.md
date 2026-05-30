# Категорія 2, тест розробки фронтенд-компонента Angular: ChatGPT GPT-5.5 (запуск 2)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | ChatGPT |
| Модель | GPT-5.5 |
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

Нижче приклад для Angular 21 зі standalone-компонентом, HttpClient, пошуком, станами loading/error і коректним очищенням підписок через takeUntilDestroyed().

user.model.ts
```typescript
export interface User {
  id: number;
  name: string;
  email: string;
  phone?: string;
}
```

user.service.ts
```typescript
import { HttpClient } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { User } from './user.model';
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
import { User } from './user.model';
import { UserService } from './user.service';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './user-list.component.html',
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef);
  readonly searchControl = new FormControl('', { nonNullable: true });
  users: User[] = [];
  filteredUsers: User[] = [];
  isLoading = false;
  errorMessage = '';
  ngOnInit(): void {
    this.isLoading = true;
    const users$ = this.userService.getUsers().pipe(
      catchError(() => {
        this.errorMessage = 'Не вдалося завантажити список користувачів.';
        return of<User[]>([]);
      }),
      finalize(() => {
        this.isLoading = false;
      })
    );
    const search$ = this.searchControl.valueChanges.pipe(
      startWith(this.searchControl.value)
    );
    combineLatest([users$, search$])
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(([users, searchTerm]) => {
        this.users = users;
        const normalizedSearch = searchTerm.trim().toLowerCase();
        this.filteredUsers = normalizedSearch
          ? users.filter((user) =>
              user.name.toLowerCase().includes(normalizedSearch)
            )
          : users;
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
    placeholder="Введіть імʼя користувача"
  />
  <p *ngIf="isLoading">Завантаження користувачів...</p>
  <p *ngIf="errorMessage && !isLoading">
    {{ errorMessage }}
  </p>
  <p *ngIf="!isLoading && !errorMessage && filteredUsers.length === 0">
    Користувачів не знайдено.
  </p>
  <ul *ngIf="!isLoading && filteredUsers.length > 0">
    <li *ngFor="let user of filteredUsers">
      <strong>{{ user.name }}</strong>
      <br />
      <span>{{ user.email }}</span>
      <br />
      <small *ngIf="user.phone">{{ user.phone }}</small>
    </li>
  </ul>
</section>
```

Щоб HTTP працював, у конфігурації застосунку має бути підключений provideHttpClient():

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
export const appConfig: ApplicationConfig = {
  providers: [provideHttpClient()],
};
```

takeUntilDestroyed(this.destroyRef) автоматично завершує підписку під час знищення компонента, тому витоків памʼяті не буде. Пошук працює реактивно через FormControl.valueChanges, а combineLatest поєднує список користувачів із поточним текстом пошуку.

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

Технічно коректний код з тим самим рівнем сучасних практик, що й у запуску 1. Архітектурний варіант з FormControl з ReactiveFormsModule і комбінацією через combineLatest з операторами catchError і finalize — інший підхід порівняно з запуском 1, але стек залишається на рівні Angular 14-16.
