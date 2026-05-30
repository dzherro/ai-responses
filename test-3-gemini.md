# Категорія 2, тест розробки фронтенд-компонента Angular: Gemini 3.1 Pro (запуск 3)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | Gemini |
| Модель | 3.1 Pro |
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

Ось повне рішення для Angular (версія 21) з використанням сучасних підходів: Standalone компонентів, Signals для керування станом, нового синтаксису шаблонів (Control Flow @if, @for) та оператора takeUntilDestroyed для коректного управління життєвим циклом та уникнення витоків пам'яті.


### 1. Модель даних (user.model.ts)

Спочатку визначимо інтерфейс для користувача.

```typescript
export interface User {
  id: number;
  name: string;
  email: string;
  username: string;
}
```


### 2. Сервіс (user.service.ts)

Створимо сервіс, який робить HTTP GET-запит до публічного тестового API JSONPlaceholder. Замість конструктора використовуємо сучасну функцію inject().

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';
@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Сучасний підхід до ін'єкції залежностей
  private http = inject(HttpClient);
```

  
```html
  // Тестовий публічний API
  private apiUrl = 'https://jsonplaceholder.typicode.com/users';
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
}
```


### 3. Компонент (user-list.component.ts)

Тут реалізовано підписку на Observable, обробку станів (завантаження, помилка), поле пошуку та відписку при знищенні компонента.

```typescript
import { Component, inject, OnInit, DestroyRef, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule, FormControl } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';
import { UserService } from './user.service';
import { User } from './user.model';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.css']
})
export class UserListComponent implements OnInit {
  private userService = inject(UserService);
  private destroyRef = inject(DestroyRef); // Використовується для безпечної відписки
  // Контрол для поля пошуку
  searchControl = new FormControl('', { nonNullable: true });
```

  
```html
  // Використовуємо Signals для зберігання стану компонента (рекомендовано в нових версіях Angular)
  allUsers = signal<User[]>([]);
  filteredUsers = signal<User[]>([]);
  isLoading = signal<boolean>(true);
  errorMessage = signal<string | null>(null);
  ngOnInit() {
    this.fetchUsers();
    this.setupSearch();
  }
  private fetchUsers() {
    this.isLoading.set(true);
    this.errorMessage.set(null);
    this.userService.getUsers()
      // takeUntilDestroyed автоматично відпишеться від Observable при знищенні компонента,
      // що гарантує відсутність витоків пам'яті.
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: (users) => {
          this.allUsers.set(users);
          this.filteredUsers.set(users);
          this.isLoading.set(false);
        },
        error: (err) => {
          console.error('Помилка HTTP:', err);
          this.errorMessage.set('Не вдалося завантажити список користувачів. Спробуйте пізніше.');
          this.isLoading.set(false);
        }
      });
  }
  private setupSearch() {
    this.searchControl.valueChanges
      .pipe(
        debounceTime(300), // Чекаємо 300мс після останнього натискання клавіші
        distinctUntilChanged(), // Пропускаємо лише якщо значення дійсно змінилося
        takeUntilDestroyed(this.destroyRef) // Також запобігаємо витоку пам'яті для цього потоку
      )
      .subscribe(term => {
        const lowerTerm = term.toLowerCase();
        // Фільтруємо збережений список усіх користувачів
        const filtered = this.allUsers().filter(user => 
          user.name.toLowerCase().includes(lowerTerm)
        );
        this.filteredUsers.set(filtered);
      });
  }
}
```


### 4. Шаблон (user-list.component.html)

Використовуємо новий синтаксис Control Flow (@if, @for, @empty), який працює набагато швидше і є стандартом для сучасного Angular.

```html
<div class="user-container">
  <h2>Список користувачів</h2>
```

  
```html
  <div class="search-box">
    <!-- Зв'язування інпута з FormControl -->
    <input 
      type="text" 
      [formControl]="searchControl" 
      placeholder="Пошук за іменем..." 
      class="search-input"
    />
  </div>
  <!-- Стан завантаження -->
  @if (isLoading()) {
    <div class="loading-state">
      <p>Завантаження даних...</p>
    </div>
  }
  <!-- Стан помилки -->
  @if (errorMessage()) {
    <div class="error-state">
      <p>{{ errorMessage() }}</p>
      <button (click)="ngOnInit()">Спробувати знову</button>
    </div>
  }
  <!-- Відображення списку -->
  @if (!isLoading() && !errorMessage()) {
    <ul class="user-list">
      @for (user of filteredUsers(); track user.id) {
        <li class="user-item">
          <div class="user-info">
            <strong>{{ user.name }}</strong>
            <span class="user-email">{{ user.email }}</span>
          </div>
        </li>
      } @empty {
        <li class="empty-state">
          Користувачів за запитом "{{ searchControl.value }}" не знайдено.
        </li>
      }
    </ul>
  }
</div>
```

Пояснення архітектурних рішень:
Signals (signal): Використані для зберігання стану компонента (allUsers, filteredUsers, isLoading, errorMessage). Це найсучасніший спосіб реактивного оновлення UI в Angular, який працює без необхідності підключати Zone.js.
takeUntilDestroyed(this.destroyRef): Найкращий спосіб управління життєвим циклом в Angular для уникнення витоків пам'яті при ручній підписці (замінив старий спосіб з ngOnDestroy та Subject). Він автоматично припиняє прослуховування Observable, коли компонент знищується.
ReactiveFormsModule та debounceTime(300): Використані для поля пошуку, щоб не запускати логіку фільтрації на кожне натискання клавіші. Це робить пошук більш продуктивним (затримка 300 мс після останнього вводу).
Control Flow (@if, @for): Вбудований синтаксис шаблонів (введений з 17 версії), який ефективніше працює з пам'яттю, ніж застарілі директиви *ngIf та *ngFor. Він також має зручний блок @empty, який автоматично показується, якщо масив порожній.
Обробка помилок і завантаження: Компонент чітко розрізняє 3 стани (завантаження, помилка, успішно отримані дані) для найкращого користувацького досвіду.

## Оцінка експерта за рубрикою

| Критерій | Бал |
|----------|-----|
| К1 — точність ідентифікації | 5 |
| К2 — фактична коректність | 4 |
| К3 — повнота відповіді | 5 |
| К4 — відтворюваність (виставляється на серію з 3 запусків) | 2 |
| К5 — педагогічна цінність | 4 |
| **Q — підсумкова оцінка** | **4,05** |

Q обчислюється за формулою: Q = 0,30·К1 + 0,25·К2 + 0,15·К3 + 0,20·К4 + 0,10·К5

## Коментар експерта

Правильний виклик takeUntilDestroyed(this.destroyRef), із сигналами, новим control flow і debounceTime. Однак припустилася антипатерну виклику lifecycle-методу ngOnInit() напряму з шаблону через атрибут (click) для повторного запиту даних, що змішує концепції життєвого циклу компонента і користувацьких дій.
