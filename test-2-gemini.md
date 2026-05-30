# Категорія 2, тест розробки фронтенд-компонента Angular: Gemini 3.1 Pro (запуск 2)

## Метадані експерименту

| Параметр | Значення |
|----------|----------|
| Сервіс | Gemini |
| Модель | 3.1 Pro |
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

Ось повний приклад сучасного компонента на Angular (з використанням стандартів Angular 17-21, таких як Standalone Components, Signals, новий синтаксис шаблонів Control Flow @if/@for та takeUntilDestroyed для керування підписками).


### 1. Модель даних (user.model.ts)

Спочатку створимо інтерфейс для типізації наших даних:

```typescript
export interface User {
  id: number;
  name: string;
  email: string;
}
```


### 2. Сервіс (user.service.ts)

Сервіс використовує HttpClient для запиту до публічного тестового API (JSONPlaceholder).

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, delay } from 'rxjs';
import { User } from './user.model';
@Injectable({
  providedIn: 'root'
})
export class UserService {
  // Сучасний спосіб ін'єкції залежностей (замість конструктора)
  private http = inject(HttpClient);
  private apiUrl = 'https://jsonplaceholder.typicode.com/users';
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl).pipe(
      // Додаємо штучну затримку, щоб візуально побачити стан завантаження
      delay(500) 
    );
  }
}
```


### 3. Компонент (user-list.component.ts)

Тут ми реалізуємо логіку завантаження, обробки помилок, фільтрації та правильної відписки від Observable.

```typescript
import { Component, OnInit, inject, signal } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged, startWith } from 'rxjs/operators';
import { UserService } from './user.service';
import { User } from './user.model';
@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.css']
})
export class UserListComponent implements OnInit {
  private userService = inject(UserService);
  // Контрол для поля пошуку
  searchControl = new FormControl('');
  // Сучасне управління станом за допомогою Signals
  users = signal<User[]>([]);
  filteredUsers = signal<User[]>([]);
  isLoading = signal<boolean>(false);
  errorMessage = signal<string | null>(null);
  constructor() {
    // Підписка на зміни тексту в полі пошуку
    this.searchControl.valueChanges.pipe(
      // Автоматично скасовує підписку при знищенні компонента (запобігає витоку пам'яті)
      takeUntilDestroyed(), 
      debounceTime(300), // Чекаємо 300мс після останнього натискання
      distinctUntilChanged(), // Пропускаємо лише якщо значення змінилось
      startWith('') // Початкове значення для первинної фільтрації
    ).subscribe(searchTerm => {
      this.filterUsers(searchTerm || '');
    });
  }
  ngOnInit(): void {
    this.loadUsers();
  }
  loadUsers(): void {
    this.isLoading.set(true);
    this.errorMessage.set(null);
    this.userService.getUsers().pipe(
      // Відписка від HTTP-запиту, якщо компонент буде знищено до завершення запиту
      takeUntilDestroyed(this) // Передаємо this, оскільки викликаємо не в конструкторі
    ).subscribe({
      next: (data) => {
        this.users.set(data);
        this.filterUsers(this.searchControl.value || '');
        this.isLoading.set(false);
      },
      error: (error) => {
        console.error('Помилка завантаження користувачів:', error);
        this.errorMessage.set('Не вдалося завантажити список користувачів. Перевірте з\'єднання.');
        this.isLoading.set(false);
      }
    });
  }
  private filterUsers(term: string): void {
    const lowerTerm = term.toLowerCase().trim();
    const filtered = this.users().filter(user =>
      user.name.toLowerCase().includes(lowerTerm)
    );
    this.filteredUsers.set(filtered);
  }
}
```


### 4. Шаблон (user-list.component.html)

Використовуємо новий синтаксис Control Flow (@if, @for), який є значно швидшим і зручнішим, ніж старі *ngIf та *ngFor.

```html
<div class="user-list-container">
  <h2>Список користувачів</h2>
  <div class="search-box">
    <input
      type="text"
      [formControl]="searchControl"
      placeholder="Пошук за іменем..."
      class="search-input"
    />
  </div>
  <!-- Стан завантаження -->
  @if (isLoading()) {
    <div class="loading">
      <p>Завантаження даних...</p>
    </div>
  }
  <!-- Стан помилки -->
  @if (errorMessage()) {
    <div class="error-container">
      <p class="error-text">{{ errorMessage() }}</p>
      <button (click)="loadUsers()">Спробувати знову</button>
    </div>
  }
  <!-- Основний контент (якщо немає завантаження та помилок) -->
  @if (!isLoading() && !errorMessage()) {
    <ul class="user-list">
      @for (user of filteredUsers(); track user.id) {
        <li class="user-item">
          <strong>{{ user.name }}</strong>
          <span class="email">{{ user.email }}</span>
        </li>
      } @empty {
        <li class="no-results">
          Користувачів із таким іменем не знайдено.
        </li>
      }
    </ul>
  }
</div>
```

Пояснення рішень та дотримання вимог:
Сервіс із Observable: Створено UserService з методом getUsers(), який повертає Observable<User[]>, використовуючи вбудований HttpClient.
Підписка в компоненті: Метод subscribe викликається в ngOnInit -> loadUsers() для отримання списку. Дані зберігаються в Angular Signals (signal<User[]>), що забезпечує оптимальний цикл рендерингу (Change Detection) у нових версіях Angular.
Поле пошуку: Використано реактивні форми (FormControl). Додано debounceTime(300), щоб оптимізувати продуктивність (пошук спрацьовує не на кожне натискання клавіші, а через 300мс після зупинки вводу).
Уникнення витоків пам'яті: Для всіх підписок застосовано оператор takeUntilDestroyed(). Це найсучасніший і найчистіший спосіб в Angular (починаючи з 16 версії) автоматично скасувати підписку (unsubscribe), коли компонент видаляється з DOM. Тобі більше не потрібно реалізовувати інтерфейс OnDestroy та створювати додаткові Subject'и.
Стани завантаження та помилки: Створено окремі сигнали isLoading та errorMessage. У шаблоні через синтаксис @if показується лоадер під час очікування, повідомлення про помилку у разі збою запиту, або ж кнопка "Спробувати знову". Новий синтаксис @empty автоматично обробляє випадок, коли масив порожній (наприклад, пошук не дав результатів).


## Оцінка експерта за рубрикою

| Критерій | Бал |
|----------|-----|
| К1 — точність ідентифікації | 5 |
| К2 — фактична коректність | 2 |
| К3 — повнота відповіді | 4 |
| К4 — відтворюваність (виставляється на серію з 3 запусків) | 2 |
| К5 — педагогічна цінність | 3 |
| **Q — підсумкова оцінка** | **3,30** |

Q обчислюється за формулою: Q = 0,30·К1 + 0,25·К2 + 0,15·К3 + 0,20·К4 + 0,10·К5

## Коментар експерта

КРИТИЧНА ПОМИЛКА КОДУ: виклик takeUntilDestroyed(this) у методі компонента, де як аргумент передано посилання на сам компонент замість DestroyRef. У строгому режимі TypeScript такий код не компілюється; у нестрогому — компілюється, але призводить до неочікуваної поведінки при знищенні компонента. Окрім помилки, модель додала пояснювальний коментар, у якому впевнено обґрунтовує цей невірний виклик — класичний приклад галюцинації моделі з правдоподібним, але технічно невірним обґрунтуванням.
