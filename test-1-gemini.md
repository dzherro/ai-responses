# Категорія 2, тест розробки фронтенд-компонента Angular: Gemini 3.1 Pro (запуск 1)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | Gemini |
| Модель | 3.1 Pro |
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

Ось повний приклад сучасного компонента на Angular 21, який відповідає всім вашим вимогам. У цьому коді використовуються найновіші можливості фреймворку: Standalone компоненти, функція inject(), новий синтаксис шаблонів (@if, @for) та оператор takeUntilDestroyed для безпечного керування підписками.


### 1. Модель даних (user.model.ts)

Спочатку створимо інтерфейс для типізації даних користувача:

```typescript
export interface User {
  id: number;
  name: string;
  email: string;
}
```


### 2. Сервіс (user.service.ts)

Сервіс використовує HttpClient для отримання даних із публічного тестового API (JSONPlaceholder).

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';
@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Використовуємо сучасний підхід з inject() замість конструктора
  private readonly http = inject(HttpClient);
  private readonly apiUrl = 'https://jsonplaceholder.typicode.com/users';
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
}
```


### 3. Компонент (user-list.component.ts)

Тут ми реалізуємо логіку завантаження, фільтрації та обробки станів.

```typescript
import { Component, OnInit, inject, DestroyRef } from '@angular/core';
import { ReactiveFormsModule, FormControl } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';
import { UserService } from './user.service';
import { User } from './user.model';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [ReactiveFormsModule], // Імпортуємо модуль для роботи з FormControl
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.css']
})
export class UserListComponent implements OnInit {
  private readonly userService = inject(UserService);
  private readonly destroyRef = inject(DestroyRef); // Потрібен для takeUntilDestroyed
  // Контрол для поля пошуку
  searchControl = new FormControl('', { nonNullable: true });
```

  
```typescript
  users: User[] = [];
  filteredUsers: User[] = [];
```

  
```typescript
  // Стани компонента
  isLoading = true;
  errorMessage = '';
  ngOnInit(): void {
    this.loadUsers();
    this.setupSearch();
  }
  private loadUsers(): void {
    this.isLoading = true;
    this.errorMessage = '';
    this.userService.getUsers()
      // takeUntilDestroyed автоматично скасовує підписку при знищенні компонента
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (data) => {
          this.users = data;
          this.filteredUsers = data; // Спочатку показуємо всіх
          this.isLoading = false;
        },
        error: (err) => {
          this.errorMessage = 'Не вдалося завантажити дані. Спробуйте пізніше.';
          this.isLoading = false;
          console.error('Помилка завантаження користувачів:', err);
        }
      });
  }
  private setupSearch(): void {
    this.searchControl.valueChanges
      .pipe(
        debounceTime(300), // Затримка 300мс, щоб не фільтрувати на кожне натискання
        distinctUntilChanged(), // Пропускаємо лише якщо текст дійсно змінився
        takeUntilDestroyed(this.destroyRef) // Запобігаємо витоку пам'яті
      )
      .subscribe(searchTerm => {
        const term = searchTerm.toLowerCase().trim();
        this.filteredUsers = this.users.filter(user => 
          user.name.toLowerCase().includes(term)
        );
      });
  }
}
```


### 4. Шаблон (user-list.component.html)

У шаблоні використовується новий Control Flow синтаксис Angular (доступний з 17-ї версії), який є швидшим і читабельнішим за старі директиви *ngIf та *ngFor.

```html
<div class="user-container">
  <h2>Список користувачів</h2>
  <div class="search-box">
    <input 
      type="text" 
      [formControl]="searchControl" 
      placeholder="Пошук за іменем..."
      class="search-input"
    >
  </div>
  <!-- Обробка стану завантаження -->
  @if (isLoading) {
    <div class="loading">
      ⏳ Завантаження даних...
    </div>
  } 
  <!-- Обробка стану помилки -->
  @else if (errorMessage) {
    <div class="error">
      ❌ {{ errorMessage }}
    </div>
  } 
  <!-- Відображення списку -->
  @else {
    <ul class="user-list">
      @for (user of filteredUsers; track user.id) {
        <li class="user-item">
          <strong>{{ user.name }}</strong> 
          <span class="email">({{ user.email }})</span>
        </li>
      } @empty {
        <!-- Блок @empty показується, якщо масив порожній (нічого не знайдено) -->
        <li class="no-results">
          🔍 Користувачів не знайдено.
        </li>
      }
    </ul>
  }
</div>
```

Пояснення ключових рішень:
Керування життєвим циклом (уникнення витоків пам'яті): Замість старого підходу зі створенням Subject та імплементацією ngOnDestroy, тут використовується сучасний оператор takeUntilDestroyed(this.destroyRef). Він прив'язує життя підписки до життєвого циклу компонента і автоматично відписується, коли компонент знищується.
Оптимізація пошуку: У методі setupSearch() використано debounceTime(300). Це означає, що фільтрація спрацює лише тоді, коли користувач перестане вводити текст на 300 мілісекунд. Це значно зменшує кількість непотрібних обчислень.
Новий Control Flow (@if, @for): У шаблоні використано вбудований синтаксис Angular. Блок @for вимагає вказівки track (у нашому випадку user.id), що допомагає Angular ефективно оновлювати DOM. Блок @empty елегантно вирішує проблему відображення повідомлення, коли результати пошуку порожні.
Функція inject(): Замість ін'єкції залежностей через конструктор, використовується функція inject(). Це робить код чистішим і спрощує наслідування класів у майбутньому.

## Оцінка експерта за рубрикою

| Критерій | Бал |
|----------|-----|
| К1 — точність ідентифікації | 5 |
| К2 — фактична коректність | 4 |
| К3 — повнота відповіді | 4 |
| К4 — відтворюваність (виставляється на серію з 3 запусків) | 2 |
| К5 — педагогічна цінність | 4 |
| **Q — підсумкова оцінка** | **3,90** |

Q обчислюється за формулою: Q = 0,30·К1 + 0,25·К2 + 0,15·К3 + 0,20·К4 + 0,10·К5

## Коментар експерта

Змішаний підхід: без використання сигналів (звичайні поля класу), але із сучасним control flow і оптимізацією debounceTime у полі пошуку. Елементи різних поколінь Angular поєднано в одному компоненті. Функціональні вимоги задачі виконано коректно.
